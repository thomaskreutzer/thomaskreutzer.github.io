---
layout: post
title:  "Spark Hive Warehouse Connector with Analytical Queries"
date:   2019-11-14 08:27:29 -0400
categories: spark
author: Thomas Kreutzer
---


### Issue:
The hive warehouse connector is very useful to provide Spark with access to hive transactional ACID tables. However, in the current version I was using it did not work with analytical queries unless a specific parameter was added. 

### Resolution:
Under **Custom hiveserver2-interactive-site** add the following property
**hive.llap.external.splits.order.by.force.single.split**=false

Now your queries will proeprly execute. 

{% highlight scala %}
scala> val hiveData = hive.executeQuery(s"""
     |   SELECT DISTINCT
     |     trip_number
     |     , medallion
     |     , pickup_datetime
     |     , dropoff_datetime
     |   FROM (
     |     SELECT
     |       trip_number
     |       ,medallion
     |       ,pickup_datetime
     |       ,dropoff_datetime
     |       ,lead(pickup_datetime) OVER(PARTITION BY medallion ORDER BY pickup_datetime ASC) AS pickup_datetime_lead
     |       ,lead(dropoff_datetime) OVER(PARTITION BY medallion ORDER BY dropoff_datetime ASC) AS dropoff_datetime_lead
     |     FROM trip
     |     ORDER BY pickup_datetime ASC
     |   ) X
     |   WHERE
     |     pickup_datetime_lead NOT BETWEEN pickup_datetime AND dropoff_datetime
     |     AND dropoff_datetime_lead NOT BETWEEN pickup_datetime AND dropoff_datetime
     | """)
hiveData: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] = [trip_number: int, medallion: string ... 2 more fields]
 
scala> hiveData.count()

res0: Long = 6149497
{% endhighlight %}
