---
title: "Scrape Data Scientist's Skills from Indeed.com"
date: "Dec 11, 2018"
output: github_document
---

## UPDATE Dec 2018 

#### Python surpasses R, Spark outranks Hadoop for DS Job in both Chicago and USA Nationwide

![Python surpass R, Spark made to 4th for DS Job in Chicago](figure/DS_Chicago_20181211.png)

![Python surpass R, Spark made to 5th for DS Job in USA Nationwide](figure/DS_USA_20181211.png)

## Introduction (all below are from Dec 2016)

During my pursuit of becoming a data scientist, I've stumbled upon this specific topic multiple times

* [The huge debate of R vs Python (by Domino Data Lab)](https://blog.dominodatalab.com/video-huge-debate-r-vs-python-data-science/)
* [Choosing R or Python for data analysis? (by DataCamp)](https://www.datacamp.com/community/tutorials/r-or-python-for-data-analysis#gs.7SYWFwY)

I am a huge fan of R, because of my King [Hadley Wickham](http://hadley.nz/)'s packages-_dplyr_, _ggplot2_, _stringr_, everything that's incredibly well developed and easy to use If you say R is a data science skill, I would argue that not until you manage to use all of the packages by Hadley. 

On the other end, Python is also popular in data science and had outranked R to some degree. There are also great packages such as _Scikit-Learn_, _Bokeh_, _Pandas_, etc that make Python a perfect tool for data science work. Even more, Python is a better than that, it's a general-purpose language. Besides DS, you can use it to develop game, application with GUI, Webapp/Website, etc. So, yes, Python is awesome!

However, the reality to me is that, none of my previous work experience guided me towards using Python. I tried it personally, but had some difficult time managing Python pacakges, virtual environments in my Windows centric ecosystem, although Anaconda with Spyder had worked well. I am still inclined to learn Python, my internal debate is that should I slowly move from R to Python, or should I concentrate my time on R. A dilemma of breadth vs. depth.

3 days ago, I came across this great analysis by Jesse Steinweg-Woods, Ph.D-[Web Scraping Indeed for Key Data Science Job Skills](https://jessesw.com/Data-Science-Skills/). The analysis was implemented in Python, and dated 2 years ago. Currently I don't see similar analysis in R therefore I am going to do that, and see what's changed since 2015 where R has evolved rapidly especially with [Microsoft's acquisition of Revolusion Analytics in 2015](https://blogs.technet.microsoft.com/machinelearning/2015/04/06/microsoft-closes-acquisition-of-revolution-analytics/)

My approach is to use R to implement a iterative web scraping from all data scientist job listings on Indeed.com. I'll elaborate with code further.

## Indeed's Advanced Search

Indeed.com is the largest US online job board, with a google-like interface and search engine, you can drill down with an [advanced search](https://www.indeed.com/advanced_search?q=Data+Scientist&l=Chicago%2C+IL&sort=date) where you can put in your search criteria. Here I want to specify job title that contains "_Data Scientist_" (this will include senior, junior or other possible prefix/suffix), location equals to "_Chicago, IL_" where I am located, and _exclude staffing agencies_ checked to remove potential duplicates. Additionally I select display _50_ listings per page sorted by _date_. This will help in our loop operation later on. 

![](figure/AdvSearch.PNG)

After you click __Find Jobs__, it yields a URL with all the specified fields and brings you to the result page. Take a closer look:

```
https://www.indeed.com/jobs?as_and=&as_phr=&as_any=&as_not=&as_ttl=%22Data+Scientist%22&as_cmp=&jt=all&st=&sr=directhire&salary=&radius=25&l=Chicago%2C+IL&fromage=any&limit=50&sort=date&psf=advsrch
```
These following fields were assigned and translated into html

* __&as_ttl=__ _%22Data+Scientist%22_
* __&sr=__ _directhire_
* __&l=__ _Chicago%2C+IL_

There are several other fields can be assigned, but for our Data Scientist specific job analysis, these are good enough. You are encouraged to do your own criteria trying the other fields. We can start writing some code now.

## Getting Started with R


```r
# Load packages
library(rvest)
library(stringr)
library(dplyr)
library(ggplot2)

# Indeed Search Words
job_title <- "\"Data+Scientist\""
location <- "Chicago%2C+IL"

# use advanced search to get 50 results in a page
BASE_URL <- 'https://www.indeed.com'
ADV_URL <- paste0('https://www.indeed.com/jobs?as_and=&as_not=&as_cmp=&jt=all&st=&salary=&sr=directhire&radius=25&fromage=any&limit=50&sort=date&psf=advsrch&as_any=&as_phr=&as_ttl=', job_title, '&l=', location)
cat(ADV_URL)
```

```
## https://www.indeed.com/jobs?as_and=&as_not=&as_cmp=&jt=all&st=&salary=&sr=directhire&radius=25&fromage=any&limit=50&sort=date&psf=advsrch&as_any=&as_phr=&as_ttl="Data+Scientist"&l=Chicago%2C+IL
```
Now we've found the URL to the search result, we can proceed to next step.

## Scrape the Search Result

The URL directs you to the first page of the search result, which lists total number of jobs, first 50 jobs, and links to the 2nd and following pages at the bottom. 

![](figure/StartPage1.PNG)
![](figure/StartPage2.PNG)

I am using Hadley Wickham's rvest pacakge for scraping operations. I am still learning it, but my impression is that this package has many signature features as other packages from Hadley. For example, the chain operation using %>% makes life easier. 


```r
# get the html file from search url
start_page <- read_html(ADV_URL)

# get the total job count 
job_count <- unlist(strsplit(start_page %>% 
                               html_node("#searchCount") %>%
                               html_text(), split = ' ')) 
job_count <- as.numeric(str_replace_all(job_count[length(job_count)],',',''))
cat('Total job count: ', job_count)
```

```
## Total job count:  88
```

Scraping the job links and page link requires deeper knowledge in html. I spent quite some time to extract those two parts out. Jobs are under html nodes:_h2_ _a_, links for search result pages are more complex, I had to use XPath to find them out. It's almost a must knowing the basic of html/css. Good lesson for me. Hadley actually pointed out a useful tool [SelectorGadget](http://selectorgadget.com/) but I didn't find it to be effective on Indeed's website. Indeed's html appears to be unstructured. Not sure if they do that on purpose to prevent scarping or not. Anyhow, the code is much simpler than the process to reach them properly.


```r
# Get start page job URLs
links <- start_page %>%
  html_nodes("h2 a") %>%
  html_attr('href')

# Get result page links
page.links <- start_page %>%
  html_nodes(xpath = '//div[contains(@class,"pagination")]//a') %>%
  html_attr('href')
```

## Scrape the Job Descriptions

Now we've collected the job links. Without programming, I would click into the links one by one, read the descriptions line by line. Find out if I am having a fit, by looking through the skills section. Slow, inefficient. This time I am going to do it in an automated fashion. It's essentially processing a series of text files. Also because I am only looking at job skills that are comprised by certain keywords. I simply need to convert the html paragraphs into individual words. If I am finding a keyword match, I'll count it once and only once (appearing multiple times in one single job description doesn't increase the importance). By doing this iterating over all jobs, we will gather a total count(occurrence) of each skill among all jobs.

Before that, I need to come up with a list of job skills that are most popular/commonly used in Data Scientist Job. 


```r
KEYWORDS <- c('Hadoop','Python','\\bSQL', 'NoSQL','\\bR\\b', 'Spark', 'SAS', 'Excel', 'AWS', 'Azure', 'Java', 'Tableau')
```

Note that I used \\\\b before 'SQL' to prevent double count on 'NoSQL'. 'R' is creating some trouble here, as 'R' as a single letter can be shown in many other words, for example. 'R' can be found in location code 'OR' which is Oregan state. 'Relocation required' can also contribute falsely because it appears in the beginning of the phrase. So using \\\\b to wrap it around would enforce a perfect match of one single letter 'R'.

Following is the function that scrapes job description and computes total job count of each skill 


```r
# Clean the raw html - removing commas, tabs, line changers, etc  
clean.text <- function(text)
{
  str_replace_all(text, regex('\r\n|\n|\t|\r|,|/|<|>|\\.'), ' ')
}

# Given running total dataframe and links to scrape skills and compute running total
ScrapeJobLinks <- function(res, job.links){
  for(i in 1:length(job.links)){
    job.url <- paste0(BASE_URL,job.links[i])
    
    Sys.sleep(1)
    cat(paste0('Reading job ', i, '\n'))
    
    tryCatch({
      html <- read_html(job.url)
      text <- html_text(html)
      text <- clean.text(text)
      df <- data.frame(skill = KEYWORDS, count = ifelse(str_detect(text, KEYWORDS), 1, 0))
      res$running$count <- res$running$count + df$count
      res$num_jobs <- res$num_jobs + 1
    }, error=function(e){cat("ERROR :",conditionMessage(e), "\n")})
  }
  return(res)
}
```

## Actual Scraping

Now we have the function ready, the links to scrape are ready. Let's run the procedures.


```r
# For display purpose, we also need the \\b removed from the keyword set
KEYWORDS_DISPLAY <- c('Hadoop','Python','SQL', 'NoSQL','R', 'Spark', 'SAS', 'Excel', 'AWS', 'Azure', 'Java', 'Tableau')

# Create running total dataframe
running <- data.frame(skill = KEYWORDS_DISPLAY, count = rep(0, length(KEYWORDS_DISPLAY)))

# Since the indeed only display max of 20 pages from search result, we cannot use job_count but need to track by creating a num_jobs
num_jobs <- 0

# Here is our results object that contains the two stats
results <- list("running" = running, "num_jobs" = num_jobs)

if(job_count != 0){
  cat('Scraping jobs in Start Page\n')
  results <- ScrapeJobLinks(results, links)
}

for(p in 1:length(page.links)-1){
  
  cat('Moving to Next 50 jobs\n')
  
  # Navigate to next page
  new.page <- read_html(paste0(BASE_URL, page.links[p]))
  
  # Get new page job URLs
  links <- new.page %>%
    html_nodes("h2 a") %>%
    html_attr('href')
  
  # Scrap job links
  results <- ScrapeJobLinks(results, links)
}
```

Let's bring up the result sorted in descending order of occurrence:


```r
# running total
print(arrange(results$running, -count))
```

```
##      skill count
## 1        R    53
## 2   Python    47
## 3      SQL    36
## 4     Java    32
## 5      SAS    23
## 6    Excel    22
## 7   Hadoop    21
## 8    Spark    17
## 9  Tableau    11
## 10   NoSQL     8
## 11     AWS     6
## 12   Azure     2
```

It's more informative to calculate the percentage of apperances, and visualize it using ggplot.


```r
# running total count as percentage
results$running$count<-results$running$count/results$num_jobs

# Reformat the Job Title and Location to readable form
jt <- str_replace_all(job_title, '\\+|\\\"', ' ')
loc <- str_replace_all(location, '\\%2C+|\\+',' ')

# Visualization
p <- ggplot(results$running, aes(reorder(skill,-count), count)) + geom_bar(stat="identity") + 
  labs(x = 'Skill', y = 'Occurrences (%)', title = paste0('Skill occurrences(%) for ', jt, ' in ', loc)) 
p + scale_y_continuous(labels = scales::percent, breaks = seq(0,1,0.1)) 
```

![plot of chunk unnamed-chunk-8](figure/unnamed-chunk-8-1.png)

## Takeaway

* R outranks Python in my limited(86) sample space. Nearly 80% of the job postings contain R in their descriptions. It's no doubt the most wanted skill for data scientists in Chicago
* Python followed R as the second most popular, which is kind of surprising given that Python has been known a more in-demand skill than R
* Tableau is not as popular as I've seen on job postings
* __I should increase my sample size__ by looking at entire US

## Nationwide

All you need is to change one line of code and re-run the whole script.


```r
location <- "Nationwide"
```

But expanding it to Nationwide yields 2000+ listings and Indeed's only showing the first 20 pages of the search results which limits our total job possibly being scraped to be under 1000 (20 pages x 50 pages). I have not found a way to generate the rest of openings. But 1000 sample size is still considered very good for our purpose. The only problem is that processing it takes much longer because there are thousands of html files to scrape. Mine took longer than 15 min for 1000ish iterations. YMMV. So be patient in your experiment.

![](figure/Nationwide.png)

## Data Analyst instead of Data Scientist?

I also wonder how much a Data Analyst's job would differ from Data Scientist's job? To some degree, Data Analyst is a junior data science role. 


```r
job_title <- "\"Data+Analyst\""
```

![](figure/DataAnalystSkills.png)

## Summary

* R consistently outranks Python, with a thin edge though. This is unlike what [Jesse's finding](https://jessesw.com/Data-Science-Skills/) had told us 2 years ago
* SQL follows at third place and 2nd place for Data Analyst jobs. We know for a fact that, SQL is a minimum requirement for junior roles
* SAS/Excel still display strong presense in job descriptions. For Data Analyst, Excel is the most wanted. However be cautious that Data Analyst can be a title doing different job than data science. Junior analyst at financial institutions can also be titled Data Analyst. This is an overly generic title
* Java, in my opinion is the next language I should study after R and Python. I should compare C family languages in my keywords but just for now, I think this comparison still shows me how strong Java skill plays a role in job market
* Tableau is surprisingly weak; my impression was that they had been really popular in Business Intelligence related job

## Follow up?

* Readers may try job title = 'Data Engineer' to see how the Skill ranking changes. My intuition is that Python would take a big lead than R in that area.
* I would add in C family languages to really compare Java (apples to apples)

