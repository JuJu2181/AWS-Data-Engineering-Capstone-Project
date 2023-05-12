# AWS-Data-Engineering-Capstone-Project
## Project Description
- This is the final capstone project for AWS Data engineering course offered by AWS Academy.
- This project includes basic ETL workflow using AWS services like Amazon S3, AWS Glue, Amazon Athena and even Amazon QuickSight for visualization
- Here firstly the data was extracted from a S3 bucket.
- Then the data was transformed to Apache Parquet format using Pandas framework in AWS Cloud9 IDE. 
- The transformed data was then loaded into S3 bucket  which was then queried using S3 select
- Then tables were created using AWS Glue and AWS Glue crawler was used to extract metadata and schema from the data stored in S3 bucket.
- The data in Glue tables were then queried and transformed using Amazon Athena to create views
- Finally Amazon QuickSight was used to visualize the data present in the Views.
- Here the file **Final Capstone Project.md** contains all details about the steps for project implementation as well as various outputs obtained during course of this project. 

## AWS Services Used
- Amazon S3
- AWS Cloud9 IDE
- AWS Glue
- Amazon Athena
- Amazon QuickSight
