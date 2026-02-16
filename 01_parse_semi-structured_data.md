# 📘 Student Assignment: Working with Nested JSON Data

## Objective

The goal of this assignment is to practice working with **semi-structured JSON data**, transform it into a **structured format**, and perform **analytical queries using window functions** to extract meaningful insights.

---

## Task Description

### 1. Dataset Selection

* Find a dataset that:

    * Is stored in **JSON format**
    * Contains a **nested structure** (e.g. arrays, nested objects). The dataset must contain **both nested data and arrays.**
    * Has a size of **more than 10 MB**
* You may use:

    * [Kaggle](https://www.kaggle.com/)
    * Open data portals
    * Any other publicly available source

Examples of nested JSON structures:

* Arrays of objects
* Objects inside objects

---

### 2. Data Loading

* Load the dataset into an analytical environment.
* You may use:

    * DuckDB
    * BigQuery
    * Any other analytical database system that supports JSON

---

### 3. Data Parsing (Mandatory)

* Transform semi-structured JSON data into a **structured format** (tables with columns).
* This includes:

    * Flattening(unnest) arrays
    * Extracting nested fields
    * Casting data types where needed

The final result should be queryable using SQL.

---

### 4. Data Analysis Using Window Functions (Mandatory)

* Use **SQL window functions** to analyze the dataset.
* Provide **at least 2 data insights**.

Examples of insights:

* Top 3 companies by revenue per region
* Ranking users by activity within categories
* Running totals or moving averages over time
* Percentage contribution within a group

Each insight should include:

* SQL query
* Short explanation of the result

---

## Deliverables

* `README.md` (this file, extended with your work)
* SQL scripts or notebooks used for:

    * Loading data
    * Parsing JSON
    * Analysis
* Post it all on your GitHub and add a link to GitHub in Moodle.
---

## Evaluation Criteria

### Obligatory Part — **10 points**

| Criteria                                   | Points |
|--------------------------------------------|--------|
| Dataset > 10 MB with nested JSON structure | 1      |
| Correct data loading                       | 1      |
| Proper parsing of semi-structured data     | 2      |
| Use of window functions                    | 2      |
| At least 2 meaningful data insights        | 2      |
| Knowledge of theory*                       | 2      |
| **Total**                                  | **10** |

_*The Theoretical Questions section will help you prepare for the defense of your theoretical knowledge. The teacher has the right to rephrase the theoretical questions._


### Additional Task — **+2.5 points (Optional)**

Choose **one** of the following:

* Add **data quality checks** (nulls, duplicates, schema validation)
* Visualize results (charts or dashboards)

Clearly document the additional work in the README.

---

## Notes

* SQL clarity and readability matter.
* Reproducibility is important.
* Insights should be logical and data-driven, not trivial aggregations.

---

Below is a **ready-to-add section** for your `README.md`.

### Dataset

**Source:** [Tweets Json file for JSON File handling nad NLTP](https://www.kaggle.com/datasets/pduvvuri0308/tweets-json-file-for-json-file-handling-nad-nltp)

**File:** `Tweets_File.json` (~38 MB)

**Structure:** Twitter tweets with nested objects and arrays:
- nested `user` object (43 fields - profile info, counts, etc.)
- nested `entities` object containing arrays: `hashtags[]`, `user_mentions[]`, `urls[]`, `symbols[]`

---

### Data Loading

Using DuckDB to load the JSON:

```sql
CREATE OR REPLACE TABLE tweets_raw AS
SELECT * FROM read_json_auto(
    'Tweets_File.json',
    format='unstructured',
    maximum_object_size=50000000
);
```

---

### Data Parsing

**1. Flatten main tweet data + extract nested user fields:**

```sql
CREATE OR REPLACE TABLE tweets_parsed AS
SELECT 
    id as tweet_id,
    created_at,
    text as tweet_text,
    source,
    lang,
    retweet_count,
    favorite_count,
    "user".id as user_id,
    "user".name as user_name,
    "user".screen_name as user_screen_name,
    "user".followers_count,
    "user".verified as user_verified,
    entities
FROM tweets_raw;
```

**2. Unnest hashtags array:**

```sql
CREATE OR REPLACE TABLE tweet_hashtags AS
SELECT
    tweet_id,
    user_screen_name,
    created_at,
    unnest(entities.hashtags).text as hashtag
FROM tweets_parsed
WHERE len(entities.hashtags) > 0;
```

**3. Unnest mentions array:**

```sql
CREATE OR REPLACE TABLE tweet_mentions AS
SELECT
    tweet_id,
    user_screen_name as author_screen_name,
    unnest(entities.user_mentions).screen_name as mentioned_screen_name
FROM tweets_parsed
WHERE len(entities.user_mentions) > 0;
```

**4. Unnest urls array:**

```sql
CREATE OR REPLACE TABLE tweet_urls AS
SELECT
    tweet_id,
    unnest(entities.urls).expanded_url as url
FROM tweets_parsed
WHERE len(entities.urls) > 0;
```

---

### Analysis with Window Functions

#### Insight 1: Top Users by Engagement

```sql
WITH user_stats AS (
    SELECT 
        user_screen_name,
        user_verified,
        count(*) as tweets,
        sum(retweet_count) as retweets,
        sum(favorite_count) as favorites,
        sum(retweet_count + favorite_count) as engagement
    FROM tweets_parsed
    GROUP BY user_screen_name, user_verified
)
SELECT 
    row_number() over(order by engagement desc) as rank,
    user_screen_name,
    user_verified,
    tweets,
    engagement,
    sum(engagement) over(order by engagement desc) as running_total,
    round(100.0 * engagement / sum(engagement) over(), 2) as pct_total
FROM user_stats
ORDER BY engagement desc 
LIMIT 15;
```

**Result:** @JoeBiden dominates with 75% of all engagement (14M retweets+favorites). Top 3 accounts (Biden, bennyjohnson, RexChapman) control 90% of total. All top 12 are verified - only positions 13-15 show non-verified users with ~0.03% each.

---

#### Insight 2: Hashtag Popularity Distribution

```sql
WITH ht_stats AS (
    SELECT 
        lower(hashtag) as tag,
        count(*) as cnt,
        count(distinct user_screen_name) as users
    FROM tweet_hashtags
    GROUP BY lower(hashtag)
    HAVING count(*) >= 2
)
SELECT 
    rank() over(order by cnt desc) as rank,
    tag,
    cnt,
    users,
    ntile(10) over(order by cnt desc) as decile,
    round(100.0 * cnt / sum(cnt) over(), 2) as pct,
    round(100.0 * sum(cnt) over(order by cnt desc) / sum(cnt) over(), 2) as cumul_pct
FROM ht_stats
ORDER BY cnt desc 
LIMIT 20;
```

**Result:** #amas leads with 18.3% of hashtag usage. Mix of entertainment (squidgame, bts, adele30), sports (gobills, epl, chargers), and news (waukesha, rittenhouse). Top 20 hashtags cover ~56% of all usage - classic long-tail distribution.

---

## 📚 Theoretical Questions

Answer the following **10 theoretical questions**. Answers should be concise, technically correct, and supported by examples where appropriate.

1. **Big Data vs Traditional Data Processing**
   Explain how Big Data differs from traditional data processing systems. Why do data volume, velocity, and distribution fundamentally change system design?

2. **The 5Vs of Big Data**
   Interpret the 5Vs (Volume, Velocity, Variety, Veracity, Value). Explain how each V impacts data ingestion, storage, processing, and analytics.

3. **Data Variety and Schema Design**
   Explain how data variety affects schema design. Why are schema-on-read approaches often used for semi-structured and unstructured data?

4. **Data Types Classification**
   Classify data into structured, semi-structured, and unstructured. Provide examples of each and explain how they are typically stored and processed.

5. **Processing Semi-Structured Data**
   Describe common techniques for processing semi-structured data (e.g. JSON, Avro, Parquet). How do analytical databases handle nested data?

6. **Massively Parallel Processing (MPP)**
   Describe MPP architectures. How do they distribute data and parallelize computation? What are the benefits and challenges of this approach?

7. **OLTP vs OLAP Systems**
    Differentiate OLTP and OLAP workloads in terms of:

    * Query patterns
    * Typical technologies used
    * Data storing

---

Good luck!
