---
layout: post
title:  "Spark Hbase Connector Insert/Update with SHC on spark-shell"
date:   2019-07-18 08:27:29 -0400
categories: spark
author: Thomas Kreutzer
---
### Goal:

Gather the dependencies to quickly use spark shell with SHC to test inserts/updates of data in Hbase.

### Steps:

**Find the SHC library on your HDP cluster**
Assuming you have locate installed on your linux box, I used the following command to find the local copy of shc to use with my spark project. 


{% highlight shell %}
%locate shc-core
/usr/hdp/2.6.5.0-292/shc/shc-core-1.1.0.2.6.5.0-292.jar
{% endhighlight %}


In addition I configured the cluster to have hbase-site.xml as a symlink in the spark2 con, adding --files /etc/hbase/conf/hbase-site.xml did not work properly for me. 
{% highlight shell %}
sudo su
ln -s /etc/hbase/conf/hbase-site.xml /etc/spark2/conf/hbase-site.xml
{% endhighlight %}


I included all the jar files for shc and HBase on the command line as follows when starting spark-shell

{% highlight shell %}
spark-shell --driver-memory 5g --jars /usr/hdp/2.6.5.0-292/shc/shc-core-1.1.0.2.6.5.0-292.jar,/usr/hdp/current/hbase-client/lib/hbase-common.jar,/usr/hdp/current/hbase-client/lib/hbase-client.jar,/usr/hdp/current/hbase-client/lib/hbase-server.jar,/usr/hdp/current/hbase-client/lib/hbase-protocol.jar,/usr/hdp/current/hbase-client/lib/guava-12.0.1.jar,/usr/hdp/current/hbase-client/lib/htrace-core-3.1.0-incubating.jar
{% endhighlight %}


**Sample Code**
Finally, the sample code below was able to run an insert/update to an hbase cell in my table. You may notices the data types and I can't yet explain it. I ran into many issues with the class and object to get this to work.

{% highlight scala %}
import org.apache.spark.sql.{SQLContext, _}
import org.apache.spark.sql.execution.datasources.hbase._
import org.apache.spark.{SparkConf, SparkContext}

:paste
case class myRecord(
  id: String
  ,completed: String
)

object myRecord {
  def apply(id: Int, completed: Int): myRecord = {
    myRecord(
      id
      ,completed
    )
  }
} 

def mycatalog = s"""{
  "table":{"namespace":"default", "name":"mysparktable", "tableCoder":"PrimitiveType"},
  "rowkey":"key",
  "columns":{
    "id":{"cf":"rowkey", "col":"key", "type":"string"},
    "completed":{"cf":"f", "col":"c", "type":"String"}
  }
}""".stripMargin


def withCatalog(cat: String): DataFrame = {
spark
.read
.options(Map(HBaseTableCatalog.tableCatalog->cat))
.format("org.apache.spark.sql.execution.datasources.hbase")
.load()
}

//Simple method to read from the table
//val df = withCatalog(mycatalog)
//df.registerTempTable("mytempsparktable")
//spark.sql("select id, completed from mytempsparktable").show

val test = new myRecord("12345","1")
//That's the kinda thing an idiot would have on his luggage!
println(test.id)

val data = Array(test)


val sc = spark.sparkContext
sc.parallelize(data).toDF.write.options(Map(HBaseTableCatalog.tableCatalog -> mycatalog, HBaseTableCatalog.newTable -> "5")).format("org.apache.spark.sql.execution.datasources.hbase").save()


//Filtered by rowkey show both
val df1 = withCatalog(mycatalog)

df1.filter($"simid" === "12345").select($"simid", $"completed").show
{% endhighlight %}


