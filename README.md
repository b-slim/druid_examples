# Node status
via Curl
```sh
curl localhost:8082/status
{"version":"0.10.1.2.6.3.0-220","modules":[{"name":"io.druid.query.aggregation.datasketches.theta.SketchModule","artifact":"druid-datasketches","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.query.aggregation.datasketches.theta.oldapi.OldApiSketchModule","artifact":"druid-datasketches","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.storage.hdfs.HdfsStorageDruidModule","artifact":"druid-hdfs-storage","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.indexing.kafka.KafkaIndexTaskModule","artifact":"druid-kafka-indexing-service","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.metadata.storage.mysql.MySQLMetadataStorageModule","artifact":"mysql-metadata-storage","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.emitter.ambari.metrics.AmbariMetricsEmitterModule","artifact":"ambari-metrics-emitter","version":"0.10.1.2.6.3.0-220"}],"memory":{"maxMemory":2058354688,"totalMemory":2058354688,"freeMemory":1727201736,"usedMemory":331152952}}

```

# Setup data

sudo su druid
wget https://transfer.sh/sWgBK/wikiticker-2015-09-12-sampled.json.gz
gunzip wikiticker-2015-09-12-sampled.json.gz
hdfs dfs -ls /apps/druid
hdfs dfs -put wikiticker-2015-09-12-sampled.json /apps/druid/warehouse

# Index data via curl

wget https://transfer.sh/5W3Nh/index_task_hdfs.json

Explain the task spec:

- type of task
- data source name
- path to the data
- specs of partition


curl -X 'POST' -H 'Content-Type:application/json' -d@index-task.json ctr-e134-1499953498516-247377-01-000004.hwx.site:8090/druid/indexer/v1/task

Look at the orveloard UI look at the indexing log

Look at the coordinator UI look at the load status

# Query the data 
http://druid.io/docs/0.10.1/querying/querying.html

## Time Boundary
wget https://transfer.sh/wzxR2/timeBoundary.json
curl -X POST "ctr-e134-1499953498516-247377-01-000002.hwx.site:8888/druid/v2/?pretty" -H 'content-type: application/json' -d@timeBoundary.json

## TopN
https://transfer.sh/Ta26s/topN.json
curl -X POST "ctr-e134-1499953498516-247377-01-000002.hwx.site:8888/druid/v2/?pretty" -H 'content-type: application/json' -d@topN.json

## Practice 
write/run your own query using druid group by query help page 
http://druid.io/docs/0.10.1/querying/groupbyquery.html

# SuperSet UI
Via Ambari UI Go to Superset and create some simple slice

# Hive QL

```sql
CREATE EXTERNAL TABLE wikipedia
STORED BY 'org.apache.hadoop.hive.druid.DruidStorageHandler'
TBLPROPERTIES ("druid.datasource" = "wikipedia");
```

##Query the existing data

```sql
DESCRIBE FORMATTED  wikipedia;
SELECT * FROM wikipedia LIMIT 10;
SELECT COUNT(*) FROM wikipedia;
SELECT COUNT(*), floor_hour(`__time`) FROM wikipedia GROUP BY floor_hour(`__time`);
```


## Index more data 
```sql
drop table login_hive;
create table login_hive(`timecolumn` timestamp, `userid` string, `num_l` double);
insert into login_hive values ('2015-01-01 00:00:00', 'user1', 5);
insert into login_hive values ('2015-01-01 01:00:00', 'user2', 4);
insert into login_hive values ('2015-01-01 02:00:00', 'user3', 2);

insert into login_hive values ('2015-01-02 00:00:00', 'user1', 1);
insert into login_hive values ('2015-01-02 01:00:00', 'user2', 2);
insert into login_hive values ('2015-01-02 02:00:00', 'user3', 8);

insert into login_hive values ('2015-01-03 00:00:00', 'user1', 5);
insert into login_hive values ('2015-01-03 01:00:00', 'user2', 9);
insert into login_hive values ('2015-01-03 04:00:00', 'user3', 2);

insert into login_hive values ('2015-03-09 00:00:00', 'user3', 5);
insert into login_hive values ('2015-03-09 01:00:00', 'user1', 0);
insert into login_hive values ('2015-03-09 05:00:00', 'user2', 0);


drop table login_druid;
CREATE TABLE login_druid
STORED BY 'org.apache.hadoop.hive.druid.DruidStorageHandler'
TBLPROPERTIES ("druid.segment.granularity" = "DAY", "druid.query.granularity" = "HOUR")
AS
select `timecolumn` as `__time`, `userid`, `num_l` FROM login_hive;

```

## Query the data 
``` sql
select * FROM login_druid;
```


# Delete the data source 
```sql
drop table login_druid;
```

# Msc commands
to kill yarn app
yarn application -kill  application_1508802793765_0005


# Personal notes setup from my local laptop
cd /Users/sbouguerra/Ydev/druid/examples/quickstart
transfer wikiticker-2015-09-12-sampled.json.gz

cd /Users/sbouguerra/Ydev/druid/qtl_tests
transfer index_task_hdfs.json
