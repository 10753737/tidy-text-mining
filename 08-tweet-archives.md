# Case study: comparing Twitter archives {#twitter}



One type of text that has certainly received its share of attention in recent years is text shared online via Twitter. In fact, several of the sentiment lexicons used in this book (and commonly used in general) were designed for use with and validated on tweets. Both of the authors of this book are on Twitter and are fairly regular users of it so in this case study, let's compare the entire Twitter archives of [Julia](https://twitter.com/juliasilge) and [David](https://twitter.com/drob).

## Getting the data and distribution of tweets

An individual can download their own Twitter archive by following [directions available here](https://support.twitter.com/articles/20170160). We each downloaded ours and will now open them up. Let's use lubridate to convert the string timestamps to date-time objects and just take a look at our tweeting patterns overall.


```r
library(lubridate)
library(ggplot2)
library(dplyr)

tweets_julia <- read.csv("data/tweets_julia.csv", stringsAsFactors = FALSE)
tweets_dave <- read.csv("data/tweets_dave.csv", stringsAsFactors = FALSE)
tweets_julia$timestamp <- with_tz(ymd_hms(tweets_julia$timestamp), 
                                  "America/Denver")
tweets_dave$timestamp <- with_tz(ymd_hms(tweets_dave$timestamp), 
                                 "America/New_York")
tweets <- bind_rows(tweets_julia %>% mutate(person = "Julia"),
                    tweets_dave %>% mutate(person = "David"))
ggplot(tweets, aes(x = timestamp, fill = person)) +
  geom_histogram(alpha = 0.5, position = "identity")
```

<img src="08-tweet-archives_files/figure-html/setup-1.png" width="768" />

David and Julia tweet at about the same rate currently and joined Twitter about a year apart from each other, but there about 5 years where David was not active on Twitter and Julia was. In total, Julia has about 4 times as many tweets as David.

## Word frequencies

Let's use `unnest_tokens` to make a tidy dataframe of all the words in our tweets, and remove the common English stop words. There are certain conventions in how people use text on Twitter, so we will do a bit more work with our text here than, for example, we did with the narrative text from Project Gutenberg. The first `mutate` line below removes links and cleans out some characters that we don't want. In the call to `unnest_tokens`, we unnest using a regex pattern, instead of just looking for single unigrams (words). This regex pattern is very useful for dealing with Twitter text; it retains hashtags and mentions of usernames with the `\@` symbol. Because we have kept these types of symbols in the text, we can't use a simple `anti_join` to remove stop words. Instead, we can take the approach shown in the `filter` line that uses `str_detect` from the stringr library.


```r
library(tidytext)
library(stringr)

reg <- "([^A-Za-z\\d#@']|'(?![A-Za-z\\d#@]))"
tidy_tweets <- tweets %>% 
  mutate(text = str_replace_all(text, "https://t.co/[A-Za-z\\d]+|http://[A-Za-z\\d]+|&amp;|&lt;|&gt;|RT", "")) %>%
  unnest_tokens(word, text, token = "regex", pattern = reg) %>%
  filter(!word %in% stop_words$word,
         str_detect(word, "[a-z]"))
```

Now we can calculate word frequencies for each person


```r
frequency <- tidy_tweets %>% group_by(person) %>% 
  count(word, sort = TRUE) %>% 
  left_join(tidy_tweets %>% group_by(person) %>% summarise(total = n())) %>%
  mutate(freq = n/total)
frequency
```

```
## Source: local data frame [23,092 x 5]
## Groups: person [2]
## 
##    person           word     n total        freq
##     <chr>          <chr> <int> <int>       <dbl>
## 1   Julia           time   567 76811 0.007381755
## 2   Julia    @selkie1970   565 76811 0.007355717
## 3   Julia       @skedman   518 76811 0.006743826
## 4   Julia            day   470 76811 0.006118915
## 5   Julia           baby   410 76811 0.005337777
## 6   David        #rstats   359 22426 0.016008205
## 7   Julia     @doctormac   342 76811 0.004452487
## 8   David @hadleywickham   306 22426 0.013644876
## 9   Julia           love   303 76811 0.003944747
## 10  Julia   @haleynburke   291 76811 0.003788520
## # ... with 23,082 more rows
```

This is a lovely, tidy data frame but we would actually like to plot those frequencies on the x- and y-axes of a plot, so we will need to use an `inner_join` and make a different dataframe.


```r
frequency <- inner_join(frequency %>% filter(person == "Julia") %>% rename(Julia = freq),
                        frequency %>% filter(person == "David") %>% rename(David = freq),
                        by = "word") %>% 
  ungroup() %>% select(word, Julia, David)
frequency
```

```
## # A tibble: 3,555 x 3
##       word       Julia        David
##      <chr>       <dbl>        <dbl>
## 1     time 0.007381755 0.0042807456
## 2      day 0.006118915 0.0014715063
## 3     baby 0.005337777 0.0001783644
## 4     love 0.003944747 0.0020065995
## 5    house 0.003775501 0.0001337733
## 6  morning 0.003645311 0.0004013199
## 7   people 0.003358894 0.0032997414
## 8     feel 0.003111534 0.0012931419
## 9   pretty 0.002942287 0.0010701864
## 10  school 0.002864173 0.0002229555
## # ... with 3,545 more rows
```

```r
library(scales)

ggplot(frequency, aes(Julia, David)) +
  geom_jitter(alpha = 0.1, size = 2.5, width = 0.4, height = 0.4) +
  geom_text(aes(label = word), check_overlap = TRUE, vjust = 1.5) +
  scale_color_gradient(limits = c(0, 0.001), low = "gray30", high = "gray75") +
  scale_x_log10(labels = percent_format()) +
  scale_y_log10(labels = percent_format()) +
  theme(legend.position="none") +
  geom_abline(color = "red")
```

<img src="08-tweet-archives_files/figure-html/spread-1.png" width="672" />


This may not even need to be pointed out, but David and Julia have used their Twitter accounts rather differently over the course of the past several years. David has used his Twitter account almost exclusively for professional purposes since he became more active, while Julia used it for entirely personal purposes until late 2015. We see these differences immediately in this plot exploring word frequencies, and they will continue to be obvious. Words near the red line in this plot are used with about equal frequencies by David and Julia, while words far away from the line are used much more by one person compared to the other. Because of the inner join we did above, words, hashtags, and usernames that appear in this plot are ones that we have both used at least once.


TODO: lots
