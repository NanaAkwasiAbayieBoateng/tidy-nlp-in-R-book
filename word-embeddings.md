# Word Embeddings {#embeddings}



> You shall know a word by the company it keeps.
> <footer>--- [John Rupert Firth](https://en.wikiquote.org/wiki/John_Rupert_Firth)</footer>

So far in our discussion of natural language features, we have discussed preprocessing steps such as tokenization, removing stop words, and stemming in detail. We implement these types of preprocessing steps to be able to represent our text data in some data structure that is a good fit for modeling. An example of such a data structure is a sparse matrix. Perhaps, if we wanted to analyse or build a model for consumer complaints to the [United States Consumer Financial Protection Bureau (CFPB)](https://www.consumerfinance.gov/data-research/consumer-complaints/), we would start with straightforward word counts.


```r
library(tidyverse)
library(tidytext)
library(SnowballC)

complaints <- read_csv("data/complaints.csv.gz")

complaints %>%
  unnest_tokens(word, consumer_complaint_narrative) %>%
  anti_join(get_stopwords()) %>%
  mutate(stem = wordStem(word)) %>%
  count(complaint_id, stem) %>%
  cast_dfm(complaint_id, stem, n)
```

```
## Document-feature matrix of: 117,214 documents, 46,099 features (99.9% sparse).
```

The dataset of consumer complaints used in this book has been filtered to those submitted to the CFPB since 1 January 2019 that include a consumer complaint narrative (i.e., some submitted text).

Another way to represent our text data is to use [tf-idf](https://www.tidytextmining.com/tfidf.html) instead of word counts. This weighting for text features can often work better in predictive modeling.


```r
complaints %>%
  unnest_tokens(word, consumer_complaint_narrative) %>%
  anti_join(get_stopwords()) %>%
  mutate(stem = wordStem(word)) %>%
  count(complaint_id, stem) %>%
  bind_tf_idf(stem, complaint_id, n) %>%
  cast_dfm(complaint_id, stem, tf_idf)
```

```
## Document-feature matrix of: 117,214 documents, 46,099 features (99.9% sparse).
```

Notice that in either case, our final data structure is incredibly sparse and of high dimensionality with a huge number of features. Some modeling algorithms and the libraries which implement them can take advantage of the memory characteristics of sparse matrices for better performance; an example of this is regularized regression implemented in **glmnet**. Some modeling algorithms, including tree-based algorithms, do not perform better with sparse input, and then some libraries are not built to take advantage of sparse data structures, even if it would improve performance for those algorithms.

#### SPARSE VS. NON SPARSE MATRIX DIAGRAM GOES HERE

Linguists have long worked on vector models for language that can reduce the number of dimensions representing text data based on how people use language; the quote that opened this chapter dates to 1957. These kinds of dense word vectors are often called **word embeddings**.

## Understand word embeddings by finding them yourself

Word embeddings are a way to represent text data as numbers based on a huge corpus of text, capturing semantic meaning from words' context. 

<div class="rmdnote">
<p>Modern word embeddings are based on a statistical approach to modeling language, rather than a linguistics or rules-based approach.</p>
</div>

We can determine these vectors for a corpus of text using word counts and matrix factorization, as outlined by @Moody2017. This approach is valuable because it allows practitioners to find word vectors for their own collections of text (with no need to rely on pre-trained vectors) using familiar techniques that are not difficult to understand. Let's walk through how to do this using tidy data principles and sparse matrices, on the dataset of CFPB complaints. First, let's filter out words that are used only rarely in this dataset and create a nested dataframe, with one row per complaint.


```r
nested_words <- complaints %>%
  select(complaint_id, consumer_complaint_narrative) %>%
  unnest_tokens(word, consumer_complaint_narrative) %>%
  add_count(word) %>%
  filter(n >= 50) %>%
  select(-n) %>%
  nest(words = c(word))

nested_words
```

```
## # A tibble: 117,170 x 2
##    complaint_id          words
##           <dbl> <list<df[,1]>>
##  1      3384392       [18 × 1]
##  2      3417821       [71 × 1]
##  3      3433198       [77 × 1]
##  4      3366475       [69 × 1]
##  5      3385399      [213 × 1]
##  6      3444592       [19 × 1]
##  7      3379924      [121 × 1]
##  8      3446975       [22 × 1]
##  9      3214857       [64 × 1]
## 10      3417374       [44 × 1]
## # … with 117,160 more rows
```


Next, let’s create a `slide_windows()` function, using the `slide()` function from the **slide** package [@Vaughan2020] which implements fast sliding window computations written in C. Our new function identifies skipgram windows in order to calculate the skipgram probabilities, how often we find each word near each other word. We do this by defining a fixed-size moving window that centers around each word. Do we see `word1` and `word2` together within this window? We can calculate probabilities based on when we do or do not.

One of the arguments to this function is the `window_size`, which determines the size of the sliding window that moves through the text, counting up words that we find within the window. The best choice for this window size depends on your analytical question because it determines what kind of semantic meaning the embeddings capture. A smaller window size, like three or four, focuses on how the word is used and learns what other words are functionally similar. A larger window size, like ten, captures more information about the domain or topic of each word, not constrained by how functionally similar the words are [@Levy2014]. A smaller window size is also faster to compute.


```r
slide_windows <- function(tbl, window_size) {
  skipgrams <- slide::slide(
    tbl,
    ~.x,
    .after = window_size - 1,
    .step = 1,
    .complete = TRUE
  )

  safe_mutate <- safely(mutate)

  out <- map2(
    skipgrams,
    1:length(skipgrams),
    ~ safe_mutate(.x, window_id = .y)
  )

  out %>%
    transpose() %>%
    pluck("result") %>%
    compact() %>%
    bind_rows()
}
```

Now that we can find all the skipgram windows, we can calculate how often words occur on their own, and how often words occur together with other words. We do this using the point-wise mutual information (PMI), a measure of association that measures exactly what we described in the previous sentence; it's the logarithm of the probability of finding two words together, normalized for the probability of finding each of the words alone. We use PMI to measure which words occur together more often than expected based on how often they occurred on their own. 

For this example, let's use a window size of **four**.

<div class="rmdnote">
<p>This next step is the computationally expensive part of finding word embeddings with this method, and can take a while to run. Fortunately, we can use the <strong>furrr</strong> package <span class="citation">[@Vaughan2018]</span> to take advantage of parallel processing because identifying skipgram windows in one document is independent from all the other documents.</p>
</div>



```r
library(widyr)
library(furrr)

plan(multiprocess) ## for parallel processing

tidy_pmi <- nested_words %>%
  mutate(words = future_map(words, slide_windows, 4,
    .progress = TRUE
  )) %>%
  unnest(words) %>%
  unite(window_id, complaint_id, window_id) %>%
  pairwise_pmi(word, window_id)

tidy_pmi
```


```
## # A tibble: 4,818,402 x 3
##    item1   item2           pmi
##    <chr>   <chr>         <dbl>
##  1 systems transworld  7.09   
##  2 inc     transworld  5.96   
##  3 is      transworld -0.135  
##  4 trying  transworld -0.107  
##  5 to      transworld -0.00206
##  6 collect transworld  1.07   
##  7 a       transworld -0.516  
##  8 debt    transworld  0.919  
##  9 that    transworld -0.542  
## 10 not     transworld -1.17   
## # … with 4,818,392 more rows
```

When PMI is high, the two words are associated with each other, likely to occur together. When PMI is low, the two words are not associated with each other, unlikely to occur together.


<div class="rmdtip">
<p>The step above used <code>unite()</code>, a function from <strong>tidyr</strong> that pastes multiple columns into one, to make a new column for <code>window_id</code> from the old <code>window_id</code> plus the <code>complaint_id</code>. This new column tells us which combination of window and complaint each word belongs to.</p>
</div>

We can next determine the word vectors from the PMI values using singular value decomposition. Let's use the `widely_svd()` function in **widyr**, creating 100-dimensional word embeddings. This matrix factorization is much faster than the previous step of identifying the skipgram windows and calculating PMI.


```r
tidy_word_vectors <- tidy_pmi %>%
  widely_svd(
    item1, item2, pmi,
    nv = 100, maxit = 1000
  )

tidy_word_vectors
```

```
## # A tibble: 747,500 x 3
##    item1   dimension    value
##    <chr>       <int>    <dbl>
##  1 systems         1 -0.0165 
##  2 inc             1 -0.0191 
##  3 is              1 -0.0202 
##  4 trying          1 -0.0423 
##  5 to              1 -0.00904
##  6 collect         1 -0.0370 
##  7 a               1 -0.0126 
##  8 debt            1 -0.0430 
##  9 that            1 -0.0136 
## 10 not             1 -0.0213 
## # … with 747,490 more rows
```

We have now successfully found word embeddings, with clear and understandable code. This is a real benefit of this approach; this approach is based on counting, dividing, and matrix decomposition and is thus easier to understand and implement than options based on deep learning. Training word vectors or embeddings, even with this straightforward method, still requires a large dataset (ideally, hundreds of thousands of documents or more) and a not insignificant investment of time and computational power. 

## Exploring CFPB word embeddings

Now that we have determined word embeddings for the dataset of CFPB complaints, let's explore them and talk about they are used in modeling. We have projected the sparse, high-dimensional set of word features into a more dense, 100-dimensional set of features. 

<div class="rmdnote">
<p>Each word can be represented as a numeric vector in this new feature space.</p>
</div>

Which words are close to each other in this new feature space of word embeddings? Let's create a simple function that will find the nearest synonyms using our newly created word embeddings.


```r
nearest_synonyms <- function(df, token) {
  df %>%
    widely(~ . %*% (.[token, ]), sort = TRUE)(item1, dimension, value) %>%
    select(-item2)
}
```

This function takes the tidy word embeddings as input, along with a word (or `token`) as a string. It uses matrix multiplication to find which words are closer or farther to the input word, and returns a dataframe sorted by similarity.

What words are closest to `"error"` in the dataset of CFPB complaints, as determined by our word embeddings?


```r
tidy_word_vectors %>%
  nearest_synonyms("error")
```

```
## # A tibble: 7,475 x 2
##    item1      value
##    <chr>      <dbl>
##  1 error     0.0373
##  2 issue     0.0237
##  3 problem   0.0235
##  4 issues    0.0194
##  5 errors    0.0187
##  6 mistake   0.0185
##  7 system    0.0170
##  8 problems  0.0151
##  9 late      0.0141
## 10 situation 0.0138
## # … with 7,465 more rows
```

Errors, problems, issues, mistakes -- sounds bad!

What is closest to the word `"month"`?


```r
tidy_word_vectors %>%
  nearest_synonyms("month")
```

```
## # A tibble: 7,475 x 2
##    item1     value
##    <chr>     <dbl>
##  1 month    0.0597
##  2 payment  0.0407
##  3 months   0.0355
##  4 payments 0.0325
##  5 year     0.0314
##  6 days     0.0274
##  7 balance  0.0267
##  8 xx       0.0265
##  9 years    0.0262
## 10 monthly  0.0260
## # … with 7,465 more rows
```

We see words about payments, along with other time periods such as days and years. Notice that we did not stem this text data (see Chapter \@ref(stemming)) but the word embeddings learned that singular and plural forms of words belong together.

What words are closest in this embedding space to `"fee"`?


```r
tidy_word_vectors %>%
  nearest_synonyms("fee")
```

```
## # A tibble: 7,475 x 2
##    item1      value
##    <chr>      <dbl>
##  1 fee       0.0762
##  2 fees      0.0605
##  3 charge    0.0421
##  4 interest  0.0410
##  5 charged   0.0387
##  6 late      0.0377
##  7 charges   0.0366
##  8 overdraft 0.0327
##  9 charging  0.0245
## 10 month     0.0220
## # … with 7,465 more rows
```

We find words about interest, charges, and overdrafts.

Since we have found word embeddings via singular value decomposition, we can use these vectors to understand what principal components explain the most variation in the CFPB complaints.


```r
tidy_word_vectors %>%
  filter(dimension <= 24) %>%
  group_by(dimension) %>%
  top_n(12, abs(value)) %>%
  ungroup() %>%
  mutate(item1 = reorder_within(item1, value, dimension)) %>%
  ggplot(aes(item1, value, fill = as.factor(dimension))) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~dimension, scales = "free_y", ncol = 4) +
  scale_x_reordered() +
  coord_flip() +
  labs(
    x = NULL, y = "Value",
    title = "First 24 principal components for text of CFPB complaints",
    subtitle = "Top words contributing to the components that explain the most variation"
  )
```

<div class="figure">
<img src="word-embeddings_files/figure-epub3/embeddingpca-1.png" alt="Word embeddings for Consumer Finance Protection Bureau complaints"  />
<p class="caption">(\#fig:embeddingpca)Word embeddings for Consumer Finance Protection Bureau complaints</p>
</div>

It becomes very clear in Figure \@ref(fig:embeddingpca) that stop words have not been removed, but notice that we can learn meaningful relationships in how very common words are used. Component 12 shows us how common prepositions are often used with words like `"regarding"`, `"contacted"`, and `"called"`, while component 9 highlights the use of *different* common words when submitting a complaint about unethical, predatory, and/or deceptive practices. Stop words do carry information, and methods like determining word embeddings can make that information usable.

We created word embeddings and can explore them to understand our text dataset, but how do we use this vector representation in modeling?

## Use pre-trained word embeddings {#glove}



```r
library(textdata)

## get GloVe embeddings
```


https://github.com/mkearney/wactor

Pre-trained word embeddings are trained based on very large, general purpose English language datasets. Commonly used [word2vec embeddings](https://code.google.com/archive/p/word2vec/) are based on the Google News dataset, and commonly used [GloVe embeddings](https://nlp.stanford.edu/projects/glove/) and [FastText embeddings](https://fasttext.cc/docs/en/english-vectors.html) are learned from the text of Wikipedia.

## Fairness and word embeddings {#fairnessembeddings}

Perhaps more than any of the other preprocessing steps this book has covered so far, using word embeddings opens an analysis or model up to the possibility of being influenced by systemic unfairness and bias. Embeddings are trained or learned from a large corpus of text data, and whatever human prejudice or bias exists in the corpus becomes imprinted into the vector data of the embeddings. This is true of all machine learning to some extent (models learn, reproduce, and often amplify whatever biases exist in training data) but this is literally, concretely true of word embeddings. @Caliskan2016 show how the GloVe word embeddings (the same embeddings we used in Section \@ref(glove)) replicate human-like semantic biases.

- African American first names are associated with more unpleasant feelings than European American first names.
- Women's first names are more associated with family and men's first names are more associated with career.
- Terms associated with women are more associated with the arts and terms associated with men are more associated with science.

Results like these have been confirmed over and over again, such as when @Bolukbasi2016 demonstrated gender stereotypes in how word embeddings encode professions or when Google Translate [exhibited apparently sexist behavior when translating text from languages with no gendered pronouns](https://twitter.com/seyyedreza/status/935291317252493312). ^[Google has since [worked to correct this problem.](https://www.blog.google/products/translate/reducing-gender-bias-google-translate/)] @Garg2018 even used how bias and stereotypes can be found in word embeddings to quantify how social attitudes towards women and minorities have changed over time. 

EMPHASIZE AGAIN THE TRAINING DATASETS -- REFERENCE FOR GENDER BALANCE ON WIKIPEDIA

<div class="rmdtip">
<p>It's safe to assume that any large corpus of language will contain latent structure reflecting the biases of the people who generated that language.</p>
</div>

When embeddings with these kinds of stereotypes are used as a preprocessing step in training a predictive model, the final model can exhibit racist, sexist, or otherwise biased characteristics. @Speer2017 demonstrated how using pre-trained word embeddings to train a straightforward sentiment analysis model can result in text such as 

> "Let's go get Italian food"

being scored much more positively than text such as

> "Let's go get Mexican food"

because of characteristics of the text the word embeddings were trained on.

## Using word embeddings in the real world

Given these profound and fundamental challenges with word embeddings, what options are out there? First, consider not using word embeddings when building a text model. Depending on the particular analytical question you are trying to answer, another numerical representation of text data (such as word frequencies or tf-idf of single words or n-grams) may be more appropriate. Consider this option even more seriously if the model you want to train is already entangled with issues of bias, such as the sentiment analysis example in Section \@ref(fairnessembeddings).

Consider whether finding your own word embeddings, instead of relying on pre-trained embeddings such as GloVe or word2vec, may help you. Building your own vectors is likely to be a good option when the text domain you are working in is **specific** rather than general purpose; some examples of such domains could include customer feedback for a clothing e-commerce site, comments posted on a coding Q&A site, or legal documents. 

Learning good quality word embeddings is only realistic when you have a large corpus of text data (say, hundreds of thousands or more documents) but if you have that much data, it is possible that embeddings learned from scratch based on your own data may not exhibit the same kind of semantic biases that exist in pre-trained word embeddings. Almost certainly there will be some kind of bias latent in any large text corpus, but when you use your own training data for learning word embeddings, you avoid the problem of *adding* historic, systemic prejudice from general purpose language datasets. You can use the same approaches discussed here to check any new embeddings for dangerous biases such as racism or sexism.

<div class="rmdnote">
<p>You can use the same approaches discussed in this chapter to check any new embeddings for dangerous biases such as racism or sexism.</p>
</div>

NLP researchers have also proposed methods for debiasing embeddings. @Bolukbasi2016 aim to remove stereotypes by postprocessing pre-trained word vectors, choosing specific sets of words that are reprojected in the vector space so that some specific bias, such as gender bias, is mitigated. This is the most established method for reducing bias in embeddings to date, although other methods have been proposed as well, such as augmenting data with counterfactuals [@Lu2018]. Recent work [@Ethayarajh2019] has explored whether the association tests used to measure bias are even useful, and under what conditions debiasing can be effective.

Other researchers, such as @Caliskan2016, suggest that corrections for fairness should happen at the point of **decision** or action rather than earlier in the process of modeling, such as preprocessing steps like building word embeddings. The concern is that methods for debiasing word embeddings may allow the stereotypes to seep back in, and more recent work shows that this is exactly what can happen. @Gonen2019 highlight how pervasive and consistent gender bias is across different word embedding models, *even after* applying current debiasing methods.

## Summary

TKTKTK It's important to keep in mind that even more advanced natural language algorithms, such as language models with transformers, also exhibit such systemic biases [@Sheng2019].