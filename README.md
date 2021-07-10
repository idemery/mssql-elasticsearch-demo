# Elasticsearch demo with Microsoft SQL Server and Logstash syncing

## Intro

This will do the following:
- Create instances of mssql, elasticsearch, logstash, and kibana
- Download WorldWideImporters sample database backup and restore it
- Create an index in elasticsearch with the Invoices table using logstash jdbc input with a scheduler to sync it based on the LastUpdatedWhen field

## How to use it
```console
git clone https://github.com/idemery/mssql-elasticsearch-demo
```

Download the mssql jdbc driver from https://docs.microsoft.com/en-us/sql/connect/jdbc/microsoft-jdbc-driver-for-sql-server  
Copy the jdbc jar file mssql-jdbc-9.2.1.jre8.jar to mssql-elasticsearch-demo/logstash/ 
```console
cd mssql-elasticsearch-demo
docker-compose build
docker-compose up
```

That's it!  
Navigate to http://localhost:9200/idx_wwi_invoices/_search?pretty=true&q=*:*&size=100 to make sure index is created and loaded with data  

Start using Kibana on http://localhost:5601/ and create a new index pattern from Analytics > Discover using index name *idx_wwi_invoices*



Password used for the demo is **qweQWE123** 

## Quick eye on the content

```bash
.
├── docker-compose.yml
├── logstash
│   ├── config
│   │   └── logstash.config
│   ├── Dockerfile
│   └── mssql-jdbc-9.2.1.jre8.jar
└── mssql
    └── Dockerfile
```
##### mssql/Dockerfile
```
FROM mcr.microsoft.com/mssql/server:2019-GA-ubuntu-16.04
ENV SA_PASSWORD "qweQWE123"
ENV ACCEPT_EULA "Y"

USER root 

ADD --chown=mssql:root https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Full.bak /var/opt/mssql/backup/wwi.bak

USER mssql

RUN /opt/mssql/bin/sqlservr --accept-eula & (echo "awaiting mssql bootup for 15 seconds" && sleep 15 && echo "restoring.."  && /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'qweQWE123' -Q 'RESTORE DATABASE WideWorldImporters FROM DISK = "/var/opt/mssql/backup/wwi.bak" WITH MOVE "WWI_Primary" TO "/var/opt/mssql/data/WideWorldImporters.mdf", MOVE "WWI_UserData" TO "/var/opt/mssql/data/WideWorldImporters_userdata.ndf", MOVE "WWI_Log" TO "/var/opt/mssql/data/WideWorldImporters.ldf", MOVE "WWI_InMemory_Data_1" TO "/var/opt/mssql/data/WideWorldImporters_InMemory_Data_1"')

```

##### logstash/Dockerfile
```
FROM docker.elastic.co/logstash/logstash:7.13.2

RUN rm -f /usr/share/logstash/pipeline/logstash.conf

USER root 

COPY mssql-jdbc-9.2.1.jre8.jar /usr/share/logstash/logstash-core/lib/jars/mssql-jdbc-9.2.1.jre8.jar
```

##### logstash/config/logstash.config
```
input {
  jdbc {
    jdbc_driver_class => "com.microsoft.sqlserver.jdbc.SQLServerDriver"
    jdbc_connection_string => "jdbc:sqlserver://sql1:1433;databaseName=WideWorldImporters;user=sa;password=qweQWE123"
    jdbc_user => "sa"
    jdbc_password => "qweQWE123"
    jdbc_paging_enabled => true
    tracking_column => "unix_ts_in_secs"
    use_column_value => true
    tracking_column_type => "numeric"
    schedule => "*/59 * * * * *"
    statement => "SELECT [InvoiceID]
      ,[CustomerID]
      ,[BillToCustomerID]
      ,[OrderID]
      ,[DeliveryMethodID]
      ,[ContactPersonID]
      ,[AccountsPersonID]
      ,[SalespersonPersonID]
      ,[PackedByPersonID]
      ,[InvoiceDate]
      ,[CustomerPurchaseOrderNumber]
      ,[IsCreditNote]
      ,[CreditNoteReason]
      ,[Comments]
      ,[DeliveryInstructions]
      ,[InternalComments]
      ,[TotalDryItems]
      ,[TotalChillerItems]
      ,[DeliveryRun]
      ,[RunPosition]
      ,[ReturnedDeliveryData]
      ,[ConfirmedDeliveryTime]
      ,[ConfirmedReceivedBy]
      ,[LastEditedBy]
      ,[LastEditedWhen], DATEDIFF_BIG(ms, '1970-01-01 00:00:00', LastEditedWhen) AS unix_ts_in_secs FROM [WideWorldImporters].[Sales].[Invoices] WHERE (DATEDIFF_BIG(ms, '1970-01-01 00:00:00', LastEditedWhen) > :sql_last_value AND LastEditedWhen < getdate()) "
  }
}
filter {
  mutate {
    copy => { "invoiceid" => "[@metadata][_id]"}
    remove_field => ["invoiceid", "@version", "unix_ts_in_secs"]
  }
}
output {
  # stdout { codec =>  "rubydebug"}
  elasticsearch {
    hosts => [ "elasticsearch:9200"]
    index => "idx_wwi_invoices"
    document_id => "%{[@metadata][_id]}"
  }
}
```

##### docker-compose.yml
```yaml
version: '2.2'
services:
  mssql:
    build: ./mssql
    container_name: sql1
    restart: always
    ports:
      - 1433:1433
    networks:
      - elastic
    volumes:
      - mssql:/var/opt/mssql
  elasticsearch:
    image: elasticsearch:7.13.2
    container_name: elasticsearch
    environment:
            - cluster.name=docker-cluster
            - bootstrap.memory_lock=true
            - discovery.type=single-node
            - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data:/usr/share/elasticsearch/data
    restart: always
    depends_on:
      - mssql
    ports:
      - 9200:9200
    networks:
      - elastic
  logstash:
    build: ./logstash
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    ports:
      - 5001:5001
    container_name: logstash
    restart: always
    networks:
      - elastic
    depends_on:
      - elasticsearch
    volumes:
      - ./logstash/config:/usr/share/logstash/pipeline
  kibana:
    image: docker.elastic.co/kibana/kibana:7.13.2
    environment:
      SERVER_HOST: 0.0.0.0
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    container_name: kibana
    depends_on:
      - elasticsearch
    ports:
      - 5601:5601
    networks:
      - elastic

volumes:
  data:
    driver: local
  mssql:
    driver: local

networks:
  elastic:
    driver: bridge
```
