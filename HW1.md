# HW1

In this assignment, you will build a simple model to predict how much a taxi driver could expect to earn in tips based on data from all 12 months of 2024 from the [NYC Taxi Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)(Yellow Taxi Trip Records). At a high level you will need to: 
- look at the schema of the data in the dataset,
- decide what variables you wish to correlate to tip amount,
- load the dataset into Google BigQuery,
- craft some SQL queries to extract information from this dataset, and
- finally use Google Colab to train the model. 

The goal of this assignment is to give you experience working with cloud SQL engines in a simple pipeline, so we will not be grading on the robustness of your model or how sound your choice of variables are. Rather, we will be looking to see that you're crafting SQL queries that will do much of the work of extracting information for you before training your model, and that you've effectively gotten some experience working with cloud databases. 

Once you've completed the assignment, be sure to 
1. put all relevant code (SQL queries, python code, etc.) into the provided Github repo for `HW1` 
2. also write a `README.md` which summarizes the choices you made and how your SQL queries were designed.

Finally, include in your writeup your experience using the cloud/any difficulties you may have had, as well as any feedback you have on the assignment itself as a way for us to improve assignments going forward and for future classes. 

## Dataset
The NYC Taxi dataset provides one file for each month representing a relational table, with each row representing a different taxi cab trip and each column providing information about the trip. You can download all 12 files for 2024 from the link above. Parquet files are often compressed and not human readable. Nearly all cloud databases offer support for data stored in Parquet files, however, so you shouldn't need to do any transformations before using this data in the cloud.

To investigate the schema of these files on your local machine, an easy way to do this is with duckdb, which is an in-process analytical database that can query Parquet files directly without needing to load them into a database.. [DuckDB](https://duckdb.org/docs/api/overview) has a number of client APIs. You can install the package simply via `python3 -m pip install duckdb`. Then, assuming your downloaded parquet file is at path `foo/january.parquet`, run the command: 

```
import duckdb 

duckdb.sql("DESCRIBE SELECT * FROM 'foo/january.parquet'")
```
Which should output the schema of the dataset: 
```
┌───────────────────────┬─────────────┬─────────┬─────────┬─────────┬─────────┐
│      column_name      │ column_type │  null   │   key   │ default │  extra  │
│        varchar        │   varchar   │ varchar │ varchar │ varchar │ varchar │
├───────────────────────┼─────────────┼─────────┼─────────┼─────────┼─────────┤
│ VendorID              │ BIGINT      │ YES     │ NULL    │ NULL    │ NULL    │
│ tpep_pickup_datetime  │ TIMESTAMP   │ YES     │ NULL    │ NULL    │ NULL    │
│ tpep_dropoff_datetime │ TIMESTAMP   │ YES     │ NULL    │ NULL    │ NULL    │
│ passenger_count       │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ trip_distance         │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ RatecodeID            │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ store_and_fwd_flag    │ VARCHAR     │ YES     │ NULL    │ NULL    │ NULL    │
│ PULocationID          │ BIGINT      │ YES     │ NULL    │ NULL    │ NULL    │
│ DOLocationID          │ BIGINT      │ YES     │ NULL    │ NULL    │ NULL    │
│ payment_type          │ BIGINT      │ YES     │ NULL    │ NULL    │ NULL    │
│ fare_amount           │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ extra                 │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ mta_tax               │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ tip_amount            │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ tolls_amount          │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ improvement_surcharge │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ total_amount          │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ congestion_surcharge  │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
│ airport_fee           │ DOUBLE      │ YES     │ NULL    │ NULL    │ NULL    │
├───────────────────────┴─────────────┴─────────┴─────────┴─────────┴─────────┤
│ 19 rows                                                           6 columns │
└─────────────────────────────────────────────────────────────────────────────┘
```
Before loading the data into BigQuery, run the following two DuckDB queries on one of the Parquet files (for example January.parquet) and include what's happening in these queries and the result of these queries in your README.md.

```
# Query 1
SELECT
    COUNT(*) AS total_trips,
    ROUND(AVG(trip_distance),2) AS avg_trip_distance,
    ROUND(AVG(fare_amount),2) AS avg_fare,
    ROUND(AVG(tip_amount),2) AS avg_tip
FROM 'January.parquet';

# Query 2
SELECT
    payment_type,
    ROUND(AVG(tip_amount),2) AS avg_tip,
    COUNT(*) AS num_trips
FROM 'January.parquet'
GROUP BY payment_type
ORDER BY payment_type
LIMIT 5;
```

## Designing Your Model
The goal is to build a model so that a taxi driver could understand how much to expect to make in tips. This dataset has a `tip_amount` column. It is up to you to pick what variables you wish to use as predictors or not. You will not be graded on the soundness of your decision, just that you made a decision and provide some justification for it (for example, choosing only `trip_distance` because you believe that a solid heuristic is that tips will be strongly correlated with the distance travelled by cabs in New York City). Be sure to include this justification in your writeup.  

## Setting up the cloud environment
We will be using Google Cloud for this class. New accounts should be able to receive $300 in credits, as well as access to a free-tier of cloud usage. If you have already used these credits please contact us so we can make sure you have the resources you need to complete the assignments. 

## Loading data into BigQuery 
First you will want to upload the 2024 taxi data to Google Cloud Storage. Navigate to the Google Cloud Storage window, create a bucket with a name of your choosing (default options are fine). You can then upload the Parquet files directly to this bucket from the web UI. If you wish to do this via CLI, you can follow the guide [here](https://cloud.google.com/sdk/docs/install).  

Next, once data is uploaded, navigate to BigQuery via the left-hand navigation bar (enable the BigQuery API if it prompts you to). Once there, navigate to BigQuery studio, where it should list a project. **Make sure to note the project ID as this will be important in the next step.** 

Click the explorer icon on the left, and under your project id select "Datasets" and then "create dataset". Make sure the location type matches what's listed on your GCS bucket (i.e. multi-region, US). 

To run SQL queries on the dataset, we first need to create tables in BigQuery. This can be done by connecting BigQuery to the data stored in Google Cloud Storage (GCS). There are two ways to do this: using an external table or creating a native BigQuery table. Each option has different advantages and disadvantages.
For this part, you'll need to pick one of two options, provide some information on what you think the tradeoffs might be for each in the writeup, and follow the guides below. 

Data can be kept in Google Cloud Storage and BigQuery makes an EXTERNAL TABLE which points to this data. When you run queries against this table, it is accessing data in GCS. 

Data can also be loaded into BigQuery. This requires more time, as data must be uploaded from GCS into BigQuery, and is then stored in BQ's properietary internal format rather than the open Parquet format in GCS. 

- [External Tables](https://cloud.google.com/bigquery/docs/external-data-cloud-storage#sql)
- [Load Data Internal](https://cloud.google.com/bigquery/docs/loading-data)

## Starting a colab notebook and querying BigQuery data
We're going to do most of our work in [Google Colab](https://colab.research.google.com). For those unfamiliar, Colab is very similar to Jupyter notebooks or other notebook interfaces. If you're having particular trouble with it, please come to office hours or reach out for help. 

Open a Colab notebook using the same google account as your google cloud account. Suppose that our project id is `project-1`, and we created dataset `Data34200` and table `January` for the first month's Parquet file. 

```
#@title Enter Google Cloud/BigQuery Project ID
project_id = 'project-1' #@param{type:"string"}

# Package used for interfacing w/ BigQuery from Python
from google.cloud import bigquery

# Create BigQuery client
bq_client = bigquery.Client(project = project_id)
```

Then, create a new codeblock to actually run a SQL query. The %% line tells Colab that we're going to execute a SQL query against the project specified, and to load the results of the query in a Pandas DataFrame called `results`. This query just grabs all tip_amounts from the January table ordered by descending tip amount. We can then create a new code block and use `results` 
```
%%bigquery results --project {project_id}
SELECT tip_amount FROM `project-1.Data34200.January` ORDER BY tip_amount DESC;
```

This lets you then run queries against your BigQuery dataset, get the results in a python environment, and then train whatever models you'd like. You can import other python packages like numpy, sklearn, and so on as well within a Colab notebook. 

Using this, get started and build your data pipeline to train your model!

## Checking costs 
Finally, go through your Google cloud billing window and look at what the costs were of running these different services. You can look at the amount of billable bytes for each query in BigQuery. There's no task involved here, just seeing how the cloud charges you, and noticing what's easy and transparent and what's harder to see and manage.

## Writeup
In your README, please include the following sections: 
- details on what variables you chose for the model,
- what SQL queries you wrote to extract information, including the output of the EXPLAIN SQL command on those queries
- Explanation and Output of the two SQL queries using DuckDB
- what model you trained and what your reasoning was. 
- Include some reasoning on what you think the difference between external tables vs. internal tables in BigQuery would be in terms of some performance metrics.
- Finally, include any feedback you have on how difficult the assignment was, any unexpected difficulties you hit, and any other general feedback you may have. 

Also, submit the executed colab notebook along with the write up.
