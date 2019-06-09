---
layout: post
title:  "Hive Phoenix Storage Handler on HDP"
date:   2019-06-09 05:27:29 -0400
categories: HDP Hive
author: Thomas Kreutzer
---
### Use Case:

Recently when working on an application with a client we had a need to have fast interactions with small amounts of data but it needed to be available immediately to Hive. The synchronization was critical for another batch process that was decoupled from the interactions of the application. Yes, we could have probably handled this many ways but in the end I wanted to see if we could have the required inserts handled via the web application running on these windows servers. In a prior blog entry I talk about TLS with PowerShell which led into some of these tests. 

### HDP Configuration

In the Ambari UI we go to the **Hive config** tab for the following and entered **/usr/hdp/current/phoenix-client/phoenix-hive.jar** in the **hive_env.sh** in two places. **NOTE:** these are comma separated values and the first value **hive-hcatalog** existed prior.. 

![hive_env.sh](https://i.imgur.com/SLkCzDA.png)


This will open up a new panel for **NiFi Flow Configuration** and click on **CONTROLLER SERVICES**, this will display all the controller services defined at this level. **NOTE:** this is currently being done at the root canvas **Nifi Flow**. 

I also added the custom hive site property for both Hive 1.x and LLAP:

**hive.aux.jars.path=file://usr/hdp/current/phoenix-client/phoenix-hive.jar**

![Hive 1.2](https://i.imgur.com/q1feDJa.png)

![Hive 2.x](https://i.imgur.com/DMJOplz.png)

After completing I restarted all hive services required and attempted the following query which failed. 


{% highlight sql %}
create table {yourhivedb}.phoenix_table (
  s1 string,
  i1 int,
  f1 float,
  d1 double
)
STORED BY 'org.apache.phoenix.hive.PhoenixStorageHandler'
TBLPROPERTIES (
  "phoenix.table.name" = "{yourphoenixdb}.phoenix_table",
  "phoenix.zookeeper.quorum" = "{removed}.com,{removed}.com,{removed}.com",
  "phoenix.zookeeper.znode.parent" = "/hbase",
  "phoenix.zookeeper.client.port" = "2181",
  "phoenix.rowkeys" = "s1, i1",
  "phoenix.column.mapping" = "s1:s1, i1:i1, f1:f1, d1:d1",
  "phoenix.table.options" = "SALT_BUCKETS=10, DATA_BLOCK_ENCODING='DIFF'"
);
{% endhighlight %}

The error as follows:

{% highlight none %}
Error: Error while processing statement: 
FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. MetaException(message:org.apache.hadoop.hbase.client.RetriesExhaustedException: Can't get the location for replica 0)
(state=08S01,code=1)
{% endhighlight %}

Upon researching, this is a znode issue. This should have been obvious as Kerberos is enabled. It is set as **/habse** and should be **/hbase-secure** 
Modified Query:

{% highlight sql %}
CREATE TABLE {yourhivedb}.phoenix_table (
  s1 string,
  i1 int,
  f1 float,
  d1 double
)
STORED BY 'org.apache.phoenix.hive.PhoenixStorageHandler'
TBLPROPERTIES (
  "phoenix.table.name" = "{yourphoenixdb}.phoenix_table",
  "phoenix.zookeeper.quorum" = "{removed}.com,{removed}.com,{removed}.com",
  "phoenix.zookeeper.znode.parent" = "/hbase-secure",
  "phoenix.zookeeper.client.port" = "2181",
  "phoenix.rowkeys" = "s1, i1",
  "phoenix.column.mapping" = "s1:s1, i1:i1, f1:f1, d1:d1",
  "phoenix.table.options" = "SALT_BUCKETS=10, DATA_BLOCK_ENCODING='DIFF'"
);

{% endhighlight %}

Now I am getting a new error:
{% highlight none %}
Error: Error while processing statement: FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. MetaException(message:org.apache.phoenix.exception.PhoenixIOException: org.apache.hadoop.hbase.security.AccessDeniedException: Insufficient permissions for user âhive/ln112133.eh.pweh.com@HDP-DEV.PW.UTC.COM',action: scannerOpen, tableName:SYSTEM:CATALOG, family:0, column: TYPE_NAME
        at org.apache.ranger.authorization.hbase.RangerAuthorizationCoprocessor.authorizeAccess(RangerAuthorizationCoprocessor.java:551)
        at org.apache.ranger.authorization.hbase.RangerAuthorizationCoprocessor.preScannerOpen(RangerAuthorizationCoprocessor.java:955)
        at org.apache.ranger.authorization.hbase.RangerAuthorizationCoprocessor.preScannerOpen(RangerAuthorizationCoprocessor.java:855)
        at org.apache.hadoop.hbase.regionserver.RegionCoprocessorHost$50.call(RegionCoprocessorHost.java:1267)
        at org.apache.hadoop.hbase.regionserver.RegionCoprocessorHost$RegionOperation.call(RegionCoprocessorHost.java:1660)
        at org.apache.hadoop.hbase.regionserver.RegionCoprocessorHost.execOperation(RegionCoprocessorHost.java:1734)
        at org.apache.hadoop.hbase.regionserver.RegionCoprocessorHost.execOperationWithResult(RegionCoprocessorHost.java:1709)
        at org.apache.hadoop.hbase.regionserver.RegionCoprocessorHost.preScannerOpen(RegionCoprocessorHost.java:1262)
        at org.apache.hadoop.hbase.regionserver.RSRpcServices.scan(RSRpcServices.java:2418)
        at org.apache.hadoop.hbase.protobuf.generated.ClientProtos$ClientService$2.callBlockingMethod(ClientProtos.java:32385)
        at org.apache.hadoop.hbase.ipc.RpcServer.call(RpcServer.java:2150)
        at org.apache.hadoop.hbase.ipc.CallRunner.run(CallRunner.java:112)
        at org.apache.hadoop.hbase.ipc.RpcExecutor$Handler.run(RpcExecutor.java:187)
        at org.apache.hadoop.hbase.ipc.RpcExecutor$Handler.run(RpcExecutor.java:167)

{% endhighlight %}


This is easily confirmed in Ranger to be a permission issue, if you have it configured to log. 
![Ranger permissions](https://i.imgur.com/8cQARc7.png)

I will provide the proper permissions to HIVE and re-execute:
![Ranger Permissions](https://i.imgur.com/op2UyaT.png)

![Ranger Permissions](https://i.imgur.com/UxRKmHG.png)

**NOTE:** Also added the user access to the DB Tables for HBASE. I had attempted this with lowercase and yourphoenixdb:*, but it appears to get created all uppercase including the table name. 

![Ranger permissions](https://i.imgur.com/vcyPucX.png)

![Ranger permissions](https://i.imgur.com/30zd86t.png)

Validate that our policy changes are pushed down, assuming you are using rangers audit.
![Policy pushed to ranger](https://i.imgur.com/vj3wNIJ.png)

Now we have successful creation and a test select on the empty table. 
![Hive Select with Phoenix](https://i.imgur.com/T1frsNM.png)

Testing an insert through hive 1.2, results of insert are slow because of the additional overhead. 

![insert](https://i.imgur.com/pJBhX5l.png)

We will connect with sql line and see if it is faster. 

{% highlight shell %}
#From a host with the phoenix query server installed
cd /usr/hdp/current/phoenix-server/bin/
./sqlline.py
{% endhighlight %}


It’s much faster
{% highlight shell %}
3 rows selected (0.029 seconds)
{% endhighlight %}

The tests onn LLAP/ hive 2.x failed and the feature is not currently supported, I hope to get it working in the future and follow up.
