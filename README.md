# FB Data Workshop

- 2017-10-17 and 2017-10-24 at National Taiwan University Applied Micro
- Keng-Chi Chang and Chunhsiang Chang

---

## Structures of Facebook Open Data and Data on Server

How Facebook open data looks like, and the organizations of the data on our server.

[Slides: data.pdf](https://github.com/NTUUSFB/workshop-2017-10/raw/master/data.pdf)

### Facebook Graph API

- Only data related to fan pages are publicly available
    - Posts on fan pages
    - Reactions to / comments of / public shares of these posts
    - Users do the above
    - Fan page likes fan pages
- A full list of variables can be found through [Graph API Documentation](https://developers.facebook.com/docs/graph-api/reference/)

### Page

Ex. [Donald J. Trump](http://www.facebook.com/153080620724)

- page_id: "153080620724"
- page_name: "Donald J. Trump"
- page_fan_count: 22,813,525
- page_url: `www.facebook.com/DonaldTrump` or `/[page_id]`
- page_category: "Public Figure"

### Post (Link / Photo / Video / Status)

Ex. [9GAG: Official White House Photographer Reveals His Favourite Photos Of Obama](http://www.facebook.com/21785951839_10155113971791840)

- post_id: "153080620724" or "21785951839_153080620724"
- post_type: "link"
- post_name: "Official White House Photographer Reveals His Favourite Photos Of Obama"
- post_description: "Click to see the pic and write a comment..."
- post_caption: "9gag.com"
- post_link: `http://9gag.com/gag/ajqEV90?ref=fbp`
- post_reactions: 1297326, post_likes: 1149630
- post_comments: 20093, post_shares: 209506
- post_created_time: "2016-11-11T07:35:00+0000" ([ISO 8601](https://en.wikipedia.org/wiki/ISO_8601))

### Reaction

LIKE / WOW / HAHA / SAD / ANGRY / THANKFUL

- post_id: "57972945858_10154109988750859"
- user_id: "766918176681835"
- user_name: "Trent Porter"
- reaction_type: "LIKE"

### Comment

- comment_id: "10154022206161680" or "10154022159491680_10154022206161680"
- comment_message: "I'm getting really sick of seeing Ann Coulter-Lite's crazed, glassy-eyed face plastered all over the place, and Trump hasn't even been sworn in yet."
- user_id: "100011100251277"
- post_id: "62317591679_10154022159491680"
- comment_created_time: "2016-11-22T05:47:18+0000"

### On Server: Page

- Top *1000 pages* talking about Trump & Clinton in August 2016
    - Weight by likes:comments:shares = 1:7:14
    - Include major news outlets, puglic figures, interest groups
- US national *politician* fan page
    - Former (last one), candidate, and present listed in Wikipedia
    - Senators, Representatives, Governors

### Note on Pages

- 366,840,068 unique users ever liked a post from above pages in 2015 and 2016
- 29,410,568 unique users ever liked a post from national politicians in 2015 and 2016, we call them *US political users*
- Overlapping pages: Tim Kaine, Bernie Sanders, U.S. Senator Bernie Sanders, Elizabeth Warren, U.S. Senator Elizabeth Warren, Ted Cruz, Rand Paul, Governor Jan Brewer, Al Franken

### On Server: Reaction

- 1000 page: 
    - Repeated capture every 20 minutes
        - 2016-09-29 to 2016-11-21
        - Record the first timestamp when observing one's reaction
    - *Likes* by *US political users* for posts from 2015-01-01 to 2016-11-30
- Politician: 2015-01-01 to 2016-11-30
    - *Likes* by *US political users* for posts from 2015-05-01 to 2016-11-30
    - Remove reactions on overlapping pages from politician folder
    - For reactions on these pages, go to 1000 page

### On Server: Post & Comment

- Post:
    - 1000 page: 2015-01-01 to 2017-04-08
    - Politician: 2015-01-01 to 2016-11-30
- Comment:
    - Comment on post of 1000 page from 2015-01-01 to 2016-11-30

### Directories

- Reference: [Gentzkow and Shapiro, Chapter 4](http://web.stanford.edu/~gentzkow/research/CodeAndData.xhtml)
- `build`: Data cleaning part
    - `input`: Data download from Google BigQuery
    - `output`: Data after basic cleaning & combining for people to use
- `analysis-[some-project]`: Data analysis part
    - `input`: Symbolic link pointing to `build/output`
    - `code`: Code to run data analysis
    - `temp`: Other data source that feeds into your code, logs, ...
    - `output`: Figures, tables, ...
- Just copy `analysis-[some-project]` to wherever you want 
    - Rename it, links are preserved, set it as working directory
    - Code in `code`, read from `input` and `temp`, export to `output`

### Why This Structure?

- No need to copy data, they are extremely huge
- Will not sync large data to your local computer if use Dropbox
- Easy to (and should always) use relative paths in your code

```r
library(tidyverse)
setwd("~/analysis-ideology")
read_csv("input/page/1000-page-info.csv")
```

```python
import os, pandas
os.chdir("~/analysis-ideology")
pandas.read_csv("input/page/1000-page-info.csv")
```

```stata
cd "~/analysis-ideology"
insheet using "input/page/1000-page-info.csv", clear
```

---

## Useful SQL and Google Cloud for Facebook Data Wrangling

Essence of SQL, Google BigQuery, and Google Cloud Storage, with an application to Facebook data wrangling.

[Slides: sql.pdf](https://github.com/NTUUSFB/workshop-2017-10/raw/master/sql.pdf)

### Motivation

- Calculate number of shared users between pages

----------------------------------------------------------
               Trump      FoxNews     Clinton     NYTimes
----------- ----------- ----------- ----------- ----------
      Trump     2243216     1078513       32731      25842
    FoxNews     1078513     2449174       87084      63401
    Clinton       32731       87084     1768980     367021
    NYTimes       25842       63401      367021     986613
----------------------------------------------------------

- For large matrix, can run out of memory even with sparse matrix
- 0.3 billion users $\times$ 2000 pages $\times$ 5% non-zero elements $\times$ 4 Byte $\div$ 1024 (KB) $\div$ 1024 (MB) $\div$ 1024 (GB) $\approx$ 112 GB use of memory
- Strategy: User SQL to group by users first

### SQL (Structured Query Language)

- Industry standard for manipulating relational data
- Useful for columnwise calculation
- Many derivatives: MySQL, NoSQL, MongoDB, Google BigQuery
- More functions: See [Google BigQuery Reference](https://cloud.google.com/bigquery/docs/reference/legacy-sql)

### Basic Structure (SELECT, FROM, WHERE)

```sql
SELECT
  data.id AS user_id,
  NTH(2, SPLIT(src, "/")) AS post_id,
FROM
  [ntue-data-sci:US_Election_Dataset_Local.old_reactions_201501_to_201611]
WHERE
  data.type = "LIKE"
```
 $\rightarrow$ [ntufbdata:USdata.old_1000_user_post_like]

- In Google BigQuery: [ProjectID:DatasetID.TableID]

### GROUP BY & Nested Subquery

```sql
SELECT
  user_id,
  GROUP_CONCAT(page_id) AS like_pages,
  GROUP_CONCAT(STRING(like_time)) AS like_times,
FROM (
  SELECT
    user_id,
    page_id,
    COUNT(*) AS like_time,
  FROM (
    SELECT
      user_id,
      NTH(1, SPLIT(post_id, "_")) AS page_id,
    FROM
      [ntufbdata:USdata.old_1000_user_post_like])
  GROUP BY
    user_id,
    page_id)
GROUP BY
  user_id
```

### Time Selection & Merging Data (JOIN)

```sql
SELECT
  F1.user_id AS user_id,
  F1.post_id AS post_id,
FROM [ntufbdata:USdata.old_1000_user_post_like] AS F1
INNER JOIN [ntufbdata:1000_page_post.201501_to_201611_all] AS F2
ON
  F1.post_id = F2.post_id
WHERE
  DATE(post_created_date_CT) >= DATE("2016-10-01") AND
  DATE(post_created_date_CT) <= DATE("2016-11-07")
```

### Workflow for Google BigQuery

1. Use SQL to extract data you want
    - Web UI, R, command line `bq`
2. [Export to Google Cloud Storage](https://cloud.google.com/bigquery/docs/exporting-data)
    - Provide a URI 
```bash
gs://ntuusfb/us_user_like/us_user_like_1000_page_and_politician_times_\
     201501_to_201611_all/*.csv
```
    - For tables > 1GB, use a folder since it will be split
3. [Download to server](https://cloud.google.com/storage/docs/object-basics)
    - Web UI, command line `gsutil`

### Use `gsutil` to Download Data

- Follow the [Quickstart](https://cloud.google.com/storage/docs/quickstart-gsutil)
    1. Install [Google Cloud SDK](https://cloud.google.com/sdk/docs/)
    2. First time: Run `gcloud init` to configure and select project
    3. Run `gsutil cp gs://[source] [destination]` to download

```bash
gsutil cp gs://ntuusfb/us_user_like/us_user_like_1000_page_and_politician_\
          times_201501_to_201611_all/*.csv 
          ~/usfb/analysis-ideology/temp/us_user_like_1000_page_and_politician_\
          times_201501_to_201611_all/
```

### General Advice for BigQuery

- Google BigQuery is pricing by *columns*
- Use subqueries for intermediate tables
    - Will save *a lot* if intermediate tables are large
- Keep a record of everything you run
    - Not only for others
    - But also for your future self
- Come up with a naming scheme *at the start* of your project
    - Project, Dataset, Table, Variable names
    - Data types (use strings for every IDs)

---

## Use R to Control Google BigQuery

Get rid of Web UI to control BigQuery using R.

[Slides: bigQueryR.pdf](https://github.com/NTUUSFB/workshop-2017-10/raw/master/bigQueryR.pdf)

### Get Authentication

```r
install.packages("bigQueryR")
library(bigQueryR)
bqr_auth()
```

### Pulling Data from BigQuery to R

```r
pull_data = bqr_query(projectId = "ntufbdata",
                      datasetId = "politician_info",
                      query = "SELECT page_name, 
                                      PC1_std AS ideology_score, 
                                      party
                                 FROM politician_pca
                                WHERE PC1_std > 0 AND page_name IS NOT null")
length(which(pull_data$party=="Republican"))
length(which(pull_data$party=="Uncertain"))
```

### Save to Another Table

```r
job_1 = bqr_query_asynch(projectId = "ntufbdata", 
                         datasetId = "politician_info",
                         query = "SELECT PC1_std, 
                                         page_id 
                                    FROM politician_pca",
                         destinationTableId = "politician_PC1_std")
```
check the result of your query

```r
bqr_get_job("ntufbdata", job_1$jobReference$jobId)
```

### Using For Loop

```r
ideology_vec = c(-3, -2, -1, 0, 1, 2)
for (i in 1:5) {
  job = bqr_query_asynch(projectId = "ntufbdata", 
                         datasetId = "bigQueryR_try",
                         query = paste0("SELECT PC1_std, 
                                                page_id 
                                           FROM politician_PC1_std_copy 
                                          WHERE PC1_std BETWEEN", 
                                                ideology_vec[i-1], 
                                                "AND", 
                                                ideology_vec[i]),
                          destinationTableId = paste0("politician_group", i-1), 
                          writeDisposition = "WRITE_TRUNCATE")
  bqr_wait_for_job(job)
}
```

### Export to Google Cloud Storage

```r
for (i in 1:5) {
  job_extract = bqr_extract_data(projectId = "ntufbdata", 
                                 datasetId = "bigQueryR_try",
                                 tableId = paste0("politician_group", i-1),
                                 cloudStorageBucket = 
             paste0("gs://ntuusfb/bigQueryR_try/politician_group", i-1,".csv"))
  bqr_wait_for_job(job_extract)
}
```
