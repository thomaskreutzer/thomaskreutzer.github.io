---
layout: post
title:  "Spark Delta's with Hive"
date:   2019-07-15 08:27:29 -0400
categories: spark
author: Thomas Kreutzer
---



### Project information:

This project was created to meet very specific requirements for a specific scenario and set of requirements. A delta's was to be derived by comparing two tables on a daily basis for the delta to create a historical representation of the data. In most cases the reporting only executes against the daily snapshot but in some cases history may be desired. This process was created to compare the previous days snapshot to the current day, the resulting delta is used in a merge statement to update history. Spark was used in order to get the detla information between the two tables. The delta processing requires use to also keep history of when a record is deleted and re-added. 

**Disclaimer:** There could be other approaches to solve this issue, however due to limitations of time, resources and other factors this direction was chosen. One approach that was suggested would be to use Hive to process the delta instead of spark in some form. I would recommend attempting and testing this approach in your project to see if fewer steps could be executed in order to achieve the goal in a faster time frame. If time permits I will examine this one my own to understand the differences. 

**Project Goals:**
1. A method to store the composite key information for each table
2. Scala code that is re-usable for all tables
3. Compare and get the delta
4. Merge into history **(Will not laid out in major detail, simple SQL example)**


**Composite Key:**
As it turns out, all of the tables in this off load had the primary/composite keys as the first columns in the table. It was determined a table would be created to store the table name and offset / number of keys in the table. This allows us to very easily use generic code in order to process the tables in Scala using Spark. Another thing to note in this case is that all of the tables are partitioned. This is considered to be part of the key and is added to the offset. 


### Scala Code Overview:
The code is procedural in nature so we will not be creating many classes, we will simply do our work in the main function that will be called. First we will add our package name, imports that we will be using and the main class. 

{% highlight scala %}
package com.yourdomain.delta_scala

import scala.collection.mutable.ArrayBuffer
import org.apache.spark.sql.functions.unix_timestamp
import org.apache.spark.sql.functions._
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.hash

object App {
  def main(args : Array[String]) {
}
{% endhighlight %}



All parameters will be passed in at execution time to the application for execution of a specific table. We will set up the params and initialize all default variables. 

**Note:** the previous day and delta table are always the table name with a suffix appended. It is also assumed here that the previous days delta is in the location and is handled with a flow that will not be described here.






{% highlight scala %}
/*
* EXAMPLE:
* /usr/hdp/2.6.4.0-91/spark2/bin/spark-submit
* --master yarn
* --driver-memory 2g
* --num-executors 5
* --executor-memory 2g
* --queue yourqueue \
* --class com.yourdomain.delta_scala.App \
* /tmp/x003075/delta-scala-0.0.1-snapshot.jar \
* spark_delta \  --Application name we are executing
* someStageDb \  --Stage database
* someTargetDb \ --Target Database
* someTableName \  --The name of the table you are processing (target)
* /scripts/hive_hql/  --The location of the HQL's you intend to execute. 
*
* /usr/hdp/2.6.4.0-91/spark2/bin/spark-submit --master yarn --driver-memory 2g --num-executors 5 --executor-memory 2g --queue yourqueue --class com.yourclass.delta_scala.App /tmp/delta-scala-0.0.1-snapshot.jar spark_delta stageDb  targetDb tablename /scripts/hive_hql/
*/

val appName = args(3) + "_" + args(0) //Table name and app name concatenated to make unique
val stageDb = args(1)
val targetDb = args(2)
val tableName = args(3)
val hdfsPath = "hdfs://" + args(4) /*This path is now the source location for HQL*/
    
val prevDayTbl = tableName + "_prev" /*The previous day table*/
val deltaTbl = tableName + "_ct" /*The detla table*/
    
/* Create the spark session */
val spark = SparkSession
  .builder()
  .appName(appName)
  .enableHiveSupport()
  .getOrCreate()
{% endhighlight %}






The next snippet of code is setting the variable f = to a path in **hdfs** for our select SQL followed by select_hql that is reading in the contents of the file. 
f2 is setting the variable = to a path in **hdfs** for the insert to our delta table and setting insert_hql equal the contents of the file while parsing out the database name with the stageDb.deltaTbl

{% highlight scala %}
/* Get the contents from HDFS */
val f = spark.sparkContext.wholeTextFiles(hdfsPath + "spark_select/" + tableName + ".sql")
val select_hql = f.take(1)(0)._2

/* Get the contents from HDFS and parse out text*/
val f2 = spark.sparkContext.wholeTextFiles(hdfsPath + "ins_ct/" + tableName + ".sql")
val insert_hql = f2.take(1)(0)._2.replace("__TARGET__", stageDb + "." + deltaTbl)
     
/* Use the select to set two SQL's we will be using later */
val select_prev = select_hql.replace("__SOURCE__", stageDb + "." + prevDayTbl)
val select_curr = select_hql.replace("__SOURCE__", targetDb + "." + tableName)
{% endhighlight %}






In the following snippet we will create a data from for the previous, current and our key tables. This is followed by two statements that get the columns from the data frames. They key count is the offset of our primary/composite key. 
**NOTE:** I am using the upper clause here in order to ensure that we match the case for the table name regardless. If I am not mistaken we saw an issue when someone put in a different case. 

{% highlight scala %}
val df_prev = spark.sql(select_prev)
val df_curr = spark.sql(select_curr)
val df_key_table = spark.sql("SELECT key_cnt FROM " + stageDb + ".etl_key_count WHERE upper(table_name) = '" + tableName.toUpperCase() + "'")
 
val df_prev_table_columns = df_prev.columns.toSeq
val df_curr_table_columns = df_curr.columns.toSeq
{% endhighlight %}





Now we will pull the key count from the table followed by another variable **key_columns** that will contain all of the keys for the table. 
{% highlight scala %}
val key_count = df_key_table.select(col("key_cnt")).first.getInt(0)
val key_columns = df_curr_table_columns.slice(0,key_count)
{% endhighlight %}




Next we want to create two arrays. 
1. An array of key columns
2. An array of all columns

{% highlight scala %}
val all_columns = df_curr_table_columns.slice(0,df_curr_table_columns.size)

*/Create all columns array*/
val all_cols_array = ArrayBuffer[String]()
for (v <- all_columns) all_cols_array += v

/*Create key columns array*/
val key_cols_array = ArrayBuffer[String]()
for (v <- key_columns) key_cols_array += v
{% endhighlight %}




Next we are going to concat all of the columns together for the keys and non-key columns independently. These will be used to execute a comparison. For performance purposes, we will also convert the concatenated data for the non-key values into a hash. You can choose to add this for the key columns if required. In most cases our concatenation of key columns is rather small and numeric so we have not hashed them. 

{% highlight scala %}
var df_prev_key_columns = df_prev.withColumn( "df_prev_table_key_columns_concat", concat_ws("", key_cols_array.map(c => col(c)): _*) )
var df_prev_all_columns = df_prev_key_columns.withColumn( "df_prev_table_columns_concat", hash(concat_ws("", all_cols_array.map(c => col(c)): _*)) )

var df_curr_key_columns = df_curr.withColumn( "df_curr_table_key_columns_concat", concat_ws("", key_cols_array.map(c => col(c)): _*) )
var df_curr_all_columns = df_curr_key_columns.withColumn( "df_curr_table_columns_concat", hash(concat_ws("", all_cols_array.map(c => col(c)): _*)) )

/* Create views */
df_prev_all_columns.createOrReplaceTempView("prev")
df_curr_all_columns.createOrReplaceTempView("curr")
{% endhighlight %}




We have collected the information required to create data frames for each of the delta's
1. Deleted
2. Inserted
3. Updated

**NOTE** we are adding a timestamp to each of the quiries to specifiy the time in when the etl transaction takes place. 
Deleted records also get their own flag, in essence we are doing soft deletes using this flag. 


{% highlight scala %}
val df_deleted  = spark.sql("SELECT current_timestamp() as etl_ld_dttm, 'D' as header__change_oper,1 as header__deleted, p.* FROM prev p LEFT OUTER JOIN curr c ON p.df_prev_table_key_columns_concat = c.df_curr_table_key_columns_concat WHERE df_curr_table_key_columns_concat IS NULL").drop("df_prev_table_key_columns_concat", "df_prev_table_columns_concat")

val df_inserted = spark.sql("SELECT current_timestamp() as etl_ld_dttm, 'I' as header__change_oper,0 as header__deleted, c.* FROM prev p RIGHT OUTER JOIN curr c ON p.df_prev_table_key_columns_concat = c.df_curr_table_key_columns_concat WHERE df_prev_table_key_columns_concat IS NULL").drop("df_curr_table_key_columns_concat", "df_curr_table_columns_concat")

val df_modified = spark.sql("SELECT current_timestamp() as etl_ld_dttm, 'U' as header__change_oper,0 as header__deleted, c.* FROM prev p INNER JOIN curr c ON p.df_prev_table_key_columns_concat = c.df_curr_table_key_columns_concat AND p.df_prev_table_columns_concat <> c.df_curr_table_columns_concat").drop("df_curr_table_key_columns_concat", "df_curr_table_columns_concat")
{% endhighlight %}







**NOTE: Then next snippet is informational only and is not used in this project**

Another way to do it similar to what you see above is with pure Scala code in place of the spark sql. Depending on your preference one may be easier than the other. I personally prefer SQL. 

{% highlight scala %}
val df_modified = df_prev_all_columns.join(df_curr_all_columns.as("df_curr_all_columns"), (df_curr_all_columns("df_curr_table_key_columns_concat") === df_prev_all_columns("df_prev_table_key_columns_concat")) && (df_curr_all_columns("df_curr_table_columns_concat") !== df_prev_all_columns("df_prev_table_columns_concat")) )
{% endhighlight %}




All of the results are concatenated into a single dataframe.

{% highlight scala %}
val unioned_df = df_inserted.union(df_deleted).union(df_modified)
unioned_df.createOrReplaceTempView("ins_data_df")
{% endhighlight %}

Finally we execute a purge from the hive stage table where the data is stored for the merge. 
{% highlight scala %}
spark.sql("set hive.exec.dynamic.partition.mode=nonstrict")
spark.sql("TRUNCATE TABLE " + stageDb + "." + deltaTbl)
spark.sql(insert_hql)
{% endhighlight %}



### SQL information:

**SQL for select example:**
{% highlight sql %}
SELECT
  partition_col
  ,`key_col_1`
  ,`key_col_2`
  ,`non_key_col_1`
  ,`non_key_col_2`
  ,`non_key_col_3`
  ,`non_key_col_4`
  ,`non_key_col_5`
  ,`non_key_col_6`
  ,`non_key_col_7`
  ,`non_key_col_8`
  ,`non_key_col_9`
  ,`non_key_col_10`
FROM __SOURCE__
{% endhighlight %}

**SQL for merge example:**
{% highlight sql %}
SET tez.queue.name=production;
SET hive.tez.container.size=6168;
SET hive.tez.java.opts=-Xmx4934m;
SET hive.merge.cardinality.check=false;
SET hive.vectorized.execution.enabled=true;
SET hive.vectorized.execution.reduce.enabled=true;
SET hive.cbo.enable=true;
SET hive.compute.query.using.stats=true;
SET hive.stats.fetch.column.stats=true;
SET hive.stats.fetch.partition.stats=true;
SET mapred.reduce.tasks=20;

MERGE INTO db_hist.target_table AS t
USING
(
  SELECT
    1 AS END_DATE, target_table_ct.*
  FROM stageDb.target_table_ct
  WHERE
    header__change_oper IN ('U', 'D')
  UNION ALL
  SELECT
    0 AS END_DATE, target_table_ct.*
  FROM stageDb.target_table_ct
  WHERE
    header__change_oper IN ('U', 'I')
) AS s
ON
  t.partition_col = s.partition_col
  AND t.`key_col_1` = s.`key_col_1`
  AND t.`key_col_2` = s.`key_col_2`
  AND s.END_DATE = 1
WHEN MATCHED
  AND t.header__to_date = CAST(CAST('9999-12-31' AS DATE) AS TIMESTAMP)
  AND t.etl_ld_dttm <> s.etl_ld_dttm
THEN UPDATE
SET
  header__to_date = current_timestamp(),
  header__deleted = s.header__deleted
WHEN NOT MATCHED
  AND s.END_DATE = 0
THEN INSERT VALUES (
  s.`etl_ld_dttm`
  ,current_timestamp()
  ,CAST(CAST('9999-12-31' AS DATE) AS TIMESTAMP)
  ,0
  ,s.`key_col_1`
  ,s.`key_col_2`
  ,s.`non_key_col_1`
  ,s.`non_key_col_2`
  ,s.`non_key_col_4`
  ,s.`non_key_col_5`
  ,s.`non_key_col_6`
  ,s.`non_key_col_7`
  ,s.`non_key_col_8`
  ,s.`non_key_col_9`
  ,s.`non_key_col_10`
  ,s.`part_col`
);
{% endhighlight %}