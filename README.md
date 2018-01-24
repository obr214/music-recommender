# Music Recommender System

### Description:
The goal of this project was to create a Music Recommender system using Spark and MLlib.
The data set was obtained and published by Audioscrobbler and contains interactions between users and songs: “User X played song Y”. This type of data is often called implicit feedback. There are no ratings between users and songs. 

For the recommender system, a collaborative filtering algorithm was used, specifically Alternating Least Squares (ALS), popularized around the time of the Netflix Prize. 
The basic premise is that two users might both like the same song because they play many other same songs. 

For more information about the project, to review the original code, and to download the datasets, please visit:
https://github.com/sryza/aas


### Personal Modifications to the project:
* The project was modified to run on Google Cloud Platform. 
* The data locations were changed from HDFS to Cloud Storage.
* The best model is saved for later use.
* Five recommendations for all users are created using the model’s method:`recommendForAllUsers()` 
* These recommendations were saved as parquet file and then exported to JSON files.

### Recommendations data
The JSON files populate a BigQuery dataset using the following command:
```
bq load [--source_format=NEWLINE_DELIMITED_JSON] \
<destination_table> <data_source_uri> [<table_schema>]
```
With the following schema:
```
[
  {
    "mode": "NULLABLE",
    "name": "user",
    "type": "INTEGER"
  },
  {
    "fields": [
      {
        "mode": "NULLABLE",
        "name": "artist",
        "type": "INTEGER"
      },
      {
        "mode": "NULLABLE",
        "name": "rating",
        "type": "FLOAT"
      }
    ],
    "mode": "REPEATED",
    "name": "recommendations",
    "type": "RECORD"
  }
]
```
Once the data set has been saved, some queries were performed:
* Get the Recommendations for a particular USER

```
SELECT user, recomms
FROM `<project_id>.<dataset_id>.user_recommendations`, 
UNNEST(recommendations) recomms
WHERE user = '<user_id>'
```
* Get the Top Recommendations for all USERS
```
WITH
  top_rating AS (
  SELECT
    user,
    MAX(recomms.rating) AS maxrating
  FROM
    `<project_id>.<dataset_id>.user_recommendations`,
    UNNEST(recommendations) recomms
  GROUP BY
    user
  ORDER BY
    user )
SELECT
  r.user,
  r.artist,
  r.rating
FROM
  top_rating tr
INNER JOIN (
  SELECT
    user,
    rec.artist,
    rec.rating
  FROM
    `<project_id>.<dataset_id>.user_recommendations`,
    UNNEST(recommendations) rec) r
ON
  tr.user = r.user
  AND tr.maxrating = r.rating
ORDER BY user
```
* Get the most Recommended Artists
``` 
SELECT
  recomms.artist,
  COUNT(recomms.artist) artist_count
FROM
  `<project_id>.<dataset_id>.user_recommendations`,
  UNNEST(recommendations) recomms
GROUP BY
  recomms.artist
ORDER BY
  artist_count DESC
```

* Get the TOP N Recommendations for all USERS (Using Rank Method)

```
SELECT
  user,
  artist,
  rating,
  ranking
FROM (
  SELECT
    user,
    recomms.artist,
    recomms.rating,
    RANK() OVER (PARTITION BY user ORDER BY recomms.rating DESC) ranking
  FROM
    `<project_id>.<dataset_id>.user_recommendations`,
    UNNEST(recommendations) recomms
  ORDER BY
    user,
    ranking )
WHERE
  ranking <= <TopN>
ORDER BY
  user,
  rating DESC
```
