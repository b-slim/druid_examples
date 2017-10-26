# Node status
via Curl
```sh
curl localhost:8082/status

{"version":"0.10.1.2.6.3.0-220","modules":[{"name":"io.druid.query.aggregation.datasketches.theta.SketchModule","artifact":"druid-datasketches","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.query.aggregation.datasketches.theta.oldapi.OldApiSketchModule","artifact":"druid-datasketches","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.storage.hdfs.HdfsStorageDruidModule","artifact":"druid-hdfs-storage","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.indexing.kafka.KafkaIndexTaskModule","artifact":"druid-kafka-indexing-service","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.metadata.storage.mysql.MySQLMetadataStorageModule","artifact":"mysql-metadata-storage","version":"0.10.1.2.6.3.0-220"},{"name":"io.druid.emitter.ambari.metrics.AmbariMetricsEmitterModule","artifact":"ambari-metrics-emitter","version":"0.10.1.2.6.3.0-220"}],"memory":{"maxMemory":2058354688,"totalMemory":2058354688,"freeMemory":1727201736,"usedMemory":331152952}}

```

# Setup data

```sh
sudo su druid

wget https://transfer.sh/sWgBK/wikiticker-2015-09-12-sampled.json.gz

gunzip wikiticker-2015-09-12-sampled.json.gz

hdfs dfs -ls /apps/druid

hdfs dfs -put wikiticker-2015-09-12-sampled.json /apps/druid/warehouse
```

# Index data via curl

wget https://transfer.sh/5W3Nh/index_task_hdfs.json

Explain the task spec:

- type of task
- data source name
- path to the data
- specs of partition

```sh
curl -X 'POST' -H 'Content-Type:application/json' -d@index-task.json ctr-e134-1499953498516-247377-01-000004.hwx.site:8090/druid/indexer/v1/task
```
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

Druid data warehouse is set by the druid property `druid.storage.storageDirectory=/apps/druid/warehouse`

```sh
hdfs dfs -ls /apps/druid/warehouse/wikipedia
```
	

# Troubleshooting
## Max Direct Memory Size
AKA *Please adjust -XX:MaxDirectMemorySize*
This is set by ambari as `druid.[NODE_TYPE].jvm.direct.memory`, the unit is Mega bytes.


```java
17) Not enough direct memory.  Please adjust -XX:MaxDirectMemorySize, druid.processing.buffer.sizeBytes, druid.processing.numThreads, or druid.processing.numMergeBuffers: maxDirectM
emory[109,051,904], memoryNeeded[2,147,483,648] = druid.processing.buffer.sizeBytes[536,870,912] * (druid.processing.numMergeBuffers[2] + druid.processing.numThreads[1] + 1)
  at io.druid.guice.DruidProcessingModule.getMergeBufferPool(DruidProcessingModule.java:124) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Mo
dules$OverrideModule -> io.druid.guice.DruidProcessingModule)
  at io.druid.guice.DruidProcessingModule.getMergeBufferPool(DruidProcessingModule.java:124) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Mo
dules$OverrideModule -> io.druid.guice.DruidProcessingModule)
  while locating io.druid.collections.BlockingPool<java.nio.ByteBuffer> annotated with @io.druid.guice.annotations.Merging()
    for the 4th parameter of io.druid.query.groupby.strategy.GroupByStrategyV2.<init>(GroupByStrategyV2.java:97)
  while locating io.druid.query.groupby.strategy.GroupByStrategyV2
    for the 3rd parameter of io.druid.query.groupby.strategy.GroupByStrategySelector.<init>(GroupByStrategySelector.java:43)
  while locating io.druid.query.groupby.strategy.GroupByStrategySelector
    for the 1st parameter of io.druid.query.groupby.GroupByQueryQueryToolChest.<init>(GroupByQueryQueryToolChest.java:104)
  at io.druid.guice.QueryToolChestModule.configure(QueryToolChestModule.java:95) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$Overri
deModule -> io.druid.guice.QueryRunnerFactoryModule)
  while locating io.druid.query.groupby.GroupByQueryQueryToolChest
  while locating io.druid.query.QueryToolChest annotated with @com.google.inject.multibindings.Element(setName=,uniqueId=64, type=MAPBINDER, keyType=java.lang.Class<? extends io.dru
id.query.Query>)
  at io.druid.guice.DruidBinders.queryToolChestBinder(DruidBinders.java:45) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideMod
ule -> io.druid.guice.QueryRunnerFactoryModule -> com.google.inject.multibindings.MapBinder$RealMapBinder)
  while locating java.util.Map<java.lang.Class<? extends io.druid.query.Query>, io.druid.query.QueryToolChest>
    for the 1st parameter of io.druid.query.MapQueryToolChestWarehouse.<init>(MapQueryToolChestWarehouse.java:36)
  while locating io.druid.query.MapQueryToolChestWarehouse
  while locating io.druid.query.QueryToolChestWarehouse
    for the 1st parameter of io.druid.client.CachingClusteredClient.<init>(CachingClusteredClient.java:117)
  at io.druid.cli.CliBroker$1.configure(CliBroker.java:94) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideModule -> io.druid.c
li.CliBroker$1)
  while locating io.druid.client.CachingClusteredClient
    for the 2nd parameter of io.druid.server.ClientQuerySegmentWalker.<init>(ClientQuerySegmentWalker.java:57)
  while locating io.druid.server.ClientQuerySegmentWalker
  at io.druid.cli.CliBroker$1.configure(CliBroker.java:107) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideModule -> io.druid.
cli.CliBroker$1)
  while locating io.druid.query.QuerySegmentWalker
    for the 2nd parameter of io.druid.server.QueryLifecycleFactory.<init>(QueryLifecycleFactory.java:53)
  at io.druid.server.QueryLifecycleFactory.class(QueryLifecycleFactory.java:53)
  while locating io.druid.server.QueryLifecycleFactory
    for the 1st parameter of io.druid.server.BrokerQueryResource.<init>(BrokerQueryResource.java:64)
  at io.druid.cli.CliBroker$1.configure(CliBroker.java:111) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideModule -> io.druid.
cli.CliBroker$1)
  while locating io.druid.server.BrokerQueryResource
```

## MMAP failure on historical
This happens when there is no enough memory in the system to allocate in order to load more segments by historical.
It is a case by case issue, most of the time the solution is to add more historicals or reduce the `druid.server.maxSize` property.

```java
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (malloc) failed to allocate 28520448 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /usr/local/druid-0.8.0-rc1/hs_err_pid25770.log

```

```java
 Caused by: java.io.IOException: Map failed
 at sun.nio.ch.FileChannelImpl.map(FileChannelImpl.java:940) ~[?:1.8.0_91]
 at com.google.common.io.Files.map(Files.java:864) ~[guava-16.0.1.jar:?]
 at com.google.common.io.Files.map(Files.java:851) ~[guava-16.0.1.jar:?]
 at com.google.common.io.Files.map(Files.java:818) ~[guava-16.0.1.jar:?]
 at com.google.common.io.Files.map(Files.java:790) ~[guava-16.0.1.jar:?]
 at com.metamx.common.io.smoosh.SmooshedFileMapper.mapFile(SmooshedFileMapper.java:124) ~[java-util-0.27.9.jar:?]
 at io.druid.segment.IndexIO$V9IndexLoader.load(IndexIO.java:1023) ~[druid-processing-0.9.1.1.jar:0.9.1.1]
 at io.druid.segment.IndexIO.loadIndex(IndexIO.java:216) ~[druid-processing-0.9.1.1.jar:0.9.1.1]
 at io.druid.segment.loading.MMappedQueryableIndexFactory.factorize(MMappedQueryableIndexFactory.java:49) ~[druid-server-0.9.1.1.jar:0.9.1.1]
 at io.druid.segment.loading.SegmentLoaderLocalCacheManager.getSegment(SegmentLoaderLocalCacheManager.java:96) ~[druid-server-0.9.1.1.jar:0.9.1.1]
 at io.druid.server.coordination.ServerManager.loadSegment(ServerManager.java:152) ~[druid-server-0.9.1.1.jar:0.9.1.1]
 at io.druid.server.coordination.ZkCoordinator.loadSegment(ZkCoordinator.java:305) ~[druid-server-0.9.1.1.jar:0.9.1.1]
 ... 7 more
 Caused by: java.lang.OutOfMemoryError: Map failed
 at sun.nio.ch.FileChannelImpl.map0(Native Method) ~[?:1.8.0_91]
 at sun.nio.ch.FileChannelImpl.map(FileChannelImpl.java:937) ~[?:1.8.0_91]
 at com.google.common.io.Files.map(Files.java:864) ~[guava-16.0.1.jar:?]
 at com.google.common.io.Files.map(Files.java:851) ~[guava-16.0.1.jar:?]
 at com.google.common.io.Files.map(Files.java:818) ~[guava-16.0.1.jar:?]
 at com.google.common.io.Files.map(Files.java:790) ~[guava-16.0.1.jar:?]
 at com.metamx.common.io.smoosh.SmooshedFileMapper.mapFile(SmooshedFileMapper.java:124) ~[java-util-0.27.9.jar:?]
 at io.druid.segment.IndexIO$V9IndexLoader.load(IndexIO.java:1023) ~[druid-processing-0.9.1.1.jar:0.9.1.1]
 at io.druid.segment.IndexIO.loadIndex(IndexIO.java:216) ~[druid-processing-0.9.1.1.jar:0.9.1.1]
 at io.druid.segment.loading.MMappedQueryableIndexFactory.factorize(MMappedQueryableIndexFactory.java:49) ~[druid-server-0.9.1.1.jar:0.9.1.1]
 at io.druid.segment.loading.SegmentLoaderLocalCacheManager.getSegment(SegmentLoaderLocalCacheManager.java:96) ~[druid-server-0.9.1.1.jar:0.9.1.1]
 at io.druid.server.coordination.ServerManager.loadSegment(ServerManager.java:152) ~[druid-server-0.9.1.1.jar:0.9.1.1]
 at io.druid.server.coordination.ZkCoordinator.loadSegment(ZkCoordinator.java:305) ~[druid-server-0.9.1.1.jar:0.9.1.1]
 ... 7 more
```
## Buffer Sizes

### Not enough memory to allocate for byte buffer allocation
```java
2017-09-07T19:51:40,928 INFO [main] io.druid.offheap.OffheapBufferGenerator - Allocating new result merging buffer[0] of size[1,073,741,824]
2017-09-07T19:51:40,934 ERROR [main] io.druid.cli.CliBroker - Error when starting up.  Failing.
com.google.inject.ProvisionException: Unable to provision, see the following errors:

1) Error in custom provider, java.lang.OutOfMemoryError
  at io.druid.guice.DruidProcessingModule.getIntermediateResultsPool(DruidProcessingModule.java:109) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideModule -> io.druid.guice.DruidProcessingModule)
  at io.druid.guice.DruidProcessingModule.getIntermediateResultsPool(DruidProcessingModule.java:109) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideModule -> io.druid.guice.DruidProcessingModule)
  while locating io.druid.collections.NonBlockingPool<java.nio.ByteBuffer> annotated with @io.druid.guice.annotations.Global()
    for the 2nd parameter of io.druid.query.groupby.GroupByQueryEngine.<init>(GroupByQueryEngine.java:81)
  at io.druid.guice.QueryRunnerFactoryModule.configure(QueryRunnerFactoryModule.java:85) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideModule -> io.druid.guice.QueryRunnerFactoryModule)
  while locating io.druid.query.groupby.GroupByQueryEngine
    for the 2nd parameter of io.druid.query.groupby.strategy.GroupByStrategyV1.<init>(GroupByStrategyV1.java:77)
  while locating io.druid.query.groupby.strategy.GroupByStrategyV1
    for the 2nd parameter of io.druid.query.groupby.strategy.GroupByStrategySelector.<init>(GroupByStrategySelector.java:43)
  while locating io.druid.query.groupby.strategy.GroupByStrategySelector
    for the 1st parameter of io.druid.query.groupby.GroupByQueryQueryToolChest.<init>(GroupByQueryQueryToolChest.java:104)
  at io.druid.guice.QueryToolChestModule.configure(QueryToolChestModule.java:91) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideModule -> io.druid.guice.QueryRunnerFactoryModule)
  while locating io.druid.query.groupby.GroupByQueryQueryToolChest
  while locating io.druid.query.QueryToolChest annotated with @com.google.inject.multibindings.Element(setName=,uniqueId=54, type=MAPBINDER, keyType=java.lang.Class<? extends io.druid.query.Query>)
  at io.druid.guice.DruidBinders.queryToolChestBinder(DruidBinders.java:45) (via modules: com.google.inject.util.Modules$OverrideModule -> com.google.inject.util.Modules$OverrideModule -> io.druid.guice.QueryRunnerFactoryModule -> com.google.inject.multibindings.MapBinder$RealMapBinder)
  while locating java.util.Map<java.lang.Class<? extends io.druid.query.Query>, io.druid.query.QueryToolChest>
    for the 1st parameter of io.druid.query.MapQueryToolChestWarehouse.<init>(MapQueryToolChestWarehouse.java:36)
  while locating io.druid.query.MapQueryToolChestWarehouse
  while locating io.druid.query.QueryToolChestWarehouse
    for the 1st parameter of io.druid.server.BrokerQueryResource.<init>(BrokerQueryResource.java:75)
  while locating io.druid.server.BrokerQueryResource
Caused by: java.lang.OutOfMemoryError
	at sun.misc.Unsafe.allocateMemory(Native Method)
	at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:127)
	at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:311)
	at io.druid.offheap.OffheapBufferGenerator.get(OffheapBufferGenerator.java:53)
	at io.druid.offheap.OffheapBufferGenerator.get(OffheapBufferGenerator.java:29)
	at io.druid.collections.StupidPool.makeObjectWithHandler(StupidPool.java:112)
	at io.druid.collections.StupidPool.<init>(StupidPool.java:83)
	at io.druid.guice.DruidProcessingModule.getIntermediateResultsPool(DruidProcessingModule.java:114)
	at io.druid.guice.DruidProcessingModule$$FastClassByGuice$$8e266e5c.invoke(<generated>)
	at com.google.inject.internal.ProviderMethod$FastClassProviderMethod.doProvision(ProviderMethod.java:264)
	at com.google.inject.internal.ProviderMethod$Factory.provision(ProviderMethod.java:401)
	at com.google.inject.internal.ProviderMethod$Factory.get(ProviderMethod.java:376)
	at com.google.inject.internal.ProviderToInternalFactoryAdapter$1.call(ProviderToInternalFactoryAdapter.java:46)
	at com.google.inject.internal.InjectorImpl.callInContext(InjectorImpl.java:1092)
	at com.google.inject.internal.ProviderToInternalFactoryAdapter.get(ProviderToInternalFactoryAdapter.java:40)
	at com.google.inject.internal.SingletonScope$1.get(SingletonScope.java:194)
	at com.google.inject.internal.InternalFactoryToProviderAdapter.get(InternalFactoryToProviderAdapter.java:41)
	at com.google.inject.internal.SingleParameterInjector.inject(SingleParameterInjector.java:38)
	at com.google.inject.internal.SingleParameterInjector.getAll(SingleParameterInjector.java:62)
	at com.google.inject.internal.ConstructorInjector.provision(ConstructorInjector.java:110)
	at com.google.inject.internal.ConstructorInjector.construct(ConstructorInjector.java:90)
	at com.google.inject.internal.ConstructorBindingImpl$Factory.get(ConstructorBindingImpl.java:268)
	at com.google.inject.internal.ProviderToInternalFactoryAdapter$1.call(ProviderToInternalFactoryAdapter.java:46)
	at com.google.inject.internal.InjectorImpl.callInContext(InjectorImpl.java:1092)
	at com.google.inject.internal.ProviderToInternalFactoryAdapter.get(ProviderToInternalFactoryAdapter.java:40)
	at com.google.inject.internal.SingletonScope$1.get(SingletonScope.java:194)
	at com.google.inject.internal.InternalFactoryToProviderAdapter.get(InternalFactoryToProviderAdapter.java:41)
	at com.google.inject.internal.SingleParameterInjector.inject(SingleParameterInjector.java:38)
	at com.google.inject.internal.SingleParameterInjector.getAll(SingleParameterInjector.java:62)
	at com.google.inject.internal.ConstructorInjector.provision(ConstructorInjector.java:110)
	at com.google.inject.internal.ConstructorInjector.construct(ConstructorInjector.java:90)
	at com.google.inject.internal.ConstructorBindingImpl$Factory.get(ConstructorBindingImpl.java:268)
	at com.google.inject.internal.SingleParameterInjector.inject(SingleParameterInjector.java:38)
	at com.google.inject.internal.SingleParameterInjector.getAll(SingleParameterInjector.java:62)
	at com.google.inject.internal.ConstructorInjector.provision(ConstructorInjector.java:110)
	at com.google.inject.internal.ConstructorInjector.construct(ConstructorInjector.java:90)
	at com.google.inject.internal.ConstructorBindingImpl$Factory.get(ConstructorBindingImpl.java:268)
	at com.google.inject.internal.SingleParameterInjector.inject(SingleParameterInjector.java:38)
	at com.google.inject.internal.SingleParameterInjector.getAll(SingleParameterInjector.java:62)
	at com.google.inject.internal.ConstructorInjector.provision(ConstructorInjector.java:110)
	at com.google.inject.internal.ConstructorInjector.construct(ConstructorInjector.java:90)
	at com.google.inject.internal.ConstructorBindingImpl$Factory.get(ConstructorBindingImpl.java:268)
	at com.google.inject.internal.ProviderToInternalFactoryAdapter$1.call(ProviderToInternalFactoryAdapter.java:46)
	at com.google.inject.internal.InjectorImpl.callInContext(InjectorImpl.java:1092)
	at com.google.inject.internal.ProviderToInternalFactoryAdapter.get(ProviderToInternalFactoryAdapter.java:40)
	at com.google.inject.internal.SingletonScope$1.get(SingletonScope.java:194)
	at com.google.inject.internal.InternalFactoryToProviderAdapter.get(InternalFactoryToProviderAdapter.java:41)
	at com.google.inject.internal.FactoryProxy.get(FactoryProxy.java:56)
	at com.google.inject.internal.InjectorImpl$2$1.call(InjectorImpl.java:1019)
	at com.google.inject.internal.InjectorImpl.callInContext(InjectorImpl.java:1092)
	at com.google.inject.internal.InjectorImpl$2.get(InjectorImpl.java:1015)
	at com.google.inject.spi.ProviderLookup$1.get(ProviderLookup.java:104)
	at com.google.inject.spi.ProviderLookup$1.get(ProviderLookup.java:104)
	at com.google.inject.spi.ProviderLookup$1.get(ProviderLookup.java:104)
	at com.google.inject.multibindings.MapBinder$RealMapBinder$ValueProvider.get(MapBinder.java:821)
	at com.google.inject.multibindings.MapBinder$RealMapBinder$RealMapProvider.get(MapBinder.java:605)
	at com.google.inject.multibindings.MapBinder$RealMapBinder$RealMapProvider.get(MapBinder.java:586)
	at com.google.inject.internal.ProviderInternalFactory.provision(ProviderInternalFactory.java:81)
	at com.google.inject.internal.InternalFactoryToInitializableAdapter.provision(InternalFactoryToInitializableAdapter.java:53)
	at com.google.inject.internal.ProviderInternalFactory.circularGet(ProviderInternalFactory.java:61)
	at com.google.inject.internal.InternalFactoryToInitializableAdapter.get(InternalFactoryToInitializableAdapter.java:45)
	at com.google.inject.internal.SingleParameterInjector.inject(SingleParameterInjector.java:38)
	at com.google.inject.internal.SingleParameterInjector.getAll(SingleParameterInjector.java:62)
	at com.google.inject.internal.ConstructorInjector.provision(ConstructorInjector.java:110)
	at com.google.inject.internal.ConstructorInjector.construct(ConstructorInjector.java:90)
	at com.google.inject.internal.ConstructorBindingImpl$Factory.get(ConstructorBindingImpl.java:268)
	at com.google.inject.internal.FactoryProxy.get(FactoryProxy.java:56)
	at com.google.inject.internal.SingleParameterInjector.inject(SingleParameterInjector.java:38)
	at com.google.inject.internal.SingleParameterInjector.getAll(SingleParameterInjector.java:62)
	at com.google.inject.internal.ConstructorInjector.provision(ConstructorInjector.java:110)
	at com.google.inject.internal.ConstructorInjector.construct(ConstructorInjector.java:90)
	at com.google.inject.internal.ConstructorBindingImpl$Factory.get(ConstructorBindingImpl.java:268)
	at com.google.inject.internal.InjectorImpl$2$1.call(InjectorImpl.java:1019)
	at com.google.inject.internal.InjectorImpl.callInContext(InjectorImpl.java:1085)
	at com.google.inject.internal.InjectorImpl$2.get(InjectorImpl.java:1015)
	at com.google.inject.internal.InjectorImpl.getInstance(InjectorImpl.java:1050)
	at io.druid.guice.LifecycleModule$2.start(LifecycleModule.java:154)
	at io.druid.cli.GuiceRunnable.initLifecycle(GuiceRunnable.java:103)
	at io.druid.cli.ServerRunnable.run(ServerRunnable.java:41)
	at io.druid.cli.Main.main(Main.java:108)
```

### Mapreduce memory issue
This happens when the amount of memory allocated by single mapper/reducer is higher than yarn/mapreduce limits

```java
Container [pid=4869,containerID=container_e08_1508530139964_0003_01_000002] is running beyond physical memory limits. Current usage: 286.8 MB of 256 MB physical memory used; 2.0 GB of 537.6 MB virtual memory used. Killing container.
Dump of the process-tree for container_e08_1508530139964_0003_01_000002 :
	|- PID PPID PGRPID SESSID CMD_NAME USER_MODE_TIME(MILLIS) SYSTEM_TIME(MILLIS) VMEM_USAGE(BYTES) RSSMEM_USAGE(PAGES) FULL_CMD_LINE
	|- 5088 4869 4869 4869 (java) 1045 130 2005045248 73086 /usr/jdk64/jdk1.8.0_112/bin/java -server -XX:NewRatio=8 -Djava.net.preferIPv4Stack=true -Dhdp.version=2.6.3.0-212 -Xmx204m -Ddruid.storage.storageDirectory=/user/druid/data -Ddruid.storage.type=hdfs -Djava.io.tmpdir=/hadoop/yarn/local/usercache/druid/appcache/application_1508530139964_0003/container_e08_1508530139964_0003_01_000002/tmp -Dlog4j.configuration=container-log4j.properties -Dyarn.app.container.log.dir=/hadoop/yarn/log/application_1508530139964_0003/container_e08_1508530139964_0003_01_000002 -Dyarn.app.container.log.filesize=0 -Dhadoop.root.logger=INFO,CLA -Dhadoop.root.logfile=syslog org.apache.hadoop.mapred.YarnChild 192.168.56.121 56571 attempt_1508530139964_0003_m_000000_0 8796093022210 
	|- 4869 4867 4869 4869 (bash) 3 3 108625920 330 /bin/bash -c /usr/jdk64/jdk1.8.0_112/bin/java -server -XX:NewRatio=8 -Djava.net.preferIPv4Stack=true -Dhdp.version=2.6.3.0-212 -Xmx204m -Ddruid.storage.storageDirectory=/user/druid/data -Ddruid.storage.type=hdfs -Djava.io.tmpdir=/hadoop/yarn/local/usercache/druid/appcache/application_1508530139964_0003/container_e08_1508530139964_0003_01_000002/tmp -Dlog4j.configuration=container-log4j.properties -Dyarn.app.container.log.dir=/hadoop/yarn/log/application_1508530139964_0003/container_e08_1508530139964_0003_01_000002 -Dyarn.app.container.log.filesize=0 -Dhadoop.root.logger=INFO,CLA -Dhadoop.root.logfile=syslog org.apache.hadoop.mapred.YarnChild 192.168.56.121 56571 attempt_1508530139964_0003_m_000000_0 8796093022210 1>/hadoop/yarn/log/application_1508530139964_0003/container_e08_1508530139964_0003_01_000002/stdout 2>/hadoop/yarn/log/application_1508530139964_0003/container_e08_1508530139964_0003_01_000002/stderr  

Container killed on request. Exit code is 143
Container exited with a non-zero exit code 143. 

2017-10-21T18:44:50,414 INFO [task-runner-0-priority-0] org.apache.hadoop.mapreduce.Job -  map 0% reduce 0%
2017-10-21T18:44:57,470 INFO [task-runner-0-priority-0] org.apache.hadoop.mapreduce.Job - Task Id : attempt_1508530139964_0003_m_000000_1, Status : FAILED
Container [pid=5164,containerID=container_e08_1508530139964_0003_01_000003] is running beyond physical memory limits. Current usage: 287.4 MB of 256 MB physical memory used; 2.0 GB of 537.6 MB virtual memory used. Killing container.
Dump of the process-tree for container_e08_1508530139964_0003_01_000003 :
	|- PID PPID PGRPID SESSID CMD_NAME USER_MODE_TIME(MILLIS) SYSTEM_TIME(MILLIS) VMEM_USAGE(BYTES) RSSMEM_USAGE(PAGES) FULL_CMD_LINE
	|- 5383 5164 5164 5164 (java) 664 90 2004996096 73239 /usr/jdk64/jdk1.8.0_112/bin/java -server -XX:NewRatio=8 -Djava.net.preferIPv4Stack=true -Dhdp.version=2.6.3.0-212 -Xmx204m -Ddruid.storage.storageDirectory=/user/druid/data -Ddruid.storage.type=hdfs -Djava.io.tmpdir=/hadoop/yarn/local/usercache/druid/appcache/application_1508530139964_0003/container_e08_1508530139964_0003_01_000003/tmp -Dlog4j.configuration=container-log4j.properties -Dyarn.app.container.log.dir=/hadoop/yarn/log/application_1508530139964_0003/container_e08_1508530139964_0003_01_000003 -Dyarn.app.container.log.filesize=0 -Dhadoop.root.logger=INFO,CLA -Dhadoop.root.logfile=syslog org.apache.hadoop.mapred.YarnChild 192.168.56.121 56571 attempt_1508530139964_0003_m_000000_1 8796093022211 
	|- 5164 5162 5164 5164 (bash) 2 4 108625920 331 /bin/bash -c /usr/jdk64/jdk1.8.0_112/bin/java -server -XX:NewRatio=8 -Djava.net.preferIPv4Stack=true -Dhdp.version=2.6.3.0-212 -Xmx204m -Ddruid.storage.storageDirectory=/user/druid/data -Ddruid.storage.type=hdfs -Djava.io.tmpdir=/hadoop/yarn/local/usercache/druid/appcache/application_1508530139964_0003/container_e08_1508530139964_0003_01_000003/tmp -Dlog4j.configuration=container-log4j.properties -Dyarn.app.container.log.dir=/hadoop/yarn/log/application_1508530139964_0003/container_e08_1508530139964_0003_01_000003 -Dyarn.app.container.log.filesize=0 -Dhadoop.root.logger=INFO,CLA -Dhadoop.root.logfile=syslog org.apache.hadoop.mapred.YarnChild 192.168.56.121 56571 attempt_1508530139964_0003_m_000000_1 8796093022211 1>/hadoop/yarn/log/application_1508530139964_0003/container_e08_1508530139964_0003_01_000003/stdout 2>/hadoop/yarn/log/application_1508530139964_0003/container_e08_1508530139964_0003_01_000003/stderr  

Container killed on request. Exit code is 143
Container exited with a non-zero exit code 143. 

```

### Query resourece limit
Query fails because of resource limit
```json
{
  "error" : "Resource limit exceeded",
  "errorMessage" : "Not enough aggregation table space to execute this query. Try increasing druid.processing.buffer.sizeBytes or enable disk spilling by setting druid.query.groupBy.maxOnDiskStorage to a positive number.",
  "errorClass" : "io.druid.query.ResourceLimitExceededException",
  "host" : "localhost:8100"
}
```



to kill yarn app
yarn application -kill  application_1508802793765_0005


# Personal notes setup from my local laptop
cd /Users/sbouguerra/Ydev/druid/examples/quickstart
transfer wikiticker-2015-09-12-sampled.json.gz

cd /Users/sbouguerra/Ydev/druid/qtl_tests
transfer index_task_hdfs.json
