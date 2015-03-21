# Capstone milestone report
Melissa Tan  
Sunday, March 15, 2015  



## Introduction

The primary aim of the capstone project is to predict the next word that will appear after a given phrase. The predictions will be based on an analysis of given texts, which were collected from publicly available sources by a web crawler.

## Download data

As instructed, the dataset must be downloaded from the link given in the Coursera website, [https://d396qusza40orc.cloudfront.net/dsscapstone/dataset/Coursera-SwiftKey.zip](https://d396qusza40orc.cloudfront.net/dsscapstone/dataset/Coursera-SwiftKey.zip). Unzip into parent directory.


```r
if (!file.exists("../final")) {
  fileUrl <- "https://d396qusza40orc.cloudfront.net/dsscapstone/dataset/Coursera-SwiftKey.zip"
  download.file(fileUrl, destfile = "../swiftkey.zip")
  unzip("../swiftkey.zip")
}
```

Unzipping gives us a directory called `final`. The datasets that we will use are inside the subdirectory `en_US`. Print them out using system call.


```r
orig.wd <- getwd() # save working dir, to return to later
setwd("../final/en_US")
system("ls")
```

We will be using these three datasets for text prediction:     
* en_US.blogs.txt - text from blog posts
* en_US.news.txt - text from 
* en_US.twitter.txt

## Basic summary of data

Here's a basic summary of each of the three datasets. For this section I'll mostly be using system tools (Unix commands), since that's quick and straightforward.

### Word and line counts: 

Word count: Use Unix command `wc`, with `-w` flag.    
Line count: Use `wc`, with `-l` flag.   
Length of longest line (in terms of word count): Use `wc`, with `-L` flag.  


```r
setwd("../final/en_US")
numwords <- system("wc -w *.txt", intern=TRUE)  # intern=TRUE to return output  
numlines <- system("wc -l *.txt", intern=TRUE)
longest <- system("wc -L *.txt", intern=TRUE)
setwd(orig.wd)  # return to original working dir, ie. the parent of /final

# number of words for each dataset
blog.numwords <- as.numeric(gsub('[^0-9]', '', numwords[1]))
news.numwords <- as.numeric(gsub('[^0-9]', '', numwords[2]))
twit.numwords <- as.numeric(gsub('[^0-9]', '', numwords[3]))
# number of lines for each dataset
blog.numlines <- as.numeric(gsub('[^0-9]', '', numlines[1]))
news.numlines <- as.numeric(gsub('[^0-9]', '', numlines[2]))
twit.numlines <- as.numeric(gsub('[^0-9]', '', numlines[3]))
# length of longest line for each dataset
blog.longest  <- as.numeric(gsub('[^0-9]', '', longest[1]))
news.longest  <- as.numeric(gsub('[^0-9]', '', longest[2]))
twit.longest  <- as.numeric(gsub('[^0-9]', '', longest[3]))

# create and display summary table
blog.stats <- c(blog.numwords, blog.numlines, blog.longest,
                round(blog.numwords/blog.numlines))
news.stats <- c(news.numwords, news.numlines, news.longest,
                round(news.numwords/news.numlines))
twit.stats <- c(twit.numwords, twit.numlines, twit.longest, 
                round(twit.numwords/twit.numlines))  
data.stats <- data.frame(rbind(blog.stats, news.stats, twit.stats))
names(data.stats) <- c("Total word count", 
                       "Total line count", 
                       "Words in longest line",
                       "Avg words per line")
kable(data.stats)  # display the above in table format
```

              Total word count   Total line count   Words in longest line   Avg words per line
-----------  -----------------  -----------------  ----------------------  -------------------
blog.stats            37334114             899288                   40833                   42
news.stats            34365936            1010242                   11384                   34
twit.stats            30359852            2360148                     173                   13

## Take random subsample

Since the datasets are too large for my laptop memory to handle, I write a function that will produce a random subsample of each of the datasets. The function essentially flips a coin to decide whether to include a particular line from the source text in the subsample. It then writes the subsample as a txt file, for convenience and reproducibility. Each subsample can be further split later on into training, validation and testing sets. 


```r
## Function to create subsample of txt file 
SampleTxt <- function(infile, outfile, seed, inlines, percent, readtype) {
  conn.in <- file(infile, readtype)  # readtype = "r" or "rb"
  conn.out <- file(outfile,"w")  
  # for each line, flip a coin to decide whether to put it in sample
  set.seed(seed)
  in.sample <- rbinom(n=inlines, size=1, prob=percent)
  i <- 0
  num.out <- 0  # number of lines written out to subsample
  for (i in 1:(inlines+1)) {
    # read in one line at a time
    currLine <- readLines(conn.in, n=1, encoding="UTF-8", skipNul=TRUE) 
    # if reached end of file, close all conns
    if (length(currLine) == 0) {  
      close(conn.out)  
      close(conn.in)
      # return number of lines written out to subsample 
      return(num.out)  
    }  
    # while not end of file, write out the selected line to file
    if (in.sample[i] == 1) {
      writeLines(currLine, conn.out)
      num.out <- num.out + 1
    }
  }
}
```

I extract about 5% of the original source text for each subsample. The subsample files are saved in the current working directory.


```r
datalist <- c("../final/en_US/en_US.blogs.txt",
              "../final/en_US/en_US.news.txt",
              "../final/en_US/en_US.twitter.txt")
mypercent <- 0.05
myseed <- 60637

if (!file.exists("./blog.sample.txt")) {
  SampleTxt(datalist[1], "blog.sample.txt", myseed, blog.numlines, mypercent, "r")
}
if (!file.exists("./news.sample.txt")) {
  SampleTxt(datalist[2], "news.sample.txt", myseed, news.numlines, mypercent, "rb")
}
if (!file.exists("./twit.sample.txt")) {
  SampleTxt(datalist[3], "twit.sample.txt", myseed, twit.numlines, mypercent, "r")
}
```

## Analysis of subsample

### Basic summary and importing to dataframe

Check the word and line statistics to see how the subsamples compare with the original text.


```r
sample.numwords <- system("wc -w *.sample.txt", intern=TRUE)  
sample.numlines <- system("wc -l *.sample.txt", intern=TRUE)
sample.longest <- system("wc -L *.sample.txt", intern=TRUE)

# number of words for each dataset
blog.sample.numwords <- as.numeric(gsub('[^0-9]', '', sample.numwords[1]))
news.sample.numwords <- as.numeric(gsub('[^0-9]', '', sample.numwords[2]))
twit.sample.numwords <- as.numeric(gsub('[^0-9]', '', sample.numwords[3]))
# number of lines for each dataset
blog.sample.numlines <- as.numeric(gsub('[^0-9]', '', sample.numlines[1]))
news.sample.numlines <- as.numeric(gsub('[^0-9]', '', sample.numlines[2]))
twit.sample.numlines <- as.numeric(gsub('[^0-9]', '', sample.numlines[3]))
# length of longest line for each dataset
blog.sample.longest  <- as.numeric(gsub('[^0-9]', '',  sample.longest[1]))
news.sample.longest  <- as.numeric(gsub('[^0-9]', '',  sample.longest[2]))
twit.sample.longest  <- as.numeric(gsub('[^0-9]', '',  sample.longest[3]))

# create and display summary table
blog.sample.stats <- c(blog.sample.numwords, blog.sample.numlines, blog.sample.longest,
                      round(blog.sample.numwords/blog.sample.numlines))
news.sample.stats <- c(news.sample.numwords, news.sample.numlines, news.sample.longest,
                      round(news.sample.numwords/news.sample.numlines))
twit.sample.stats <- c(twit.sample.numwords, twit.sample.numlines, twit.sample.longest,
                      round(twit.sample.numwords/twit.sample.numlines))  
sample.stats <- data.frame(rbind(blog.sample.stats, 
                                 news.sample.stats, 
                                 twit.sample.stats))
names(sample.stats) <- c("Sample word count", 
                       "Sample line count", 
                       "Words in longest line", 
                       "Avg words per line")
kable(sample.stats)  # display the above in table format
```

                     Sample word count   Sample line count   Words in longest line   Avg words per line
------------------  ------------------  ------------------  ----------------------  -------------------
blog.sample.stats              1884523               45506                   12403                   41
news.sample.stats              1727402               51087                    2363                   34
twit.sample.stats              1536112              119260                     371                   13

Read in each of the subsample txts as dataframe.

``{r readsubsample}
blog.mini <- readLines()
news.mini <- readLines()
twit.mini <- readLines()
```

### Clean up text

Use the `tm` text mining package in R to clean and analyze the sample text. Convert text to lowercase, remove punctuation

``{r corpus}
library(tm)
# build a corpus, specifying the source as a character vector
blog.corpus <- Corpus(VectorSource(#charactervector))

# convert text to lowercase
blog.corpus <- tm_map(blog.corpus, content_transformer(tolower))
# remove numbers
blog.corpus <- tm_map(blog.corpus, content_transformer(removeNumbers))
blog.corpus <- tm_map(blog.corpus, content_transformer(removePunctuation))
# remove URLs
removeURL <- function(x) {
  gsub("http[[:alnum:]]", "", x)  # won't work with some links e.g. bit.ly
}
blog.corpus <- tm_map(blog.corpus, content_transformer(removeURL))
```

### Find frequent words and word associations

Create a term-document matrix (TDM). In general, a TDM is a mathematical matrix that describes the frequency of terms (words) that occur in a collection of documents (source: Wikipedia). The rows of the matrix correspond to terms, and the columns correspond to the documents. (A document-term matrix has it the other way round, and is essentially the transpose of the TDM.)

My specific term-document matrix will have the columns corresponding to the blog, news, and Twitter texts, and the rows corresponding to the words found in them. 





### Histogram to see word frequency

``{r freq}
datalist <- c("en_US.blogs.txt","en_US.news.txt","en_US.twitter.txt")
mk.lower <- "tr 'A-Z' 'a-z'"
rm.punct <- 'tr -d "-" | tr -d "\'" | tr -s "[:punct:]" " "'
split.uniq.sort.out <- "tr ' ' '\n' | sort | uniq -c | sort -n -r > output.txt"

unix.blog <- paste0(mk.lower," < ",datalist[1]," | ",rm.punct," | ",split.uniq.sort.out)
unix.news <- paste0(mk.lower, " < ", datalist[2], " | ",split.uniq.sort.out)
unix.twit <- paste0(mk.lower, " < ", datalist[3], " | ",split.uniq.sort.out)

freq.blog <- system("tr 'A-Z' 'a-z' < test.txt | less", intern=T)
system("tr 'A-Z' 'a-z' < test.txt | less")
freq.news <- system(unix.news, intern=TRUE)
freq.twit <- system(unix.twit, intern=TRUE)
```


## end

********

#### Notes and citations:

I learned about how to use the `tm` package from these sources:

* http://www.unt.edu/rss/class/Jon/Benchmarks/TextMining_L_JDS_Jan2014.pdf

* http://www.r-bloggers.com/text-mining-the-complete-works-of-william-shakespeare/

* http://www.slideshare.net/rdatamining/text-mining-with-r-an-analysis-of-twitter-data