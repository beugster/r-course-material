NLP Processing in R
================
Wouter van Atteveldt & Kasper Welbers
2019-04

-   [Installing Spacyr](#installing-spacyr)
    -   [Installing Spacy using anaconda (esp. for Windows):](#installing-spacy-using-anaconda-esp.-for-windows)
    -   [Installing Spacy using pip (esp. for Mac/Linux):](#installing-spacy-using-pip-esp.-for-maclinux)
    -   [Initializing spacyr](#initializing-spacyr)
    -   [Troubleshooting](#troubleshooting)
-   [Using Spacy(r)](#using-spacyr)
    -   [Extracting information from spacy output](#extracting-information-from-spacy-output)
    -   [Reading spacy output into Quanteda](#reading-spacy-output-into-quanteda)
-   [Closing the door](#closing-the-door)

For text analysis it is often useful to POS tag and lemmatize your text, especially with non-English data. R does not really have built-in functions for that, but there are libraries that connect to external tools to help you do this. This handout reviews spacy, which has a great R library and support for a number of (western) languages.

Installing Spacyr
=================

Spacy is a python package with processing models for 6 different languages, which makes it attractive to use if you need e.g. French or German lemmatizing.

To install it, you need to install the `spacy` module in python and download the appropriate language model. This depends on your computer and can be tricky to get right. The sections below give some instructions on how to do this depending on your OS, see <https://spacy.io/usage/> for more information.

Installing Spacy using anaconda (esp. for Windows):
---------------------------------------------------

1.  Install [anaconda](https://www.anaconda.com/)
2.  In Anaconda navigator, create a virtual environment, e.g. named 'spacy', based on python 3
3.  Once that environment is setup up, click the 'Play' button to open a terminal, and run: (replacing `de_core_news_sm` with the model you need)

        conda install -c conda-forge spacy
        python -m spacy download de_core_news_sm

    tions on how to do this depending on your OS, see <https://spacy.io/usage/> for more information.

Installing Spacy using pip (esp. for Mac/Linux):
------------------------------------------------

1.  Open a terminal (ubuntu: control-alt-T, mac: finder -&gt; terminal)
2.  For mac, install xcode using `xcode-select --install`
3.  In the terminal, type the following commands to create a virtual environment, install spacy, and download the language model of your choice:

        python3 -m venv spacy-env
        spacy-env/bin/pip install spacy
        spacy-env/bin/python -m spacy download de_core_news_sm

Initializing spacyr
-------------------

The next step is to initialize spacyr by pointing it to your installed environment.

For anaconda, use the following (substituting the name you picked for the virtual environment and the language model of your choice):

``` r
library(spacyr)
spacy_initialize("de_core_news_sm", condaenv = "spacy")
```

For pip, use the following:

``` r
library(spacyr)
spacy_initialize("de_core_news_sm", python_executable = path.expand("~/spacy-env/bin/python"))
```

Troubleshooting
---------------

As said, getting spacy and spacyr installed can be tricky. If you get an error message on initializing spacyr, it might help to install spacyr and reticulate directly from source. Before installing, clean your environment (with the broomstick icon) and restart R using the Session menu.

``` r
if (!require(devtools)) install.packages(devtools)
install_github("rstudio/reticulate")
install_github("quanteda/spacyr")
```

If that doesn't help, you can try installnig spacy the other way (following the instructions above), and/or googling your error message.

Using Spacy(r)
==============

With spacy successfully installed and spacyr initialized, we can now use it to parse text:

``` r
tokens = spacy_parse("Ich bin ein Berliner")
head(tokens)
```

Extracting information from spacy output
----------------------------------------

Before we go on, let's get a more realistic example: a German wikinews article about Volkswagen.

``` r
url = "https://gist.githubusercontent.com/vanatteveldt/bf9527ac6510e9b3e5c6b198b917ddd1/raw/45e6f6bfa0abba219935543eb70cca9f675703c7/VW_erneut_unter_Verdacht.txt"
library(readtext)
d = readtext(url, encoding = "utf-8")
d$text
```

We can parse it as earlier:

``` r
tokens = spacy_parse(d$text, nounphrase = T)
head(tokens)
```

Of course, since the resulting tokens are simply a data frame, we can use our normal functions to e.g. list all verbs:

``` r
library(tidyverse)
tokens %>% filter(pos=="VERB") %>% group_by(lemma) %>% summarize(n=n()) %>% arrange(-n)
tokens %>% filter(lemma == "stehen")
```

And we can quite easily recreate sentences as well by summarizing with the str\_c function:

``` r
tokens %>% filter(sentence_id == 3) %>% arrange(token_id) %>% summarize(sentence=str_c(token, collapse=" "))
tokens %>% filter(sentence_id == 3) %>% arrange(token_id) %>% summarize(sentence=str_c(lemma, pos, sep = "/", collapse=" "))
```

We can extract all entities (which combines multi-word entities):

``` r
entity_extract(tokens)
entity_extract(tokens) %>% filter(entity_type=="ORG") %>% group_by(entity) %>% summarize(n=n())
```

Or all noun phrases (requires `nounphrase=T` in the parsing):

``` r
nounphrase_extract(tokens)
```

You can also 'consolidate' the entities or nounphrases, meaning that the tokens will actually be replaced by them:

``` r
tokens2 = entity_consolidate(tokens) 
head(tokens2)
tokens2 = nounphrase_consolidate(tokens) 
head(tokens2)
```

Reading spacy output into Quanteda
----------------------------------

It can be very useful to read spacy output back in quanteda. For example, we might wish to do a word cloud, dictionary analysis, or topic model of only the nouns in a text.

`spacyr` and `quanteda` are both developed by the group of Kenneth Benoit's group, so they are quite easy to integrate.

The `as.tokens` function transforms the tokens dataframe into a quanteda `tokens` object, which can then be used to create a dfm.

For example, this creates a word cloud of all lemmata:

``` r
library(quanteda)
tokens %>% as.tokens(use_lemma=T) %>% dfm %>% textplot_wordcloud(min_count = 1)
```

It is often useful to filter on POS tag before creating the dfm. Unfortunately, you can't use the normal filter operation as that drops the information from the tokens object on which quanteda depends. However, you can extract the tokens with the POS, and then use that to filter:

``` r
dfm_nouns = tokens %>% as.tokens(include_pos = "pos") %>%
  tokens_select(pattern = c("*/NOUN")) %>% dfm
dfm_nouns %>% textplot_wordcloud(min_count = 1)
```

If you wish to drop the `/NOUN` from the features afterwards, you can access the column names of the dfm directly:

``` r
library(magrittr)
colnames(dfm_nouns) %<>% str_remove("/noun")
dfm_nouns %>% textplot_wordcloud(min_count = 1)
```

(note: the fancy notation `[x %<>% ...]` is the same as `[x = x %>% ...]`, but you need to library magrittr explicitly)

This also works with the entity consolidation:

``` r
library(quanteda)
tokens %>% entity_consolidate %>% as.tokens(use_lemma=T, include_pos = "pos") %>% 
    tokens_select(pattern = c("*/NOUN", "*/ENTITY")) %>% dfm %>%
 textplot_wordcloud(min_count = 1)
```

Closing the door
================

Spacyr keeps a python process running, which can consume quite a lot of memory. When you are done with spacy (but want to continue with R), you can finalize spacy:

``` r
spacy_finalize()
```

After this, you will have to initialize spacy again before you can parse new text.
