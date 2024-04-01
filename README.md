# Youtube Data Analysis - AWS

 ## Overview

This project aims to securely manage, streamline, and perform analysis on the structured and semi-structured YouTube videos data based on the video categories and the trending metrics.

## Project Goals
1. Data Ingestion — Build a mechanism to ingest data from different sources.
2. Data lake — We will be getting data from multiple sources so we need centralized repo to store them
3. ETL System — We are getting data in raw format, transforming this data into the proper format
4. Reporting — Build a dashboard to get answers to the question we asked earlier

## Services we will be using
1. Amazon S3: Amazon S3 is an object storage service that provides manufacturing scalability, data availability, security, and performance.
2. AWS Glue: A serverless data integration service that makes it easy to discover, prepare, and combine data for analytics, machine learning, and application development.
3. AWS Lambda: Lambda is a computing service that allows programmers to run code without creating or managing servers.
4. AWS Athena: Athena is an interactive query service for S3 in which there is no need to load data it stays in S3.
5. AWS IAM: This is nothing but identity and access management which enables us to manage access to AWS services and resources securely.
6. QuickSight: Amazon QuickSight is a scalable, serverless, embeddable, machine learning-powered business intelligence (BI) service built for the cloud.

## Dataset Used
This Kaggle dataset contains statistics (CSV files) on daily popular YouTube videos over the course of many months. There are up to 200 trending videos published every day for many locations. The data for each region is in its own file. The video title, channel title, publication time, tags, views, likes and dislikes, description, and comment count are among the items included in the data. A category_id field, which differs by area, is also included in the JSON file linked to the region.

https://www.kaggle.com/datasets/datasnaek/youtube-new

## Architecture Diagram


**Step 1** <br/>
## Data Ingestion
We ingest the kaggle data into the S3 bucket by using the copy command 

```aws s3 cp . s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics_reference_data/ --recursive --exclude "*" --include "*.json" ```
```aws s3 cp CAvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=ca/
aws s3 cp DEvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=de/
aws s3 cp FRvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=fr/
aws s3 cp GBvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=gb/
aws s3 cp INvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=in/
aws s3 cp JPvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=jp/
aws s3 cp KRvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=kr/
aws s3 cp MXvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=mx/
aws s3 cp RUvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=ru/
aws s3 cp USvideos.csv s3://de-youtube-raw-data-us-east-2-dev/youtube/raw_statistics/region=us/
```

**Step 2** <br/>
## Data Lake Creation

We have the raw data stored in the S3 bucket now.
We have some data in .json format and some data in CSV format.
We will then create a Data Catalog using AWS Glue.
AWS Glue will extract the metadata out of this raw data that we ingested and will create a Data Catalog around it.

![Glue Crawler for raw data](https://github.com/srajeevan/Youtube-Data-Analysis---AWS/blob/main/Assets/Glue_Crawler_raw_data.png)

While the csv data can be queried in Athena, the .json data cannot be queried as it is nested.
This raw_data needs to be cleansed for the reporting and for doing aanalysis on the data.
We will need to convert the raw_data into parquet format so that the reporting and analysis can be done.

![Athena UI](https://github.com/srajeevan/Youtube-Data-Analysis---AWS/blob/main/Assets/Athena_UI_raw_data.png)

** Step 3** <br/>
## Data Cleansing & Setting up an ETL

We will use AWS Lambda to cleanse and flatten the .json files in our s3 bucket.
AWS Lambda is a compute service that runs your code in response to events and automatically manages the compute resources, making it the fastest way to turn an idea into a modern, production, serverless applications.Since our dataset is small using AWS Lambda is ideal as well.

![AWS Lambda](https://github.com/srajeevan/Youtube-Data-Analysis---AWS/blob/main/Assets/AWS%20Lambda_json_paquet.png)

Within the lambda function we have defined some environmental variables as shown below.

```
os_input_s3_cleansed_layer = os.environ['s3_cleansed_layer']
os_input_glue_catalog_db_name = os.environ['glue_catalog_db_name']
os_input_glue_catalog_table_name = os.environ['glue_catalog_table_name']
os_input_write_data_operation = os.environ['write_data_operation']
```
We have defined the cleaned glue_catalog_db_name, glue_catalog_table_name etc.
This function will process the nested `item` data structure in the json files and normalize it and extract the required columns and write it into the new S3 location that we mentioned.
This will essentially flatten the raw nested data for us.

```
# Creating DF from content
        df_raw = wr.s3.read_json('s3://{}/{}'.format(bucket, key))

        # Extract required columns:
        df_step_1 = pd.json_normalize(df_raw['items'])

        # Write to S3
        wr_response = wr.s3.to_parquet(
            df=df_step_1,
            path=os_input_s3_cleansed_layer,
            dataset=True,
            database=os_input_glue_catalog_db_name,
            table=os_input_glue_catalog_table_name,
            mode=os_input_write_data_operation
        )
```

The environment variable, s3_cleansed_layer will be S3 location to the cleansed S3 bucket.
We also added trigger to the AWS Lambda so that whenever a new json file gets added to the raw_data S3 location, this function gets triggered and will convert the new json file to parquet format.

![AWS Trigger config](https://github.com/srajeevan/Youtube-Data-Analysis---AWS/blob/main/Assets/Lambda%20Trigger%20config.png)

For the CVS files to be cleansed and added to the new cleansed S3 location, we will be using AWS Glue Visual ETL.

In the AWS Glue console, select Visual ETL where we added a data source node, transformation node and a target node.

![AWS Glue ETL](https://github.com/srajeevan/Youtube-Data-Analysis---AWS/blob/main/Assets/AWS_Glue_ETL.png)

For the dat source node, we choose the AWS Glue Catelog and configure to use the raw cvs files as the source.
In the transformation section we choose the datatypes for the fields etc.
In the targte node, we choose the cleaned S3 bucket location and that is where the parquert files will be stored.
Once we set up the visual ETL it will create the script for the ETL. 

Now we have the cleaned data in the separate S3 bucket which we can use for different reporting purposes.
As an additional step we can even create another ETL which will establish the join between these two cleaned tables and save that in a another S3 bucket which can be provided to the reporting team.
Implementing the joins and storing the final data set for analytics purpose can reduce the compute overhead or can help if the reporting analysts are not technical persons.
Remember to create another Glue crawler to create a Glue Data Catalog for these cleaned data sets. 

Lets create another ETL which gets data from those two cleaned dataset with joins and move that to a final analytics s3 bucket.
This is how the ETL job looks like.

![Analytics ETL](https://github.com/srajeevan/Youtube-Data-Analysis---AWS/blob/main/Assets/final_etl.png)

Run this ETL job and then the joined dataset gets stored to the final analytics S3 bucket.

** Step 4** <br/>
## Reporting
Once you have this analytics data set you can create dashboads for repoting using AWS Quicksight.
Sign up for the standard version and create dataset by giving permissions to the analytics S3 bucket and by connecting to the Athena.

![Analytics dataset](https://github.com/srajeevan/Youtube-Data-Analysis---AWS/blob/main/Assets/quicksight_dataset.png)

Then create dashbaords by creating individual vizualizations.
Here is an exmaple of a dashboard that i created.
![Analytics Dashboard](https://github.com/srajeevan/Youtube-Data-Analysis---AWS/blob/main/Assets/quicksight_dashboard.png)

Thank you.

