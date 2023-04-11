## 2. Azure Synapse Analytics

### 2.1 Azure Synapse

-   recursos do Azure Synapse Analytics
-   opções de computação:
    -   pool SQL sem servidor
    -   pool SQL dedicado
        -   DWU - unidade de processamento de data warehousing
    -   pool Apache Spark

### 2.2 Tabelas externas e Pool SQL sem servidor / Pool SQL dedicado
#### passos para criar e usar uma tabela externa
- criar um banco de dados no espaço de trabalho do Synapse
- criar uma chave mestra do banco de dados com criptografia por senha. Isso será usado para proteger a Assinatura de Acesso Compartilhado (SAS)
- usar o SAS para autorizar o uso da conta ADLS, criar credencial de escopo de banco de dados SasToken
- criar um recurso de dados externos (pode ser Hadoop, armazenamento de blob ou ADLS)
- criar um objeto de formato de arquivo externo que define os dados externos (formato de arquivo = DELIMITEDTEXT ou PARQUET)
- definir a tabela externa
- usar a tabela para análise (haverá uma defasagem, pois os dados são armazenados na fonte externa)

 ``` sql
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

CREATE EXTERNAL TABLE [logdata] (
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

### 2.3 Carregando dados no data warehouse (pool SQL)

-   usando a instrução T-SQL COPY
-   usando o pipeline do Azure Synapse, é possível realizar transformações nos dados antes de copiá-los para o data warehouse
-   usando o Polybase para definir tabelas externas, use as tabelas externas para criar as tabelas internas
-   **1 - carregando dados usando a instrução COPY**
    -   nunca use a conta de administrador para operações de carga
    -   crie um usuário separado para operações de carga
    -   melhor prática - crie um grupo de carga de trabalho - para segregar a porcentagem de CPU entre grupos de usuários
    -   conceda permissões
    -   csv:
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
          CREDENTIAL=
            (
              IDENTITY= 'Shared Access Signature', 
              SECRET='sv=2021-06-08&ss=b&srt=sco&sp=rl&se=2022-12-22T14:08:01Z&st=2022-12-22T06:08:01Z&spr=https&sig=WU%2FFh62PcCSx7wSEuccKC%2FdlgAwIto2aHJVXMiPovfM%3D')
          )
        ```

    - Polybase: 
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

    - BULK Insert: 
		1.  Abra o Azure Data Studio.
		2.  Conecte-se a um data warehouse usando suas credenciais.
		3.  Clique em "Connect to external data" para conectar-se a uma fonte de dados externa.
		4.  Selecione "ADLS Gen2" na lista de serviços conectados.
		5.  Crie uma conexão com o armazenamento de dados externo preenchendo os detalhes necessários.
		6.  Antes de copiar dados, dê permissões específicas à conta de armazenamento de dados.
		7.  Acesse o controle de acesso na conta de armazenamento de dados.
		8.  Adicione uma atribuição de função, selecionando "Storage Blob Data Contributor" para permitir a leitura e gravação de dados.
		9.  Escolha a conta de administração do Azure e salve as alterações.
		10.  Volte ao Azure Data Studio, vá em "Linked" e selecione a conexão do ADLS conectado.
		11.  Selecione o arquivo que deseja carregar.
		12.  Clique com o botão direito do mouse no arquivo e selecione "Bulk Load".
		13.  Isso criará automaticamente o script SQL necessário para o processo de carga em massa.

### 2.4 Projetando um data warehouse

-   tabela de fatos
    -   contém fatos mensuráveis
    -   geralmente grande em tamanho
-   tabela de dimensões
-   nota: não há conceito de chave estrangeira no data warehouse SQL no pool dedicado SQL no Azure Synapse
-   esquema estrela
-   prática ideal ao construir tabelas de dimensão:
    -   não ter valores NULL para propriedades na tabela de dimensão, não dará resultados desejados ao usar ferramentas de relatórios
    -   tente substituir NULL por algum valor padrão
-   chave substituta: nova chave adicionada na tabela de dimensão ao misturar duas fontes de dados diferentes com as mesmas chaves primárias
-   pode usar o recurso "Coluna de identidade" no Azure Synapse para gerar um ID exclusivo
-   abordagem correta -> pegar tabelas diferentes e criar tabela de fatos no próprio Synapse usando o ADF para migrar tabelas do banco de dados do Azure para o Azure Synapse

### 2.5 Transferir dados do banco de dados SQL do Azure para o Azure Synapse

-   criar estrutura da tabela no Synapse
-   abrir o Synapse Studio -> Integrate -> Copy data tool
-   fazer conexão com a fonte (banco de dados SQL do Azure) - selecionar o servidor, banco de dados e detalhes da tabela
-   fazer conexão com o destino (Azure Synapse) - selecionar os detalhes necessários
-   selecionar a área de preparação no ADLS2 ou blob - usada pela instrução de cópia

### 2.6 Lendo arquivos JSON do ADLS/Blob

Aqui estamos usando a função OPENROWSET
```sql
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

-- A instrução acima retorna tudo como uma única string, linha por linha
-- Em seguida, podemos converter em colunas separadas

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

### 2.7 Arquitetura do Azure Synapse

-   existem 60 distribuições
-   os dados são compartilhados em distribuições para otimizar o desempenho do trabalho
-   dados e computação são separados, podendo ser dimensionados independentemente
-   **nó de controle** - otimiza a consulta para processamento paralelo
-   o trabalho é então passado para os **nós de computação**, que farão o trabalho em paralelo

##### 2.8 Tipos de tabelas

-   Tabelas distribuídas round-robin:
    -   os dados são distribuídos aleatoriamente
    -   distribuição padrão ao criar tabelas
    -   melhor para tabelas temporárias ou de encenação
    -   se não houver junções realizadas nas tabelas, você pode considerar usar este tipo de tabela
    -   também se não houver uma coluna clara candidata para distribuição por hash da tabela.
-   Tabelas distribuídas por hash:
    -   os dados são distribuídos com base em HASH (<coluna específica>)
    -   bom para tabelas grandes - tabelas de fatos
    -   ao escolher a coluna de distribuição: - certifique-se de que ela tenha muitos valores únicos - os dados são espalhados por mais distribuições - caso contrário, pode resultar em DATA SKEW - não use coluna de data - não tem NULLS ou muito poucos NULLS - é usada nas cláusulas JOIN, GROUP BY e HAVING - não é usada na cláusula WHERE
		```sql
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

- Tabelas replicadas:

	-   cópia completa da tabela é armazenada em cache em cada distribuição (nó de computação)
	-   bom para tabelas de dimensão
	-   ideal para tabelas com menos de 2 GB
	-   não é ideal para tabelas com inserções, atualizações e exclusões frequentes
	-   Use tabelas replicadas para consultas com predicados de consulta simples, como igualdade ou desigualdade
	-   Use tabelas distribuídas para consultas com predicados de consulta complexos, como LIKE ou NOT LIKE
		```sql
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

-   Se não estivermos usando tabelas distribuídas por hash para tabelas de fatos e tabelas replicadas para tabelas de dimensão, ao realizar JOINs ou outras operações, os dados precisam ser movidos de uma distribuição para outra distribuição. Essa operação é chamada de "**OPERAÇÃO DE MOVIMENTAÇÃO DE DADOS SHUFFLE**" - isso pode levar a um atraso de tempo para tabelas muito grandes.

### 2.9 Chaves substitutas para tabelas de dimensão

-   chave substituta == chave não empresarial
-   valores inteiros incrementais simples
-   em tabelas do SQL pool, use a funcionalidade de coluna IDENTITY

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
-   no Synapse Studio, integre o método de cópia de dados -> a coluna Identity - não incrementada um por um - mas pelo número de distribuições
-   O ADF pode criar adequadamente números incrementais na coluna IDENTITY.

### 2.10 Dimensões de alteração lenta

-   SCD tipo 1: atualiza o valor ANTIGO com o valor NOVO no data warehouse.
-   SCD tipo 2: mantém ambos os valores ANTIGO e NOVO (data de início, data de fim e está ativo).
-   SCD tipo 3: em vez de ter várias linhas, colunas adicionais são adicionadas para sinalizar a mudança.

### 2.11 Tabelas Heap

-   Isso não cria uma tabela de armazenamento em colunas clusterizada.
-   Tabela de armazenamento em colunas clusterizada: usada para tabelas finais.
-   Para tabelas temporárias, as tabelas HEAP são preferidas.
-   Nas tabelas HEAP, não há opção para criar um ÍNDICE de armazenamento em colunas clusterizado.
-   Portanto, podemos criar um ÍNDICE não clusterizado usando `CREATE INDEX`.



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

	CREATE INDEX ProductIDIndex ON [dbo].	[SalesFact_staging] (ProductID)
	```

### 2.12 Partições

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

**Alteando Partições**
```sql
ALTER TABLE [logdata] SWITCH PARTITION 2 TO [logdata_new] PARTITION 1;
```

### 2.13 Índices

-   Índices de armazenamento em colunas clusterizado
-   Tabelas Heap
-   Índices clusterizados
-   Índices não clusterizados
