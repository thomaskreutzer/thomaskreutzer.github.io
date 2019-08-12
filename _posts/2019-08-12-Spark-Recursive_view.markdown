---
layout: post
title:  "Spark Recursive View"
date:   2019-08-12 08:27:29 -0400
categories: spark
author: Thomas Kreutzer
---
### Goal:

Hive does not have recursive views, Teradata conversions could run into issues in this area. Especially if you are working on the dreaded lift and shift projects. I found an article (https://www.pythian.com/blog/recursion-in-hive/) on methods to replicate recursive views using spark and I used this to load data into a table on ingest. This is a bit more complex of a case than existed in the original tutorial. 


### Example:
in this case we called scala code dynamically passing in an ID to query and create some recursive data. 

{% highlight scala %}
val appName = "some_name"
val appID = __APPID__
val tgtDb = "mytargetdb"
val srcTable = "mysourcetable"
val tgtTable = "mytargettable"

//SQL to get seed data
val df_list_seed = spark.sql(s"""
SELECT
  *
  ,0 AS cnt
  ,ROW_NUMBER() OVER (PARTITION BY appid, col1, col2, col3, cole4, col5 ORDER BY col6) AS sequence
FROM
  $tgtDb.$srcTable
WHERE appid = $appID
""");

//Register the dataframe as a temp table to be used in the net step for iteration.
df_list_seed.createOrReplaceTempView("vt_seed0")

//Hold the value of the number of rows in the new dataset
var df_cnt:Int=1

//Iteration Counter
var cnt:Int=1

//Use the following while loop to generate a new data frame for each run.
//We have generated a new data frame with a sequence. At each step, the previous data frame is used to retrieve a new resultset.
//If the data frame does not have any rows, then the loop is terminated.
//Same query from "Iteration" statement issues here too
//Also only register a temporary table if the data frame has rows in it, hence the "IF" condition is present in WHILE loop.
//Also note, col6 is the one we will use to build hierarchy information. 

while (df_cnt !=0) {
  var tblnm = "vt_seed".concat((cnt-1),toString);
  var tblnm1 = "vt_seed".concat((cnt).toString);
  val df_list_rec = spark.sql("""
    SELECT
      concat_ws(',', tb2.col6, tbl1.col6) AS col6
      ,tbl1.sequence
      ,tbl1.appid
      ,tbl1.col1
      ,tbl1.col2
      ,tbl1.col3
      ,tbl1.col4
      ,tbl1.col5
      ,$cnt AS cnt
    FROM
      vt_seed0 tb1
      ,$tblnm  tb2
    WHERE
      tb2.sequence+1=tbl1.sequence
      AND tb2.col1 = tbl1.col1
      AND tb2.col2 = tbl1.col2
      AND tb2.col3 = tbl1.col3
      AND tb2.col4 = tbl1.col4
      AND tb2.col5 = tbl1.col5
  """);
  df_cnt=df_list_rec.count.toInt;
  if(df_cnt!=0){
    df_list_rec.createOrReplaceTempView(s"$tblnm1");
  }
  cnt=cnt+1;
}


//UNION all of the tables together
var union_query = "";
for ( a <- 0 to (cnt -2)){
  if(a ==0 ){
    union_query = union_query.concat("SELECT col6, sequence, appid, col1, col2, col3, col4, col5, cnt FROM vt_seed").concat(a.toString());
  }
  else {
    union_query = union_query.concat("UNION SELECT col6, sequence, appid, col1, col2, col3, col4, col5, cnt FROM vt_seed").concat(a.toString());
  }
}


//Create the dataframe with the SQL
val df_union = spark.sql(s"$union_query")
df_union.createOrReplaceTempView("vt_union")


//Execute the analytical query to extract only the max iteration based on partition
//We want only the max sequence from each set in the result

spark.sql("set hive.exec.dynamic.partition.mode=nonstrict")

val df_maxrecord = spark.sql(s"""
  SELECT
    col1
    ,col2
    ,col3
    ,col4
    ,col5
    ,col6
    ,appid
  FROM
  (
    SELECT
      col1
      ,col2
      ,col3
      ,col4
      ,col5
      ,col6
      ,appid
      ,MAX(sequence) over (PARTITION BY appid, col1, col2, col3, col4, col5) max_sequence
    FROM
    (
      SELECT
        col1
        ,col2
        ,col3
        ,col4
        ,col5
        ,col6
        ,appid
        ,cnt
        ,MAX(cnt) OVER (PARTITION BY sequence, appid, col1, col2, col3, col4, col5) max_cnt
      FROM vt_union
    )
    WHERE
      cnt = max_cnt
  )
  WHERE
   sequence = max_sequence
  DISTRIBUTE BY appid
""")
df_maxrecord.createOrReplaceTempView("vt_maxrecord)

//Finally insert the result to your target table
val df_materialize = spark.sql(s"""
  INSERT OVERWRITE TABLE $tgtDb.tgtTable PARTITION (appid)
  SELECT * FROM vt_maxrecord
""")

{% endhighlight %}