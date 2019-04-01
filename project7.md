Sentiment Analysis
================
Erika Vargas
February 14, 2019

``` r
# required packages
library(twitteR)
library(sentiment)
```

    ## Loading required package: tm

    ## Loading required package: NLP

    ## Loading required package: Rstem

``` r
library(plyr)
```

    ## 
    ## Attaching package: 'plyr'

    ## The following object is masked from 'package:twitteR':
    ## 
    ##     id

``` r
library(ggplot2)
```

    ## 
    ## Attaching package: 'ggplot2'

    ## The following object is masked from 'package:NLP':
    ## 
    ##     annotate

``` r
library(wordcloud)
```

    ## Loading required package: RColorBrewer

``` r
library(RColorBrewer)

#load credentials
consumer_key <- "bC8PuvI0U2CbVkKw8hgqgWbaI"
consumer_secret<- "CBydw33rSKfnZO8w21VbNOXPYdjLWW3lZ6xG8rEC2YEyp0FDAA"
access_token <- "1095555167217364992-7Rz3ZjNA11f0HQP0qw7joHiRu0o8zR"
access_secret <- "IaOVqsFrdM7LbGThdsFtthcBKIBVQg00Qhvy6DlbRU4Jh"

#set up to authenticate
setup_twitter_oauth(consumer_key ,consumer_secret,access_token ,access_secret)
```

    ## [1] "Using direct authentication"

``` r
# harvest some tweets
some_tweets = searchTwitter("starbucks", n=1500, lang="en")
# get the text
some_txt = sapply(some_tweets, function(x) x$getText())
```

``` r
#step3
# remove retweet entities
some_txt = gsub("(RT|via)((?:\\b\\W*@\\w+)+)", "", some_txt)
# remove at people
some_txt = gsub("@\\w+", "", some_txt)
# remove punctuation
some_txt = gsub("[[:punct:]]", "", some_txt)
# remove numbers
some_txt = gsub("[[:digit:]]", "", some_txt)
# remove html links
some_txt = gsub("http\\w+", "", some_txt)
# remove unnecessary spaces
some_txt = gsub("[ \t]{2,}", "", some_txt)
some_txt = gsub("^\\s+|\\s+$", "", some_txt)

# define "tolower error handling" function
try.error = function(x)
{
  # create missing value
  y = NA
  # tryCatch error
  try_error = tryCatch(tolower(x), error=function(e) e)
  # if not an error
  if (!inherits(try_error, "error"))
    y = tolower(x)
  # result
  return(y)
}
# lower case using try.error with sapply
some_txt = sapply(some_txt, try.error)
# remove NAs in some_txt
some_txt = some_txt[!is.na(some_txt)]
names(some_txt) = NULL
```

``` r
#step4 
# classify emotion
class_emo = classify_emotion(some_txt, algorithm="bayes", prior=1.0)
# get emotion best fit
emotion = class_emo[,7]
# substitute NA's by "unknown"
emotion[is.na(emotion)] = "unknown"
# classify polarity
class_pol = classify_polarity(some_txt, algorithm="bayes")
# get polarity best fit
polarity = class_pol[,4]
```

``` r
#step5

# data frame with results
sent_df = data.frame(text=some_txt, emotion=emotion,
                     polarity=polarity, stringsAsFactors=FALSE)
# sort data frame
sent_df = within(sent_df,
                 emotion <- factor(emotion, levels=names(sort(table(emotion),
                                                              decreasing=TRUE))))
```

``` r
#step6
# plot distribution of emotions
ggplot(sent_df, aes(x=emotion)) +
  geom_bar(aes(y=..count.., fill=emotion)) +
  scale_fill_brewer(palette="Dark2") +
  labs(x="emotion categories", y="number of tweets") +
  ggtitle("Sentiment Analysis of Tweets about Starbucks\n(classification
       by emotion)")
```

![](project7_files/figure-markdown_github/unnamed-chunk-6-1.png)

*In this boxplot we can see that more than 1000 tweets don’t have a particular emotion attached to Starbucks. However, the emotion joy is related to almost 250 Starbucks tweets.* *More than 100 Starbucks tweets are related to Sadness*

``` r
# plot distribution of polarity
ggplot(sent_df, aes(x=polarity)) +
  geom_bar(aes(y=..count.., fill=polarity)) +
  scale_fill_brewer(palette="RdGy") +
  labs(x="polarity categories", y="number of tweets") +
  ggtitle("Sentiment Analysis of Tweets about Starbucks\n(classification
       by polarity)")
```

![](project7_files/figure-markdown_github/unnamed-chunk-7-1.png)

*In this boxplot we can se that more than 800 tweets about Starbucks are positive. Negative and neutral tweets about Starbucks are less than half of the positive ones.*

``` r
#step7
# separating text by emotion  and visualize the words with a comparison cloud
emos = levels(factor(sent_df$emotion))
nemo = length(emos)
emo.docs = rep("", nemo)
for (i in 1:nemo)
{
  tmp = some_txt[emotion == emos[i]]
  emo.docs[i] = paste(tmp, collapse=" ")
}

# remove stopwords
emo.docs = removeWords(emo.docs, stopwords("english"))

library(stringr)
usableT=str_replace_all(emo.docs,"[^[:graph:]]", " ")
# create corpus
corpus = Corpus(VectorSource(usableT))
tdm = TermDocumentMatrix(corpus)
tdm = as.matrix(tdm)
colnames(tdm) = emos

# comparison word cloud
comparison.cloud(tdm, colors = brewer.pal(nemo, "Dark2"),
                 scale = c(3,.5), random.order = FALSE,
                 title.size = 1.5)
```

![](project7_files/figure-markdown_github/unnamed-chunk-8-1.png)

*The word cloud helps us identify those words in tweets that are related to unknown emotions, we can see that just the name Starbucks and coffee are the most popular for unknown emotions. It makes sense because they are not emotions, but they are related and therefore they people may type “coffee” and “Starbucks” when tweeting about the company.*

*Words such as “love”, “good”, “like” are reflect a sentiment of joy, we see also the high frequency of “the”, which does not really give relevant information and we might want to remove it next time before making the word cloud.* *- For sad tweets referring to Starbucks we can tell that people usually tweet “Roses are red, Violets are blue”. “date”, “Frappuccino’s” are also words that reflect sadness.* *- Anger, disgust, fear, and surprise, don’t have significant words, but I can tell that people that angry when getting Starbucks will type words such as “bitter”, “taste”, “Seattle’s”.* *- People that maybe didn’t like the coffee they got will tweet words such as ”get”, “different”, ”tired”, ”frappe”, “vanilla”. It could be that the new drink Starbucks is offering is not liked by its customers.*
