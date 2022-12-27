# DP-203 Notes:

### 2. Data Storage:
- azure storage account
- Azure Blob Storage vs. Azure Data Lake Storage Gen2
- storage account -> access keys
- storage account -> shared access signature (SAS)
- storage account -> Redundancy
    - LRS
    - ZRS
    - GRS
    - Read-access GRS
    - GZRS
    - Read-access GZRS
- storage account -> access tiers
    - Hot: frequently access data
    - Cool: infrequently access data
    - archive: not availavble at storage account level, available at indivudual blob level
- storage account -> Lifecycle management



### 3. T-SQL
- when using WHERE clause in a data warehouse -> use PARTITIONS in data warehouse -> to increase efficiency of SQL queries


### 4. Azure Synapse Analytics
##### 4.1 Azure Synapse
- features of azure synapse analytics
- compute options: 
    - serverless SQL pool
    - dedicated SQL pool
        - DWU - datawarehousing unit
    - apache spark pool

##### 4.2 External tables & Serverless SQL pool / Dedicated SQL pool
- steps to create and use external table
    - create a database in synabse workspace
    - create a database master key with encryption by password. this will be used to protect Shared Access Signature
    - use SAS to authroize the use of ADLS account, crate database scoped credential SasToken
    - create external datasource (can be hadoop, blob storage, ADLS)
    - create external file format object that defines the external data (file format = DELIMITEDTEXT or PARQUET)
    - define the external table
    - use the table for analysis (there will be a lag, as data is stored in external source)

    ```sql
    CREATE DATABASE SCOPED CREDENTIAL AzureStorageCredential
    WITH
    IDENTITY = 'ADLS-name',
    SECRET = 'ACCESS_KEY';

    -- In the SQL pool, we can use Hadoop drivers to mention the source

    CREATE EXTERNAL DATA SOURCE log_data
    WITH (    LOCATION   = 'abfss://data@ADLSNAME.dfs.core.windows.net',
            CREDENTIAL = AzureStorageCredential,
            TYPE = HADOOP
    )

    -- Drop the table if it already exists
    DROP EXTERNAL TABLE [logdata]

    -- Here we are mentioning the file format as Parquet

    CREATE EXTERNAL FILE FORMAT parquetfile  
    WITH (  
        FORMAT_TYPE = PARQUET,  
        DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'  
    );

    -- Notice that the column names don't contain spaces
    -- When Azure Data Factory was used to generate these files, the column names could not have spaces

    CREATE EXTERNAL TABLE [logdata]
    (
        [Id] [int] NULL,
        [Correlationid] [varchar](200) NULL,
        [Operationname] [varchar](200) NULL,
        [Status] [varchar](100) NULL,
        [Eventcategory] [varchar](100) NULL,
        [Level] [varchar](100) NULL,
        [Time] [datetime] NULL,
        [Subscription] [varchar](200) NULL,
        [Eventinitiatedby] [varchar](1000) NULL,
        [Resourcetype] [varchar](1000) NULL,
        [Resourcegroup] [varchar](1000) NULL
    )
    WITH (
    LOCATION = '/parquet/',
        DATA_SOURCE = log_data,  
        FILE_FORMAT = parquetfile
    )

    /*
    A common error can come when trying to select the data, here you can get various errors such as MalformedInput

    You need to ensure the column names map correctly and the data types are correct as per the parquet file definition

    */


    SELECT * FROM [logdata]
    ```

##### 4.3 Loading data into data warehouse (SQL pool)
- using T-SQL COPY statement
- using azure Synapse pipeline, can perform tranformations on data before copying the data to the warehouse
- using Polybase to define external tables, use external tables to create the internal tables
- **1 - load data using COPY statement**
    - never use the admin account for load operations
    - create a seperate user for load operations
    - best practice - create a workload group - to segregate CPU percentage across groups of users
    - grant permissions
    - csv: 
        ```sql
        COPY INTO logdata FROM 'https://appdatalake7000.blob.core.windows.net/data/Log.csv'
        WITH
        (
        FIRSTROW=2
        )
        ```
    - parquet: 
        ```sql
        COPY INTO [logdata] FROM 'https://jibsyadls.blob.core.windows.net/data/raw/parquet/*.parquet'
        WITH
        (
        FILE_TYPE='PARQUET',
        CREDENTIAL=(IDENTITY= 'Shared Access Signature', SECRET='sv=2021-06-08&ss=b&srt=sco&sp=rl&se=2022-12-22T14:08:01Z&st=2022-12-22T06:08:01Z&spr=https&sig=WU%2FFh62PcCSx7wSEuccKC%2FdlgAwIto2aHJVXMiPovfM%3D')
        )
        ```

- **2 - load data using external table using polybase**

    ```sql
    CREATE TABLE [logdata]
    WITH
    (
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED INDEX (id)   
    )
    AS
    SELECT  *
    FROM  [logdata_external];
    ```
- **3 - BULK INSERT from Azure Synapse**
    - in azure data studio -> connect to external data
    - select ADLS gen2 (new linked service --- part of azure data factory)
    - creating connection to data store
    - fill the details and create
    - before copying data -> specific permissions have to be given to the storage account
    - go to Access Control in the data storage account
    -  add a role assignment
    - role = storage blob data contributor (allows user to read and write data)
    - choose azure admin account & save
    - go to azure data studio ->  linked -> select connected ADLS -> select the file
    - select the file, right click and select bulk load
    - this wil automatically create the SQL script

##### 4.4 Designing a data warehouse
- fact table 
    - contains measurable facts
    - usually large in size
- dimension table
- note: there is no concept of foreign key in sql data warehouse in dedicated SQL pool in azure synapse
- star schema
- ideal practice while building dimension tables:
    - dont have NULL values for properties in dimension table, wont give desired results when using reporting tools
    - try to replace NULL with some default value
- surrogate key: new key added in dimension table when mixing two different data sources with same primary keys
- can use "Identity column" feature in azure synapse to generate unique ID
- right approach -> take different tables & create fact table in synapse itself using ADF to migrate tables from azure database to azure synapse


##### 4.5 Transfer data from azure sql database to azure synapse
- create table structure in synapse
-  open synapse studio -> Integrate -> Copy data tool
- make connection with the source (azure sql database) - select the server, database and table details
- make connection to the target (azure synapse) - select the required details
- select the staging area in ADLS2 or blob - used by the copy statement

##### 4.6 Reading JSON files from ADLS/Blob
```sql
-- Here we are using the OPENROWSET Function

SELECT TOP 100
    jsonContent
FROM
    OPENROWSET(
        BULK 'https://appdatalake7000.dfs.core.windows.net/data/log.json',
        FORMAT = 'CSV',
        FIELDQUOTE = '0x0b',
        FIELDTERMINATOR ='0x0b',
        ROWTERMINATOR = '0x0a'
    )
    WITH (
        jsonContent varchar(MAX)
    ) AS [rows]

-- The above statement only returns all as a single string line by line
-- Next we can cast to seperate columns

SELECT 
   CAST(JSON_VALUE(jsonContent,'$.Id') AS INT) AS Id,
   JSON_VALUE(jsonContent,'$.Correlationid') As Correlationid,
   JSON_VALUE(jsonContent,'$.Operationname') AS Operationname,
   JSON_VALUE(jsonContent,'$.Status') AS Status,
   JSON_VALUE(jsonContent,'$.Eventcategory') AS Eventcategory,
   JSON_VALUE(jsonContent,'$.Level') AS Level,
   CAST(JSON_VALUE(jsonContent,'$.Time') AS datetimeoffset) AS Time,
   JSON_VALUE(jsonContent,'$.Subscription') AS Subscription,
   JSON_VALUE(jsonContent,'$.Eventinitiatedby') AS Eventinitiatedby,
   JSON_VALUE(jsonContent,'$.Resourcetype') AS Resourcetype,
   JSON_VALUE(jsonContent,'$.Resourcegroup') AS Resourcegroup
FROM
    OPENROWSET(
        BULK 'https://appdatalake7000.dfs.core.windows.net/data/log.json',
        FORMAT = 'CSV',
        FIELDQUOTE = '0x0b',
        FIELDTERMINATOR ='0x0b',
        ROWTERMINATOR = '0x0a'
    )
    WITH (
        jsonContent varchar(MAX)
    ) AS [rows]
```


##### 4.6 Azure Synapse Architecture

- there are 60 distributions
- data is shared into distributions to optimize the performance of work
- data and compute are seperate, they can scale independently
- **control node** - optimizes the query for parallel processing
- work is then passed to the **compute nodes**, these nodes will do the work in parallel

##### 4.7 Types of tables
- Round-robin distributed tables:
     - data is distributed randomly
     - default distribution while creating tables
     - best for temporary or staging tables
     - If there are no joins performed on tables, then you can consider using this table type
     - Also, if there is no clear candidate column for hash distributing the table.
- Hash-distributed tables:
     - data is distributed based on HASH(<particular column>)
     - good for large tables - fact tables
     - while choosing distribution column:
            - ensure it has many unique values - data gets spread across more distributions
            - if not, it may result in DATA SKEW
            - dont use date column
            - does not have NULLS or very few NULLS
            - is used in JOIN, GROUP BY and HAVING clauses
            - is not used in the WHERE clause
    -  ```sql
            CREATE TABLE [dbo].[SalesFact](
            [ProductID] [int] NOT NULL,
            [SalesOrderID] [int] NOT NULL,
            [CustomerID] [int] NOT NULL,
            [OrderQty] [smallint] NOT NULL,
            [UnitPrice] [money] NOT NULL,
            [OrderDate] [datetime] NULL,
            [TaxAmt] [money] NULL
            )
            WITH  
            (   
                DISTRIBUTION = HASH (CustomerID)
            )
        ```
- Replicated tables:
    - full copy of table is cached on every distribution (compute node)
    - good for dimension tables
    - ideal for tables less than 2 GB
    - not ideal for tables with frequent insert, update and delete
    - Use replicated tables for queries with simple query predicates, such as equality or inequality
    - Use distributed tables for queries with complex query predicates, such as LIKE or NOT LIKE
    - ```sql
        CREATE TABLE [dbo].[SalesFact](
        [ProductID] [int] NOT NULL,
        [SalesOrderID] [int] NOT NULL,
        [CustomerID] [int] NOT NULL,
        [OrderQty] [smallint] NOT NULL,
        [UnitPrice] [money] NOT NULL,
        [OrderDate] [datetime] NULL,
        [TaxAmt] [money] NULL
        )
        WITH  
        (   
            DISTRIBUTION = REPLICATE
        )
        ```

- If we are not using hash-distributed tables for fact tables & replicated tables for dimension tables, while performing JOINs or any other operations - data has to be moved from one distribution to the other distribution. this operation is called as "**DATA SHUFFLE MOVE OPERATION**" - this may lead to time lag for very big tables.

##### 4.8 Surrogate keys for dimension tables
- surrogate key == non-business key
- simple incrementing integer values
- in SQL pool tables, use IDENTITY column feature

```sql
CREATE TABLE [dbo].[DimProduct](
	[ProductSK] [int] IDENTITY(1,1) NOT NULL,
	[ProductID] [int] NOT NULL,
	[ProductModelID] [int] NOT NULL,
	[ProductSubcategoryID] [int] NOT NULL,
	[ProductName] varchar(50) NOT NULL,
	[SafetyStockLevel] [smallint] NOT NULL,
	[ProductModelName] varchar(50) NULL,
	[ProductSubCategoryName] varchar(50) NULL
)
```

- in synapse studio integrate data copy method -> the Identity column - not oncremented one by one - but by number of distributions
- ADF can properly create incremental nubers in IDENTIY column


##### 4.9 Slowly changing dimensions
- type-1 SCD: updates the OLD value with the NEW value in the data warehouse
- type-2 SCD: keeps both OLD and NEW values (start_date and end_date and is_active)
- type-3 SCD: instead of having multiple rows, additional columns are added to signify the change

##### 4.10 Heap tables
```sql
CREATE TABLE [dbo].[SalesFact_staging](
	[ProductID] [int] NOT NULL,
	[SalesOrderID] [int] NOT NULL,
	[CustomerID] [int] NOT NULL,
	[OrderQty] [smallint] NOT NULL,
	[UnitPrice] [money] NOT NULL,
	[OrderDate] [datetime] NULL,
	[TaxAmt] [money] NULL
)
WITH(HEAP,
DISTRIBUTION = ROUND_ROBIN
)

CREATE INDEX ProductIDIndex ON [dbo].[SalesFact_staging] (ProductID)
```
- this does not create a clustered column store table
- clustered column store table: used for final tables
- for temporary tables - HEAP tables are prefered
- In heap tables - no option to create clustered column store INDEX
- so, we can create a non-clustered INDEX using `CREATE INDEX`

##### 4.11 Partitions

```sql
-- Let's create a new table with partitions
CREATE TABLE [logdata]
(
    [Id] [int] NULL,
	[Correlationid] [varchar](200) NULL,
	[Operationname] [varchar](200) NULL,
	[Status] [varchar](100) NULL,
	[Eventcategory] [varchar](100) NULL,
	[Level] [varchar](100) NULL,
	[Time] [datetime] NULL,
	[Subscription] [varchar](200) NULL,
	[Eventinitiatedby] [varchar](1000) NULL,
	[Resourcetype] [varchar](1000) NULL,
	[Resourcegroup] [varchar](1000) NULL
)
WITH
(
PARTITION ( [Time] RANGE RIGHT FOR VALUES
            ('2021-04-01','2021-05-01','2021-06-01')

   )  
)
```

**Switching partitions**
```sql
ALTER TABLE [logdata] SWITCH PARTITION 2 TO [logdata_new] PARTITION 1;
```


##### 4.12 Indexes
- Clusterd Columnstore Indexes
- Heap tables
- Clustered Indexes
- NonClustered Indexes


### 5. Design and Develop Data Processing - Azure Data Factory

##### 5.1 Azure Data Factory
- cloud-based ET tool
- data-driven orchestrated workflows

**ADF components:**
- **Linked Service:** can create required compute resources to enable ingestion of data from the source 
- **Datasets:** represents the data structure within the data store that is being referenced by the Linked Service object
- **Activity:** contains the actual transformation logic

- azure pipeline will create compute infrastructure known as **Integration runtime** - responsible for taking data from source and copying it to the destination


##### 5.2 Mapping Data Flows
- This helps to visualize the data transformations in Azure Data Factory.
- Here you can write the required transformation logic without actually writing any code.
- The data flows are run on **Apache Spark clusters**.
- Here Azure Data Factory will handle the transformations in the data flow.
- **Debug mode** – You also have a Debug mode in place. Here you can actually see the results of each transformation.
- In the debug mode session, the data flow is run interactively on a Spark cluster.
- minimum cluster size to run a Data Flow is 8 vCores.


##### 5.3 Self-Hosted Integration runtime
- if the database in own custom system sitting inside a VM
- install the integration runtime on VM
- register the server with the data factory


### 6. Azure Event Hubs and Streaming Analytics

##### 6.1 Azure Event Hubs

- big data streaming platform
- can receive and process millions of events per second
- can stream log data, telemetry data, any sort of events to azure events hub
- event hubs namespace -> event hubs
-  event hubs - multiple partitions - ingest more data at a time - event receivers can take data from one partition or multiple partitions - helps event receivers to consume data in faster rate

**Components of Azure event hubs:**
- event producers: entity that sends data to event hub - events can be published using the protocols - HTTPS, AMQP, Apache Kafka
- partitions: data is split across partitions - allows for better throughput of data onto event hubs
- consumer groups: view (state, position or offset) of an entire event hub
- throughput: controls the throughput capacity of event hubs
- receivers: entity taht reads event data



### 7. Spark Pool

##### 7.1 Azure Synapse - Apache Spark pool
- serverless spark pool
- not charged on creation of pool
- charged when underlying jobs are running
- large datasets and distribute computation across multiple pools
- driver node and executors

- spark scala
- creates RDD - Resilient Distributed Dataset
```scala
val dist = sc.parallelize(data)
```

##### 7.2 Spark Dataset
- This is a strongly typed collection of domain-specific objects
- This data can then be transformed in parallel
- Normally you will perform either transformations or actions on a dataset
- The transformation will produce a new dataset
- The action will trigger a computation and produce the required result
- The benefit of having a Dataset is that you can use powerful transformations on the underlying data

##### 7.3 Spark Dataframe
-  The DataFrame is nothing but a Dataset that is organized into named columns.
- Its like a table in a relational database.
- You can construct DataFrames from external files.
- When it comes to Datasets, the API for working with Datasets is only available for Scala and Java.
- For DataFrames, the API is available in Scala, Java, Python and R.


- In the spark pool, the spark instances are created when you connect to a spark pool, create a session and run a job
- when you submit another job, if there is capacity in the pool and the spark instance has spare capacity, itwill run the 2nd job
- else, it will crate a new spark instance to run the job

##### 7.4 Spark table
- stored in metastore of spark pool (HIVE META STORE)
- not for storing data, just for temporary tables
- the benefit of spark table: metastore is shared with serverless SQL pool as well

```scala
%%spark
val df = spark.read.sqlanalytics("jibsypool.dbo.logdata") 
df.write.mode("overwrite").saveAsTable("logdatainternal")

%%sql
SELECT * FROM logdatainternal
```

##### 7.5 Spark tables - Creation
- spark tables are parquet based tables
- 
```scala
%%sql
CREATE DATABASE internaldb
CREATE TABLE internaldb.customer(Id int,name varchar(200)) USING Parquet

%%sql
INSERT INTO internaldb.customer VALUES(1,'UserA')

%%sql
SELECT * FROM internaldb.customer


// If you want to load data from the log.csv file and then save to a table
%%pyspark
df = spark.read.load('abfss://data@datalake2000.dfs.core.windows.net/raw/Log.csv', format='csv', header=True)
df.write.mode("overwrite").saveAsTable("internaldb.logdatanew")

%%sql
SELECT * FROM internaldb.logdatanew
```

- to delete the database, tables have to be dropped first

##### 7.6 Spark Pool - JSON files

```scala
%%spark

val df = spark.read.format("json").load("abfss://data@datalake2000.dfs.core.windows.net/raw/customer/customer_arr.json")
display(df)

// Now we need to expand the courses information

%%spark
import org.apache.spark.sql.functions._
val df = spark.read.format("json").load("abfss://data@datalake2000.dfs.core.windows.net/raw/customer/customer_arr.json")
val newdf=df.select(col("customerid"),col("customername"),col("registered"),explode(col("courses")))
display(newdf)

// Reading the customer object file
%%spark
import org.apache.spark.sql.functions._
val df = spark.read.format("json").load("abfss://data@datalake2000.dfs.core.windows.net/raw/customer/customer_obj.json")
val newdf=df.select(col("customerid"),col("customername"),col("registered"),explode(col("courses")),col("details.city"),col("details.mobile"))
display(newdf)
```

### 8. Databricks

##### 8.1 Databricks
- makes use of apache spark to provide a unified analytics platform
- creates the underlying compute infra
- has its own underlying file system - abstraction of an underlying storage layer
- will install spark by itself - also has comatibility for other libs - ML libs
- provides workspace - notebooks with collaboration and visualization features


##### 8.2 Azure Databricks
- completely azure-managed environment
- makes use of underlying compute infrastructure and virtual networks
- makes use of azure security - azure active directory and role-based access control

**Clusters in Azure Databricks**
- inside cluster - 2 types of nodes
    - worker nodes - perform the underlying tasks
    - driver node - distributes the task to worker nodes

- 2 types of clusters
    - Interactive clustrer: interactive notebooks and multiple users can use a cluster for collaboration
    - Job cluster: cluster is started when the job has to run, and will be terminated once the job is completed

- 2 types of Interactive cluster
    - Standard cluster:
        - recommended if you are a single user
        - no fault isolation - if multiple users are using and one user has fault - this might impact workloads of other users
        - resources of a cluster might get allocated to a single workload
        - has support for python, R, SQL and Scala
    - High concurrency cluster:
        - for multiple users
        - fault isolation
        - resources are shared across different user workloads
        - support for python, R and SQL (no scala)
        - table access control: can grant and revke access to data from python and SQL

##### 8.3 Autoscaling a cluster

- When creating an Azure Databricks cluster, you can specify a minimum and  maximum number of workers for the cluster.
- Databricks will then choose the ideal number of workers to run the job.
- If a certain phase of your job requires more compute power, the workers will be assigned accordingly.
- There are two types of autoscaling
    - **Standard autoscaling**
        - Here the cluster starts with 8 nodes
        - Scales down only when the cluster is completely idle and it has been underutilized for the last 10 minute
        - Scales down exponentially, starting with 1 node
    - **Optimized autoscaling**
        - This is only available for Azure Databricks Premium Plan
        - Can scale down even if the cluster is not idle by looking at shuffle file state
        - Scales down based on a percentage of current nodes
        - On job clusters, scales down if the cluster is underutilized over the last 40 seconds
        - On all-purpose clusters, scales down if the cluster is underutilized over the last 150 seconds

##### 8.4 Azure Databricks Table
- In Azure Databricks, you can also create a database and tables
- The table is a collection of structured data
- You can then perform operations on the data that are supported by Apache Spark on DataFrames on Azure Databricks tables
- There are two types of tables – global and local tables.
- A global table is available across all clusters
- A global table is registered in the Azure Databricks Hive metastore or an external metastore
- The local table is not accessible from other clusters and is not registered in the Hive metastore


##### 8.5 Delta Lake

- ACID transactions on Spark - Serializable isolation levels ensure that readers never see inconsistent data
- Scalable metadata handling - Leverages Spark distributed processing power to handle all the metadata for petabyte-scale tables with billions of files at ease.
- Streaming and batch unification - A table in Delta Lake is a batch table as well as a streaming source and sink. Streaming data ingest, batch historic backfill, interactive queries all just work out of the box.
- Schema enforcement - Automatically handles schema variations to prevent insertion of bad records during ingestion.
- Time travel - Data versioning enables rollbacks, full historical audit trails, and reproducible machine learning experiments.
- Upserts and deletes - Supports merge, update and delete operations to enable complex use cases like change-data-capture, slowly-changing-dimension (SCD) operations, streaming upserts, and so on.

### 9 Security

- **Azure Key Vault** - certificates, encryption keys and secrets (passwords and login details)

- **Azure Data Factory – Encryption**
    - Azure Data Factory already encrypts data at rest which also includes entity definitions and any data that is cached. 
    - The encryption is carried out with Microsoft-managed keys.
    - But you can also define your own keys using the Azure Key vault service.
    - For the key vault, you have to ensure that **Soft delete** is enabled and the setting of **Do Not Purge** is also enabled.
    - Also grant Azure Data Factory the key permissions of 'Get', 'Unwrap Key' and 'Wrap Key'

- **Azure Synapse - Data Masking**
    - Here the data in the table can be limited in its exposure to non-privileged users.
    - You can create a rule that can mask the data. 
    - Based on the rule you can decide on the amount of data to expose to the user.
    - There are different masking rules.
    - Credit Card masking rule – This is used to mask the column that contain credit card details. Here only the last four digits of the field are exposed.
    - Email – Here first letter of the email address is exposed. And the domain name of the email address is replaced with XXX.com.
    - Custom text- Here you decide which characters to expose for a field.
    - Random number- Here you can generate a random number for the field.

- **Azure Synapse - Auditing** 
    - You can enable auditing for an Azure SQL Pool in Azure Synapse Analytics.
    - This feature can be used to track database events and write them to an audit log. 
    - The logs can be stored in an Azure storage account, a Log Analytics workspace and Azure Event Hubs.
    - This helps in regulatory compliance. It helps to gain insights on any anomalies when it comes to database activities.
    - Auditing can be enabled at the data warehouse level or server level.
    - If it is applied at the server level, then it will be applied to all of the data warehouses that reside on the server

- **Azure Synapse - Data Discovery and Classification**
    - This feature provides capabilities for discovering, classifying, labelling, and reporting the sensitive data in your databases.
    - The data discovery feature can scan the database and identify columns that contains sensitive data. You can then view and apply the recommendations accordingly.
    - You can then apply sensitivity labels to the column. This helps to define the sensitivity level of the data stored in the column.

- **Row level security:**

```sql
-- Create a new schema for the security function

CREATE SCHEMA Security;  

-- Create an inline table function
-- The function returns 1 when a row in the Agentcolumn is the same as the user executing the query 
-- (@Agent = USER_NAME()) or if the user executing the query is the Manager user (USER_NAME() = 'Supervisor').

CREATE FUNCTION Security.securitypredicate(@Agent AS nvarchar(50))  
    RETURNS TABLE  
WITH SCHEMABINDING  
AS  
    RETURN SELECT 1 AS securitypredicate_result
WHERE @Agent = USER_NAME() OR USER_NAME() = 'Supervisor';  

-- Create a security policy adding the function as a filter predicate. The state must be set to ON to enable the policy.

CREATE SECURITY POLICY Filter  
ADD FILTER PREDICATE Security.securitypredicate(Agent)
ON [dbo].[Orders] 
WITH (STATE = ON);  
GO

-- Lab - Azure Synapse - Row-Level Security

-- Allow SELECT permissions to the function

GRANT SELECT ON Security.securitypredicate TO Supervisor;
GRANT SELECT ON Security.securitypredicate TO AgentA;  
GRANT SELECT ON Security.securitypredicate TO AgentB;
```


- **Azure Synapse - Column level security**

```sql
CREATE USER Supervisor WITHOUT LOGIN;  
CREATE USER UserA WITHOUT LOGIN;  

-- Grant access to the tables for the users

GRANT SELECT ON [dbo].[Orders] TO Supervisor; 
GRANT SELECT ON [dbo].[Orders](OrderID,Course,Quantity) TO UserA; 

```
