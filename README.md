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

