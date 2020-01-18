# Data {#appendixdata .unnumbered}

TODO fill in

## hcandersenr {-}

The **hcandersenr**[@R-hcandersenr] package includes the text of the 157 known fairy tales by the Danish author H.C. Andersen. 
The text comes with 5 different languages with 

- 156 in English,
- 154 in Spanish,
- 150 in German,
- 138 in Danish and
- 58 in French

The package comes with a dataset for each language with the naming convention `hcandersen_**`,
where `**` is a country code.
Each dataset comes as a data.frame with two columns; `text` and `book` where the `book` variable has the text divided into strings of up to 80 characters.

The package also comes with a dataset called `EK` which includes information about the publication date, language of origin and names in the different languages.

## scotus {-}

The **scotus**[@R-scotus] package contains a sample of the Supreme Court of the United States' opinions.
The `scotus_sample` data.frame includes 1 opinion per row along with the year, case name, docket number, and a unique ID number.

The text has had minimal preprocessing done on them and will include the header information in the text field.
Example of the beginning of an opinion is shown below


```
## No. 97-1992
## VAUGHN L. MURPHY, Petitioner v. UNITED PARCEL SERVICE, INC.
## ON WRIT OF CERTIORARI TO THE UNITED STATES COURT OF APPEALS FOR THE TENTH
## CIRCUIT
## [June 22, 1999]
## Justice O'Connor delivered the opinion of the Court.
## Respondent United Parcel Service, Inc. (UPS), dismissed petitioner Vaughn
## L. Murphy from his job as a UPS mechanic because of his high blood pressure.
## Petitioner filed suit under Title I of the Americans with Disabilities Act of
## 1990 (ADA or Act), 104 Stat. 328, 42 U.S.C. § 12101 et seq., in Federal District
## Court. The District Court granted summary judgment to respondent, and the Court
## of Appeals for the Tenth Circuit affirmed. We must decide whether the Court
## of Appeals correctly considered petitioner in his medicated state when it held
## that petitioner's impairment does not "substantially limi[t]" one or more of
## his major life activities and whether it correctly determined that petitioner
## is not "regarded as disabled." See §12102(2). In light of our decision in Sutton
## v. United Air Lines, Inc., ante, p. ____, we conclude that the Court of Appeals'
## resolution of both issues was correct.
```

## GitHub issue {-}

This [dataset](https://github.com/explosion/projects/tree/master/textcat-docs-issues) includes 1161 Github issue title and an indicator of whether the issue was about documentation or not.
The dataset is split into a training data set and evaluation data set.

TODO find out how to store this data


```r
library(jsonlite)
library(readr)
library(tidyverse)

json_to_df <- function(x) {
  json <- parse_json(x)
  
  tibble(text = json$text,
         label = json$label,
         answer = json$answer)
}

github_issues_training <- read_lines("https://raw.githubusercontent.com/explosion/projects/master/textcat-docs-issues/docs_issues_training.jsonl") %>%
  map_dfr(json_to_df)

github_issues_eval <- read_lines("https://raw.githubusercontent.com/explosion/projects/master/textcat-docs-issues/docs_issues_eval.jsonl") %>%
  map_dfr(json_to_df)
```
