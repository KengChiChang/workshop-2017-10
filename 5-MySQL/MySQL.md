---
title: "MySQL on Server"
author: "Chunhsiang Chang"
date: "2017-10-24"
---

# Why MySQL

-   Free and popular 
-   Support most programming language
-   Easy to understand

# Installation and Management

-   Download and install
    
    ```bash
    yum install mysql mysql-server
    service mysqld start
    ```

-   Default access account: root
    -   User account could be created and granted different privileges

# Demo (1)

-   Log in 
-   Use database
-   Show tables
-   Show fields

```sql
SHOW 
  FIELDS 
FROM 
  `1000_page_2016-01`;
```

# Demo (2)

-   Get size of a table

```sql
SELECT 
  table_name AS `Table`, 
  ROUND(((data_length + index_length) / 1024 / 1024/ 1024), 2) 
    AS `Size in GB` 
FROM information_schema.TABLES 
WHERE table_schema = "1000_page_us_user_like_post" AND 
      table_name = "1000_page_2016-01";
```

# Import User Like Page Data by Month (1)

1.  Import library "dbConnect"
2.  Connect to MySQL
 
```r
con = dbConnect(MySQL(), 
                user = "root", 
                password = "*******", 
                dbname = "1000_page_us_user_like_post", 
                host = "localhost")
```

3.  Create a table schema for month data 
 
```r
dbGetQuery(con, paste("create table", 
                      year_month, 
                      "(user_id varchar(20), 
                        post_id varchar(41), 
                        post_created_date_CT date);"))
```

# Import User Like Page Data by Month (2)

4.  Read daily data into R to remove duplicate and save it as another file
5.  Expand the month data table by "load data infile"

```r
dbGetQuery(con, paste("load data local infile", 
                      like_date_table, 
                      "ignore into table", 
                      year_month,     
                      "fields terminated by ',' 
                       lines terminated by '\n' 
                       ignore 1 lines ;"))
```
  
# Extract Weekly Data

1.  Selecting the date range
    - Union needed if you need data from two tables
2.  Group your data by `user_id`, `page_id`
3.  Turn it into `user_like_page` data by "Group_Concat"
    - Be aware of `group_concat_max_len`