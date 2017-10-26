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


## Indexing data issue
### 
This will show on the coordinator
```java
2017-10-26T02:42:04,178 WARN [Coordinator-Exec--0] io.druid.server.coordinator.rules.LoadRule - Not enough [_default_tier] servers or node capacity to assign segment[wiki5_2015-09-01T00:00:00.000Z_2015-10-01T00:00:00.000Z_2017-10-26T02:37:31.038Z]! Expected Replicants[2]
```
And this will be printed on the historical

```java
2017-10-26T02:44:04,251 ERROR [ZkCoordinator-0] io.druid.server.coordination.ZkCoordinator - Failed to load segment for dataSource: {class=io.druid.server.coordination.ZkCoordinator, exceptionType=class io.druid.segment.loading.SegmentLoadingException, exceptionMessage=Exception loading segment[wiki5_2015-09-01T00:00:00.000Z_2015-10-01T00:00:00.000Z_2017-10-26T02:37:31.038Z], segment=DataSegment{size=4354747, shardSpec=NoneShardSpec, metrics=[count, added, deleted, delta], dimensions=[channel, comment, isAnonymous, isMinor, isNew, isRobot, isUnpatrolled, namespace, page, user, cityName, countryIsoCode, countryName, regionIsoCode, regionName, metroCode], version='2017-10-26T02:37:31.038Z', loadSpec={type=hdfs, path=hdfs://ctr-e134-1499953498516-247377-01-000005.hwx.site:8020/apps/druid/warehouse/wiki5/20150901T000000.000Z_20151001T000000.000Z/2017-10-26T02_37_31.038Z/0_index.zip}, interval=2015-09-01T00:00:00.000Z/2015-10-01T00:00:00.000Z, dataSource='wiki5', binaryVersion='9'}}
io.druid.segment.loading.SegmentLoadingException: Exception loading segment[wiki5_2015-09-01T00:00:00.000Z_2015-10-01T00:00:00.000Z_2017-10-26T02:37:31.038Z]
        at io.druid.server.coordination.ZkCoordinator.loadSegment(ZkCoordinator.java:327) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.coordination.ZkCoordinator.addSegment(ZkCoordinator.java:368) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.coordination.SegmentChangeRequestLoad.go(SegmentChangeRequestLoad.java:45) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.coordination.ZkCoordinator$1.childEvent(ZkCoordinator.java:158) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at org.apache.curator.framework.recipes.cache.PathChildrenCache$5.apply(PathChildrenCache.java:522) [curator-recipes-2.11.0.jar:?]
        at org.apache.curator.framework.recipes.cache.PathChildrenCache$5.apply(PathChildrenCache.java:516) [curator-recipes-2.11.0.jar:?]
        at org.apache.curator.framework.listen.ListenerContainer$1.run(ListenerContainer.java:93) [curator-framework-2.11.0.jar:?]
        at com.google.common.util.concurrent.MoreExecutors$SameThreadExecutorService.execute(MoreExecutors.java:297) [guava-16.0.1.jar:?]
        at org.apache.curator.framework.listen.ListenerContainer.forEach(ListenerContainer.java:84) [curator-framework-2.11.0.jar:?]
        at org.apache.curator.framework.recipes.cache.PathChildrenCache.callListeners(PathChildrenCache.java:513) [curator-recipes-2.11.0.jar:?]
        at org.apache.curator.framework.recipes.cache.EventOperation.invoke(EventOperation.java:35) [curator-recipes-2.11.0.jar:?]
        at org.apache.curator.framework.recipes.cache.PathChildrenCache$9.run(PathChildrenCache.java:773) [curator-recipes-2.11.0.jar:?]
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [?:1.8.0_112]
        at java.util.concurrent.FutureTask.run(FutureTask.java:266) [?:1.8.0_112]
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [?:1.8.0_112]
        at java.util.concurrent.FutureTask.run(FutureTask.java:266) [?:1.8.0_112]
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [?:1.8.0_112]
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [?:1.8.0_112]
        at java.lang.Thread.run(Thread.java:745) [?:1.8.0_112]
Caused by: com.metamx.common.ISE: Segment[wiki5_2015-09-01T00:00:00.000Z_2015-10-01T00:00:00.000Z_2017-10-26T02:37:31.038Z:4,354,747] too large for storage[/apps/druid/segmentCache:-4,354,717].
        at io.druid.segment.loading.SegmentLoaderLocalCacheManager.loadSegmentWithRetry(SegmentLoaderLocalCacheManager.java:148) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.segment.loading.SegmentLoaderLocalCacheManager.getSegmentFiles(SegmentLoaderLocalCacheManager.java:130) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.segment.loading.SegmentLoaderLocalCacheManager.getSegment(SegmentLoaderLocalCacheManager.java:105) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.SegmentManager.getAdapter(SegmentManager.java:197) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.SegmentManager.loadSegment(SegmentManager.java:158) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.coordination.ZkCoordinator.loadSegment(ZkCoordinator.java:323) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        ... 18 more
```

### Wrong input path

```java
2017-10-26T03:04:29,302 ERROR [task-runner-0-priority-0] io.druid.indexing.overlord.ThreadPoolTaskRunner - Exception while running task[HadoopIndexTask{id=index_hadoop_wiki5_2017-10-26T03:04:16.829Z, type=index_hadoop, dataSource=wiki5}]
java.lang.RuntimeException: java.lang.reflect.InvocationTargetException
	at com.google.common.base.Throwables.propagate(Throwables.java:160) ~[guava-16.0.1.jar:?]
	at io.druid.indexing.common.task.HadoopTask.invokeForeignLoader(HadoopTask.java:218) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexing.common.task.HadoopIndexTask.run(HadoopIndexTask.java:178) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexing.overlord.ThreadPoolTaskRunner$ThreadPoolTaskRunnerCallable.call(ThreadPoolTaskRunner.java:436) [druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexing.overlord.ThreadPoolTaskRunner$ThreadPoolTaskRunnerCallable.call(ThreadPoolTaskRunner.java:408) [druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at java.util.concurrent.FutureTask.run(FutureTask.java:266) [?:1.8.0_112]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [?:1.8.0_112]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [?:1.8.0_112]
	at java.lang.Thread.run(Thread.java:745) [?:1.8.0_112]
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_112]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_112]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_112]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_112]
	at io.druid.indexing.common.task.HadoopTask.invokeForeignLoader(HadoopTask.java:215) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	... 7 more
Caused by: java.lang.RuntimeException: org.apache.hadoop.mapreduce.lib.input.InvalidInputException: Input path does not exist: hdfs://ctr-e134-1499953498516-247377-01-000005.hwx.site:8020/appsz/druid/warehouse/wikiticker-2015-09-12-sampled.json
	at com.google.common.base.Throwables.propagate(Throwables.java:160) ~[guava-16.0.1.jar:?]
	at io.druid.indexer.DetermineHashedPartitionsJob.run(DetermineHashedPartitionsJob.java:210) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexer.JobHelper.runJobs(JobHelper.java:369) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexer.HadoopDruidDetermineConfigurationJob.run(HadoopDruidDetermineConfigurationJob.java:91) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexing.common.task.HadoopIndexTask$HadoopDetermineConfigInnerProcessing.runTask(HadoopIndexTask.java:308) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_112]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_112]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_112]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_112]
	at io.druid.indexing.common.task.HadoopTask.invokeForeignLoader(HadoopTask.java:215) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	... 7 more
Caused by: org.apache.hadoop.mapreduce.lib.input.InvalidInputException: Input path does not exist: hdfs://ctr-e134-1499953498516-247377-01-000005.hwx.site:8020/appsz/druid/warehouse/wikiticker-2015-09-12-sampled.json
	at org.apache.hadoop.mapreduce.lib.input.FileInputFormat.singleThreadedListStatus(FileInputFormat.java:323) ~[?:?]
	at org.apache.hadoop.mapreduce.lib.input.FileInputFormat.listStatus(FileInputFormat.java:265) ~[?:?]
	at org.apache.hadoop.mapreduce.lib.input.FileInputFormat.getSplits(FileInputFormat.java:387) ~[?:?]
	at org.apache.hadoop.mapreduce.lib.input.DelegatingInputFormat.getSplits(DelegatingInputFormat.java:115) ~[?:?]
	at org.apache.hadoop.mapreduce.JobSubmitter.writeNewSplits(JobSubmitter.java:301) ~[?:?]
	at org.apache.hadoop.mapreduce.JobSubmitter.writeSplits(JobSubmitter.java:318) ~[?:?]
	at org.apache.hadoop.mapreduce.JobSubmitter.submitJobInternal(JobSubmitter.java:196) ~[?:?]
	at org.apache.hadoop.mapreduce.Job$10.run(Job.java:1290) ~[?:?]
	at org.apache.hadoop.mapreduce.Job$10.run(Job.java:1287) ~[?:?]
	at java.security.AccessController.doPrivileged(Native Method) ~[?:1.8.0_112]
	at javax.security.auth.Subject.doAs(Subject.java:422) ~[?:1.8.0_112]
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1866) ~[?:?]
	at org.apache.hadoop.mapreduce.Job.submit(Job.java:1287) ~[?:?]
	at io.druid.indexer.DetermineHashedPartitionsJob.run(DetermineHashedPartitionsJob.java:118) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexer.JobHelper.runJobs(JobHelper.java:369) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexer.HadoopDruidDetermineConfigurationJob.run(HadoopDruidDetermineConfigurationJob.java:91) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexing.common.task.HadoopIndexTask$HadoopDetermineConfigInnerProcessing.runTask(HadoopIndexTask.java:308) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_112]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_112]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_112]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_112]
	at io.druid.indexing.common.task.HadoopTask.invokeForeignLoader(HadoopTask.java:215) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	... 7 more
```

HDFS access issue when indexing 

```java
Caused by: java.lang.RuntimeException: org.apache.hadoop.security.AccessControlException: Permission denied: user=druid, access=EXECUTE, inode="/apps/druid/warehouse/wikiticker-2015-09-12-sampled.json":hive:hive:d---rwx---
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:353)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkTraverse(FSPermissionChecker.java:292)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:238)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:190)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:1956)
	at org.apache.hadoop.hdfs.server.namenode.FSDirStatAndListingOp.getFileInfo(FSDirStatAndListingOp.java:108)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getFileInfo(FSNamesystem.java:4142)
	at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.getFileInfo(NameNodeRpcServer.java:1137)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.getFileInfo(ClientNamenodeProtocolServerSideTranslatorPB.java:866)
	at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:640)
	at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:982)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2351)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2347)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1866)
	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2347)

	at com.google.common.base.Throwables.propagate(Throwables.java:160) ~[guava-16.0.1.jar:?]
	at io.druid.indexer.DetermineHashedPartitionsJob.run(DetermineHashedPartitionsJob.java:210) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexer.JobHelper.runJobs(JobHelper.java:369) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexer.HadoopDruidDetermineConfigurationJob.run(HadoopDruidDetermineConfigurationJob.java:91) ~[druid-indexing-hadoop-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at io.druid.indexing.common.task.HadoopIndexTask$HadoopDetermineConfigInnerProcessing.runTask(HadoopIndexTask.java:308) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_112]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_112]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_112]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_112]
	at io.druid.indexing.common.task.HadoopTask.invokeForeignLoader(HadoopTask.java:215) ~[druid-indexing-service-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
	... 7 more
Caused by: org.apache.hadoop.security.AccessControlException: Permission denied: user=druid, access=EXECUTE, inode="/apps/druid/warehouse/wikiticker-2015-09-12-sampled.json":hive:hive:d---rwx---
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:353)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkTraverse(FSPermissionChecker.java:292)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:238)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:190)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:1956)
	at org.apache.hadoop.hdfs.server.namenode.FSDirStatAndListingOp.getFileInfo(FSDirStatAndListingOp.java:108)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getFileInfo(FSNamesystem.java:4142)
	at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.getFileInfo(NameNodeRpcServer.java:1137)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.getFileInfo(ClientNamenodeProtocolServerSideTranslatorPB.java:866)
	at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:640)
	at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:982)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2351)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2347)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1866)
	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2347)
```

### Loading segments from HDFS

```java
2017-10-26T03:24:05,141 ERROR [ZkCoordinator-0] io.druid.segment.loading.SegmentLoaderLocalCacheManager - Failed to load segment in current location /apps/druid/segmentCache, try next location if any: {class=io.druid.segment.loading.SegmentLoaderLocalCacheManager, exceptionType=class io.druid.segment.loading.SegmentLoadingException, exceptionMessage=Error loadin
g [hdfs://ctr-e134-1499953498516-247377-01-000005.hwx.site:8020/apps/druid/warehouse/wikipedia/20150912T000000.000Z_20150913T000000.000Z/2017-10-24T15_05_04.692Z/0_index.zip], location=/apps/druid/segmentCache}
io.druid.segment.loading.SegmentLoadingException: Error loading [hdfs://ctr-e134-1499953498516-247377-01-000005.hwx.site:8020/apps/druid/warehouse/wikipedia/20150912T000000.000Z_20150913T000000.000Z/2017-10-24T15_05_04.692Z/0_index.zip]
        at io.druid.storage.hdfs.HdfsDataSegmentPuller.getSegmentFiles(HdfsDataSegmentPuller.java:283) ~[?:?]
        at io.druid.storage.hdfs.HdfsLoadSpec.loadSegment(HdfsLoadSpec.java:62) ~[?:?]
        at io.druid.segment.loading.SegmentLoaderLocalCacheManager.loadInLocation(SegmentLoaderLocalCacheManager.java:206) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.segment.loading.SegmentLoaderLocalCacheManager.loadInLocationWithStartMarker(SegmentLoaderLocalCacheManager.java:195) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.segment.loading.SegmentLoaderLocalCacheManager.loadSegmentWithRetry(SegmentLoaderLocalCacheManager.java:154) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.segment.loading.SegmentLoaderLocalCacheManager.getSegmentFiles(SegmentLoaderLocalCacheManager.java:130) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.segment.loading.SegmentLoaderLocalCacheManager.getSegment(SegmentLoaderLocalCacheManager.java:105) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.SegmentManager.getAdapter(SegmentManager.java:197) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.SegmentManager.loadSegment(SegmentManager.java:158) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.coordination.ZkCoordinator.loadSegment(ZkCoordinator.java:323) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.coordination.ZkCoordinator.addSegment(ZkCoordinator.java:368) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.coordination.SegmentChangeRequestLoad.go(SegmentChangeRequestLoad.java:45) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.server.coordination.ZkCoordinator$1.childEvent(ZkCoordinator.java:158) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at org.apache.curator.framework.recipes.cache.PathChildrenCache$5.apply(PathChildrenCache.java:522) [curator-recipes-2.11.0.jar:?]
        at org.apache.curator.framework.recipes.cache.PathChildrenCache$5.apply(PathChildrenCache.java:516) [curator-recipes-2.11.0.jar:?]
        at org.apache.curator.framework.listen.ListenerContainer$1.run(ListenerContainer.java:93) [curator-framework-2.11.0.jar:?]
        at com.google.common.util.concurrent.MoreExecutors$SameThreadExecutorService.execute(MoreExecutors.java:297) [guava-16.0.1.jar:?]
        at org.apache.curator.framework.listen.ListenerContainer.forEach(ListenerContainer.java:84) [curator-framework-2.11.0.jar:?]
        at org.apache.curator.framework.recipes.cache.PathChildrenCache.callListeners(PathChildrenCache.java:513) [curator-recipes-2.11.0.jar:?]
        at org.apache.curator.framework.recipes.cache.EventOperation.invoke(EventOperation.java:35) [curator-recipes-2.11.0.jar:?]
        at org.apache.curator.framework.recipes.cache.PathChildrenCache$9.run(PathChildrenCache.java:773) [curator-recipes-2.11.0.jar:?]
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [?:1.8.0_112]
        at java.util.concurrent.FutureTask.run(FutureTask.java:266) [?:1.8.0_112]
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [?:1.8.0_112]
        at java.util.concurrent.FutureTask.run(FutureTask.java:266) [?:1.8.0_112]
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [?:1.8.0_112]
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [?:1.8.0_112]
        at java.lang.Thread.run(Thread.java:745) [?:1.8.0_112]
Caused by: org.apache.hadoop.security.AccessControlException: Permission denied: user=druid, access=EXECUTE, inode="/apps/druid/warehouse/wikipedia/20150912T000000.000Z_20150913T000000.000Z/2017-10-24T15_05_04.692Z/0_index.zip":hive:hive:d---rwx---
        at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:353)
        at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkTraverse(FSPermissionChecker.java:292)
        at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:238)
        at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:190)
        at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:1956)
        at org.apache.hadoop.hdfs.server.namenode.FSDirStatAndListingOp.getFileInfo(FSDirStatAndListingOp.java:108)
        at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getFileInfo(FSNamesystem.java:4142)
        at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.getFileInfo(NameNodeRpcServer.java:1137)
        at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.getFileInfo(ClientNamenodeProtocolServerSideTranslatorPB.java:866)
        at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:640)
        at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:982)
        at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2351)
        at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2347)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:422)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1866)
        at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2347)
```

### Metadata access issues
This happens when the database is not created
```java
2017-10-26T03:49:16,286 ERROR [main] io.druid.cli.CliCoordinator - Error when starting up.  Failing.
org.skife.jdbi.v2.exceptions.CallbackFailedException: org.skife.jdbi.v2.exceptions.UnableToExecuteStatementException: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Tabl
e 'druid.druid_rules' doesn't exist [statement:"SELECT id from druid_rules where datasource=:dataSource", located:"SELECT id from druid_rules where datasource=:dataSource", rewritte
n:"SELECT id from druid_rules where datasource=?", arguments:{ positional:{}, named:{dataSource:'_default'}, finder:[]}]
        at org.skife.jdbi.v2.DBI.withHandle(DBI.java:284) ~[jdbi-2.63.1.jar:2.63.1]
        at io.druid.metadata.SQLMetadataRuleManager.createDefaultRule(SQLMetadataRuleManager.java:83) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.metadata.SQLMetadataRuleManagerProvider$1.start(SQLMetadataRuleManagerProvider.java:72) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.java.util.common.lifecycle.Lifecycle.start(Lifecycle.java:263) ~[java-util-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.guice.LifecycleModule$2.start(LifecycleModule.java:156) ~[druid-api-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.cli.GuiceRunnable.initLifecycle(GuiceRunnable.java:103) [druid-services-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.cli.ServerRunnable.run(ServerRunnable.java:41) [druid-services-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.cli.Main.main(Main.java:108) [druid-services-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
Caused by: org.skife.jdbi.v2.exceptions.UnableToExecuteStatementException: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Table 'druid.druid_rules' doesn't exist [statement:"SELECT id from druid_rules where datasource=:dataSource", located:"SELECT id from druid_rules where datasource=:dataSource", rewritten:"SELECT id from druid_rules where datasource=?", arguments:{ positional:{}, named:{dataSource:'_default'}, finder:[]}]
        at org.skife.jdbi.v2.SQLStatement.internalExecute(SQLStatement.java:1334) ~[jdbi-2.63.1.jar:2.63.1]
        at org.skife.jdbi.v2.Query.fold(Query.java:173) ~[jdbi-2.63.1.jar:2.63.1]
        at org.skife.jdbi.v2.Query.list(Query.java:82) ~[jdbi-2.63.1.jar:2.63.1]
        at org.skife.jdbi.v2.Query.list(Query.java:75) ~[jdbi-2.63.1.jar:2.63.1]
        at io.druid.metadata.SQLMetadataRuleManager$1.withHandle(SQLMetadataRuleManager.java:97) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.metadata.SQLMetadataRuleManager$1.withHandle(SQLMetadataRuleManager.java:85) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at org.skife.jdbi.v2.DBI.withHandle(DBI.java:281) ~[jdbi-2.63.1.jar:2.63.1]
        ... 7 more
Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Table 'druid.druid_rules' doesn't exist
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) ~[?:1.8.0_112]
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62) ~[?:1.8.0_112]
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[?:1.8.0_112]
        at java.lang.reflect.Constructor.newInstance(Constructor.java:423) ~[?:1.8.0_112]
        at com.mysql.jdbc.Util.handleNewInstance(Util.java:411) ~[?:?]
        at com.mysql.jdbc.Util.getInstance(Util.java:386) ~[?:?]
```

This happens when the encoding is not UTF8

```java
Manager.start()] on object[io.druid.metadata.SQLMetadataSupervisorManager@1f4f0fcc].
2017-10-26T03:58:24,826 WARN [main] io.druid.metadata.SQLMetadataConnector - Exception creating table
org.skife.jdbi.v2.exceptions.CallbackFailedException: io.druid.java.util.common.ISE: Database default character set is not UTF-8.
  Druid requires its MySQL database to be created using UTF-8 as default character set.
        at org.skife.jdbi.v2.DBI.withHandle(DBI.java:284) ~[jdbi-2.63.1.jar:2.63.1]
        at io.druid.metadata.SQLMetadataConnector$2.call(SQLMetadataConnector.java:130) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.java.util.common.RetryUtils.retry(RetryUtils.java:63) ~[java-util-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.java.util.common.RetryUtils.retry(RetryUtils.java:81) ~[java-util-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.metadata.SQLMetadataConnector.retryWithHandle(SQLMetadataConnector.java:134) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.metadata.SQLMetadataConnector.retryWithHandle(SQLMetadataConnector.java:143) ~[druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.metadata.SQLMetadataConnector.createTable(SQLMetadataConnector.java:184) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.metadata.SQLMetadataConnector.createSupervisorsTable(SQLMetadataConnector.java:379) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.metadata.SQLMetadataConnector.createSupervisorsTable(SQLMetadataConnector.java:504) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.metadata.SQLMetadataSupervisorManager.start(SQLMetadataSupervisorManager.java:79) [druid-server-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_112]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_112]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_112]
        at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_112]
        at io.druid.java.util.common.lifecycle.Lifecycle$AnnotationBasedHandler.start(Lifecycle.java:364) [java-util-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.java.util.common.lifecycle.Lifecycle.start(Lifecycle.java:263) [java-util-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.guice.LifecycleModule$2.start(LifecycleModule.java:156) [druid-api-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.cli.GuiceRunnable.initLifecycle(GuiceRunnable.java:103) [druid-services-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.cli.ServerRunnable.run(ServerRunnable.java:41) [druid-services-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
        at io.druid.cli.Main.main(Main.java:108) [druid-services-0.10.1.2.6.3.0-220.jar:0.10.1.2.6.3.0-220]
```
### ZK issue
Caution ambari does show that druid nodes are up and running even if ZK is down.

```java
        at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1125) [zookeeper-3.4.6.2.6.3.0-220.jar:3.4.6-220--1]
2017-10-25T23:22:07,116 WARN [main-SendThread(ctr-e134-1499953498516-247377-01-000004.hwx.site:2182)] org.apache.zookeeper.ClientCnxn - Session 0x0 for server null, unexpected error, closing socket connection and attempting reconnect
java.net.ConnectException: Connection refused
        at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method) ~[?:1.8.0_112]
        at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:717) ~[?:1.8.0_112]
        at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:361) ~[zookeeper-3.4.6.2.6.3.0-220.jar:3.4.6-220--1]
        at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1125) [zookeeper-3.4.6.2.6.3.0-220.jar:3.4.6-220--1]
2017-10-25T23:22:07,217 WARN [main-SendThread(ctr-e134-1499953498516-247377-01-000002.hwx.site:2182)] org.apache.zookeeper.ClientCnxn - Session 0x0 for server null, unexpected error, closing socket connection and attempting reconnect
```


to kill yarn app
yarn application -kill  application_1508802793765_0005


# Personal notes setup from my local laptop
cd /Users/sbouguerra/Ydev/druid/examples/quickstart
transfer wikiticker-2015-09-12-sampled.json.gz

cd /Users/sbouguerra/Ydev/druid/qtl_tests
transfer index_task_hdfs.json


CREATE DATABASE druid DEFAULT CHARACTER SET utf8;
