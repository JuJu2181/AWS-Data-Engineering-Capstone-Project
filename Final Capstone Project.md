### Objectives
-   Launch and configure an AWS Cloud9 integrated development environment (IDE) instance.
-   Run SQL queries against a single file by using the Amazon S3 Select feature of Amazon Simple Storage Service (Amazon S3).
-   Transform CSV-formatted data files to the Apache Parquet format and upload them to Amazon S3.
-   Create an AWS Glue crawler to infer the structure of the data.
-   Use Amazon Athena to query the data.
-   Create an Athena view.
-   Use Amazon QuickSight to visualize the data.

### Dataset
The Sea Around Us website provides a dataset with extensive historical information about fisheries in all parts of every ocean globally. The data includes information about yearly fishery catches from 1950 to 2018.

The data can be downloaded in CSV format from the [Sea Around Us](https://www.seaaroundus.org/data/) website. The dataset includes columns of information for each year, including which countries caught which types of fish in which areas. The data also indicates how many tonnes of fish were caught and what the value of the catch was, measured in 2010 US dollars.

To understand the data, it will be helpful to understand what is meant by _open seas_ areas and _EEZ_ areas:

-   **Open seas (also called high seas):** Areas of the ocean that are at least 200 nautical miles away from any single country's shoreline. The resources, including the fish, in these areas are generally accepted as not belonging to any one country.
-  **Exclusive Economic Zones (EEZs):** Areas within 200 nautical miles of a country's shoreline. Each country typically claims exclusive access to the resources in the zones, including the fish within them.

### Scenario
You have been tasked to create the infrastructure to host fishing data so that data analysts in your organization can create reports about fishing impact in the open seas. You have decided to build the infrastructure in your AWS account and test it by using three data files from the Sea Around Us dataset.

In this capstone project, you will work with three data files from the Sea Around Us website:

-   The first file contains data from _all open seas areas_.
-   The second file contains data from a _single open seas area_ in the Pacific ocean, referred to as _Pacific, Western Central_, which is not far from Fiji and many other countries.
-   The third file contains data from the _EEZ_ of a single country (Fiji), which is near the Pacific, Western Central open seas area.

### Initial Environment
![[Pasted image 20230502084509.png]]

### Final Architecture
![[Pasted image 20230502084522.png]]

### Tasks done
### 1. Configuring the development environment
- To observe details of CapstoneGlueRole we access IAM 
	- 4 policies found: AWSQuickSightAthenaAccess, AmazonS3FullAccess,  AmazonAthenaFullAccess, AWSGlueServiceRole
- Create an AWS cloud9 environment
	-  environment: CapstoneIDE, new Ec2 instance (t2.micro) deploy instance to Capstone VPC in Capstone public subnet. Choose SSH else error occurs, other settings are default
-  Create two S3 buckets with following settings
	- Region: us-east-1, first bucket: data-source-#### and second bucket: query-results-####; #### is random number, other settings are default
- Download three .csv source data files in Cloud9 IDE
```
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACDSCI-1-66873/lab-capstone/s3/SAU-GLOBAL-1-v48-0.csv
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACDSCI-1-66873/lab-capstone/s3/SAU-HighSeas-71-v48-0.csv
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACDSCI-1-66873/lab-capstone/s3/SAU-EEZ-242-v48-0.csv
```
- Observe the column header row and first five rows of data in SAU-GLOBAL-1-v48-0.csv
```
head -6 SAU-GLOBAL-1-v48-0.csv
```
Following Output is obtained
```
year,fishing_entity,fishing_sector,catch_type,reporting_status,gear_type,end_use_type,tonnes,landed_value
1950,"Albania",Industrial,Discards,Unreported,bottom trawl,Direct human consumption,145.0013231580,212571.939749628
1950,"Albania",Industrial,Discards,Unreported,pelagic trawl,Direct human consumption,0.9863122230,1445.9337189179998
1950,"Albania",Industrial,Discards,Unreported,purse seine,Direct human consumption,0.4728285250,693.16661765
1950,"Albania",Industrial,Landings,Reported,bottom trawl,Direct human consumption,742.8529325409,1089022.3991050033
1950,"Albania",Industrial,Landings,Reported,bottom trawl,Other,0.7435965291,1090.11251161662
```
Analysis: Among other details, each line of data in this dataset includes:
The year that the fishing occurred
The country (fishing_entity) that did the fishing
The tonnes of fish caught that year by that country
The value in 2010 US dollars (landed_value) of the fish caught that year by that country
- When we run ` wc -l SAU-GLOBAL-1-v48-0.csv` it returns no of lines in dataset
`561675 SAU-GLOBAL-1-v48-0.csv`
The dataset includes reported and "best guess" data for all fishing that occurred in the global high seas (meaning, _not_ in any one country's EEZ) between 1950 and 2018.
- Convert the SAU-GLOBAL-1-v48-0.csv file to Parquet format.
First install pandas pyarrow and fastparquet
```
sudo pip3 install pandas pyarrow fastparquet
```
Then to convert the file to parquet format
```
# Start the python interactive shell 
python
# Use pandas to convert the file to parquet format
import pandas as pd
df = pd.read_csv('SAU-GLOBAL-1-v48-0.csv')
df.to_parquet('SAU-GLOBAL-1-v48-0.parquet')
exit()
```
- Finally upload the data in parquet format to data-soure bucket using AWS CLI
```
 aws s3 cp SAU-GLOBAL-1-v48-0.parquet s3://data-source-0001
```
Check in S3 console to see new file uploaded succesfully
##### Summary
Here in this first task, we firstly configured AWS Cloud9 environment and created 2 buckets in S3. Then using Cloud9 terminal we downloaded the csv files from source. We then used pandas to convert data to parquet format and finally uploaded the parquet file to S3 bucket. Till this taks 0$ were expended

### 2. Querying a single file with S3 Select
In Amazon S3 console we use S3 select to run the following query
```
SELECT fishing_entity, gear_type, tonnes FROM s3object s limit 10;
```
Selected output as CSV so the output is
```
Albania,bottom trawl,145.001323158
Albania,pelagic trawl,0.986312223
Albania,purse seine,0.472828525
Albania,bottom trawl,742.8529325409
Albania,bottom trawl,0.7435965291
Albania,pelagic trawl,91.7104345512
Albania,pelagic trawl,0.0918022368
Albania,purse seine,47.2355696275
Albania,purse seine,0.0472828525
Albania,subsistence fishing gear,5.865919086
```
Summary: Simply queried dataset, no cost incurred

### 3. Using AWS Glue crawler and querying multiple files with Athena
Here our aim is to query the data stored in multiple files. For this we configure AWS Glue crawler to discover the structure of data and then use Athena to query the data
- Firstly using head command in cloud9 terminal observe column header row and first few lines of data from SAU-HighSeas-71-v48-0.csv
```
head -5 SAU-HighSeas-71-v48-0.csv
```
Output:
```
area_name,area_type,year,scientific_name,common_name,functional_group,commercial_group,fishing_entity,fishing_sector,catch_type,reporting_status,gear_type,end_use_type,tonnes,landed_value
"Pacific, Western Central",high_seas,1950,"Marine fishes not identified","Marine fishes nei","Medium demersals (30 - 89 cm)","Other fishes & inverts","Japan",Industrial,Discards,Unreported,longline,Discards,908.3175635957,1331593.5482313475
"Pacific, Western Central",high_seas,1950,"Marine fishes not identified","Marine fishes nei","Medium demersals (30 - 89 cm)","Other fishes & inverts","USA",Industrial,Discards,Unreported,longline,Discards,19.2128353413,28166.016610289986
"Pacific, Western Central",high_seas,1950,"Marine pelagic fishes not identified","Pelagic fishes","Medium pelagics (30 - 89 cm)","Other fishes & inverts","Japan",Industrial,Discards,Unreported,longline,Discards,3.8182340507,5597.531118260252
"Pacific, Western Central",high_seas,1950,"Marine pelagic fishes not identified","Pelagic fishes","Medium pelagics (30 - 89 cm)","Other fishes & inverts","Philippines",Industrial,Landings,Reported,hand lines,Direct human consumption,96.9876281672,142183.8628931179
```
It can be seen that this file has same columns of previous file along with some additional columns
- Convert this Highseas dataset file to parquet format and upload to data-source bucket
- To convert to parquet format
```
# Start the python interactive shell 
python
# Use pandas to convert the file to parquet format
import pandas as pd
df = pd.read_csv('SAU-HighSeas-71-v48-0.csv')
df.to_parquet('SAU-HighSeas-71-v48-0.parquet')
exit()
```
- To upload the data to S3 bucket
```
 aws s3 cp SAU-HighSeas-71-v48-0.parquet s3://data-source-0001
```
Check S3 bucket to verify if data was succesfully uploaded or not
- Now create an AWS Glue database and AWS Glue crawler with following settings
```
database: fishdb
crawler: fishcrawler
crawler role: CapstoneGlueRole
Output: fishdb
crawler frequency: On demand
```
To create database=> glue console => databases => create database => fishdb
To create crawler => glue console => Crawlers => create crawler => name: fishcrawler => next => add data source => from this account => browse s3 => data-source-XXXX => choose existing IAM role => CapstoneGlueRole => target database: fishdb => Crawler schedule: frequency: On demand => create crawler
- Then run the crawler to create a table that contains metadata in AWS Glue database, also verify that the expected table is created
To run
fishcrawler => Run crawler => check status => when it changes from running to completed tables will be created
To verify that table is created
Databases => fishdb => see tables
a table for data_source_0001 is created with following schema
![[Pasted image 20230502095222.png]]
- To confirm that the table properly categorized the data, we use Athena to run SQ queries against each column in new table
- Before running SQL queries in Athena, configure Athena Query editor to output data to query-results bucket
Athena => Query editor => Settings => Browse S3 => query-results-00001 => save
Go to editor to run queries
Example query:
```
SELECT DISTINCT area_name FROM fishdb.data_source_0001;
```
Output:
![[Pasted image 20230502104316.png]]
**Note:** The example query returns two results. For this column, every row in the dataset contains either the value "Pacific, Western Central" (for rows pulled from _SAU-HighSeas-71-v48-0.parquet_) or a null value (for rows pulled from _SAU-GLOBAL-1-v48-0.parquet_).
Null values indcates the data from all high seas areas
- Now that the data table is defined, run queries to confirm that it provides useful results
- To find the value in US dollars of a ll fish caught by country Fiji from Pacific, Western, Central high areas since 2001, organized by year
```
SELECT year, fishing_entity AS Country, CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValuePacificWCSeasCatch
FROM fishdb.data_source_0001
WHERE area_name LIKE '%Pacific%' and fishing_entity='Fiji' AND year > 2000
GROUP BY year, fishing_entity
ORDER By year
```

Output:
![[Pasted image 20230502103453.png]]
```
Code Explanation

This code is an SQL query that retrieves data from a database table called <FMI_1> that contains information on the landed value of fish caught in the Pacific waters by different countries. Here is an explanation of each line of the code:

SELECT year, fishing_entity AS Country, CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValuePacificWCSeasCatch
This line is the SELECT statement that specifies the columns that will be retrieved from the table. In this case, the query will return three columns: the year, the fishing entity (which will be renamed as "Country" in the output), and the sum of the landed value of fish caught in the Pacific waters by Fiji. The landed value is first calculated as a sum using the SUM function and then converted to a double-precision floating-point number using CAST. Finally, it is converted to a decimal number with a precision of 38 and a scale of 2 using the DECIMAL function.

FROM <FMI_1> => in our case fishdb.data_source_0001
This line specifies the name of the table from which the data will be retrieved. In this case, the table is called <FMI_1>.

WHERE area_name LIKE '%Pacific%' and fishing_entity='Fiji' AND year > <FMI_2>
This line is the WHERE clause that specifies the conditions that must be met for a row to be included in the output. The condition requires that the area_name column contains the word "Pacific", the fishing_entity column equals "Fiji", and the year is greater than a parameter that must be passed to the query as <FMI_2> => 2000 as we want data since 2001.

GROUP BY year, fishing_entity
This line specifies that the results should be grouped by year and fishing_entity. This means that the output will show the sum of the landed value of fish caught by Fiji in each year.

ORDER By year
This line specifies that the results should be sorted in ascending order based on the year column. This means that the output will be sorted from the oldest to the most recent year.
```
Challenge: Find the value in US dollars of all fish caught by the country Fiji from all high seas areas since 2001, organized by year. In your output results, name the US dollar value column `ValueAllHighSeasCatch`
```
SELECT year, fishing_entity AS Country, CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueAllHighSeasCatch
FROM fishdb.data_source_0001
WHERE area_name IS NULL and fishing_entity='Fiji' AND year > 2000
GROUP BY year, fishing_entity
ORDER By year
```
Output:
![[Pasted image 20230502104051.png]]
- Create a view based on query
Create => view from query => name: challenge
Summary:
Firstly converted highseas dataset to parquet format and then uploaded to S3 bucket. Then created glue database and glue crawler. Ran the crawler to create table from data stored in S3 bucket. Finally used Athena to run queries on created table. Also no money deducted yet

### 4. Transforming a new file and adding it to dataset
Here we add the _SAU-EEZ-242-v48-0.csv_ data file to your dataset in Amazon S3.This file has a few column names that don't match the data that you already added to your S3 bucket. However, the data in these columns _does_ align with the existing data. We need to modify column names before we add data to bucket
- Analyse data structure of _SAU-EEZ-242-v48-0.csv_ file. Compare its columns with columns of other files
- We can use head command to check the column names

**For SAU-HighSeas-71-v48-0.csv dataset**
Columns: area_name,area_type,year,scientific_name,common_name,functional_group,commercial_group,fishing_entity,fishing_sector,catch_type,reporting_status,gear_type,end_use_type,tonnes,landed_value

Single row:
"Pacific, Western Central",high_seas,1950,"Marine fishes not identified","Marine fishes nei","Medium demersals (30 - 89 cm)","Other fishes & inverts","Japan",Industrial,Discards,Unreported,longline,Discards,908.3175635957,1331593.5482313475

**For SAU-EEZ-242-v48-0.csv**
Columns:
area_name,area_type,data_layer,uncertainty_score,year,scientific_name,fish_name,functional_group,commercial_group,country,fishing_sector,catch_type,reporting_status,gear_type,end_use_type,tonnes,landed_value

Single row:
Fiji,eez,Reconstructed domestic catch,2.0,1950,Marine fishes not identified,Marine fishes nei,Medium demersals (30 - 89 cm),Other fishes & inverts,Fiji,Subsistence,Landings,Reported,subsistence fishing gear,Direct human consumption,1100.0,5009747.023464293

Different columns:
New columns: data_layer, uncertainity_score
Changed columns => common_name -> fish_name, fishing_entity -> country

**Analysis:** The data in the _fish_name_ column needs to be merged with the data in the _common_name_ column from the HighSeas dataset. Likewise, the data in the _country_ column needs to be merged with the data in the _fishing_entity_ column from the HighSeas dataset.

- Use Python pandas library to fix column names. Then convert EEZ file to Parquet format
```
# Make a backup of the file before you modify it in place
cp SAU-EEZ-242-v48-0.csv SAU-EEZ-242-v48-0-old.csv

# Start the python interactive shell
python
import pandas as pd

# Load the backup version of the file
data_location = 'SAU-EEZ-242-v48-0-old.csv'

# Use Pandas to read the CSV into a dataframe
df = pd.read_csv(data_location)

# View the current column names
print(df.head(1))

# Change the names of the 'fish_name' and 'country' columns to match the column names where this data appears in the other data files already in your data-source bucket
df.rename(columns = {"fish_name": "common_name", "country": "fishing_entity"}, inplace = True)

# Verify the column names have been changed
print(df.head(1))

# Write the changes to disk
df.to_csv('SAU-EEZ-242-v48-0.csv', header=True, index=False)
df.to_parquet('SAU-EEZ-242-v48-0.parquet')
exit()
```
-  Upload new EEZ data to data-source bucket
Check S3 console to verify uploaded data
- Update table metadata with additional columns by running AWS Glue crawler again
Before running crawler there are 15 fields, after running crawler schema updated can be seen in athena editor tab
Running crawler first time took 2 min 42 sec, but now during updating it only took 53 sec

- Run some queries in Athena
- To verify the values in the _area_name_ column,
```
SELECT DISTINCT area_name FROM fishdb.data_source_0001;
```
Output:
![[Pasted image 20230502111008.png]]
With the addition of the EEZ file to the dataset, this query now returns three results, including the result for rows where the _area_name_ column doesn't have any data.
- To find the value in US dollars of all fish caught by Fiji _from the open seas_ since 2001, organized by year, use the following query:
```
SELECT year, fishing_entity AS Country, CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueOpenSeasCatch 
FROM fishdb.data_source_0001
WHERE area_name IS NULL AND fishing_entity='Fiji' AND year > 2000
GROUP BY year, fishing_entity
ORDER By year
```
Output:
![[Pasted image 20230502111122.png]]
- To find the value in US dollars of all fish caught by Fiji _from the Fiji EEZ_ since 2001, organized by year, use the following query:
```
SELECT year, fishing_entity AS Country, CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueEEZCatch
FROM fishdb.data_source_0001
WHERE area_name LIKE '%Fiji%' AND fishing_entity='Fiji' AND year > 2000
GROUP BY year, fishing_entity
ORDER By year
```
Output:
![[Pasted image 20230502111230.png]]
- To find the value in US dollars of all fish caught by Fiji _from either the Fiji EEZ or the open seas_ since 2001, organized by year, use the following query:
```
SELECT year, fishing_entity AS Country, CAST(CAST(SUM(landed_value) AS DOUBLE) AS DECIMAL(38,2)) AS ValueEEZAndOpenSeasCatch 
FROM fishdb.data_source_0001
WHERE (area_name LIKE '%Fiji%' OR area_name IS NULL) AND fishing_entity='Fiji' AND year > 2000
GROUP BY year, fishing_entity
ORDER By year
```
Output:
![[Pasted image 20230502111335.png]]
**Analysis:** If your data is formatted well and the AWS Glue crawler properly updated the metadata table, then the results that you get from the first two queries in this step should add up to the results that you get from the third query.

For example, if you add the _2001 ValueOpenSeasCatch_ value and the _2001 ValueEEZCatch_ value, the total should equal the _2001 ValueEEZAndOpenSeasCatch_ value. If your results are consistent with this description, then it is a good indication that your solution is working as intended.
From output this is confirmed that system working properly

- Create a view in Athena, which will be useful to visualize the data
```
CREATE OR REPLACE VIEW MackerelsCatch AS
SELECT year, area_name AS WhereCaught, fishing_entity as Country, SUM(tonnes) AS TotalWeight
FROM fishdb.data_source_0001
WHERE common_name LIKE '%Mackerels%' AND year > 2014
GROUP BY year, area_name, fishing_entity, tonnes
ORDER BY tonnes DESC
```
Verify by viewing the data in Data panel, under views => preview mackerelscatch view
Output:
![[Pasted image 20230502111611.png]]

Summary:
Firstly, we used pandas to rename the columns of a dataset to consistent format i-e columns of dataset already stored in bucket, Then converted it to parquet format and uplaoded to s3 bucket. After uploading to s3, we ran crawler again to update the schema of table. Finally using Athena we ran various queries of the table and also created view at last for visualization
Till no 0$ been spend 

### 5. Visualizing the results in Quicksight
Here we will use QuickSight to create a bar chart from our dataset
- Create QuickSight Account
QuickSight console => Signup for QuickSight => Enterprise => Continue
On Create your QuickSigth Account page => QuickSight Account name: capstone-0001 => Notification email address: your email address => IAM role: use existing role: aws-quicksight-service-role-v0 => Finish => Goto Amazon QuickSight after created
Output:
![[Pasted image 20230502113011.png]]
- Create a new QuickSight dataset
Navigation pane => Datasets => Choose New dataset => Athena 
Data source name:  `MackerelsView` , Athena workgroup: primary => validate connection => create data source 
Configure => Catalog: AwsDataCatalog => Database: fishdb => Tables: mackerelscatch => select => Directly query your data => visualize
- Create a chart in QuickSight
Fields list => Drag year into empty chart => expand field wells area => verify that year is in Y axis field well, if not drag it there now => From the **Fields list**, drag **country** to the **Group/Color** field well. => drag totalweight to value field
Adjust chart settings => choose two arrow icons in top right corner of chart to expand it => Double-click the chart title, "Sum of Totalweight by Year and Country." => place cursor in text field that reads Default and enter following as new chart title
`Tonnes of mackerel caught by year by country in the Fiji EEZ and in the Pacific, Western Central open seas` => save
On left side of chart choose arrow icon above year => choose format => choose number format that doesn't contain commas or period => choose arrow icon on sheet1 tab and rename the sheet to Fish data
Output:
![[Pasted image 20230502114030.png]]

- Reduce the amount of data shown in chart
Return to Athena query editor and modify view your created so that it only includes data for six fishing entities that caught the highest TotalWeight of mackerel fish in any year betwen 2013 and 2018
```
CREATE OR REPLACE VIEW MackerelsCatch AS
SELECT year, area_name AS WhereCaught, fishing_entity as Country, SUM(tonnes) AS TotalWeight
FROM fishdb.data_source_0001
WHERE common_name LIKE '%Mackerels%' AND year BETWEEN 2013 AND 2018
GROUP BY year, area_name, fishing_entity
HAVING fishing_entity IN (
  SELECT fishing_entity
  FROM fishdb.data_source_0001
  WHERE common_name LIKE '%Mackerels%' AND year BETWEEN 2013 AND 2018
  GROUP BY fishing_entity
  ORDER BY SUM(tonnes) DESC
  LIMIT 6
)
ORDER BY TotalWeight DESC;
```
```
Description

CREATE OR REPLACE VIEW MackerelsCatch AS
This line creates or replaces a database view named "MackerelsCatch". A view is a virtual table that is based on the result of a SELECT statement.

SELECT year, area_name AS WhereCaught, fishing_entity as Country, SUM(tonnes) AS TotalWeight
This line specifies the columns that will be retrieved from the data source. Specifically, it retrieves the year, area name (which is renamed to "WhereCaught"), fishing entity (which is renamed to "Country"), and the sum of the weight of mackerel caught in tonnes, which is also given an alias "TotalWeight".

FROM fishdb.data_source_0001
This line specifies the data source table "fishdb.data_source_0001" from which the data will be retrieved.

WHERE common_name LIKE '%Mackerels%' AND year > 2014
This line specifies the filtering conditions that the data source must meet for the view. Specifically, it filters to only select rows where the common name column contains the string "Mackerels" and the year is greater than 2014.

GROUP BY year, area_name, fishing_entity, tonnes
This line groups the data by year, area name, fishing entity, and tonnes columns, so that the results are aggregated by those columns.

ORDER BY tonnes DESC
This line sorts the aggregated data by the total weight in tonnes in descending order.

HAVING fishing_entity IN (
  SELECT fishing_entity
  FROM fishdb.data_source_0001
  WHERE common_name LIKE '%Mackerels%' AND year BETWEEN 2013 AND 2018
  GROUP BY fishing_entity
  ORDER BY SUM(tonnes) DESC
  LIMIT 6
)
This line applies a filter to the aggregated data, using a subquery to first identify the top six fishing entities with the highest total weight of mackerel caught between 2013 and 2018. The subquery groups the data by fishing entity, sorts it by the sum of the weight in descending order, and then limits the results to the top six. The main query then selects only rows where the fishing entity is in the list of top six entities.

ORDER BY TotalWeight DESC;
This line orders the final results of the view by the total weight in tonnes in descending order.
```

Output
![[Pasted image 20230502115034.png]]

Summary:
Created quick sight account then added quicksight dataset from Athena view. Then we create bar chart in quicksight for default athena view. We then change the view in athena to reduce amount of data shown in chart and again display the chart in QuickSight. Even at end it shows 0$ spend.

Capstone project completed successfully
