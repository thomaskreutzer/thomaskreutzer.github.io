---
layout: post
title:  "Connecting to Hive through Knox using PowerShell"
date:   2019-07-18 08:27:29 -0400
categories: hdp hive
author: Thomas Kreutzer
---
### Goal:

Connect to Hive with Knox and Powershell

### Steps:

1. Install the ODBC Driver

2. Execute a test in PowerShell
{% highlight shell %}
$s_ID = "youruser"
$s_PWD = "yourpassword"
 
$conn_string = "DRIVER={Hortonworks Hive ODBC Driver};ThriftTransport=2;SSL=1;Host=yourhost.com;Port=8443;Schema=yourdb;HiveServerType=2;AuthMech=3;HTTPPath=/gateway/default/llap;CAIssuedCertNamesMismatch=1;AllowSelfSignedServerCert=1;SSP_mapreduce.job.queuename=somequeue;SSP_tez.queue.name=somequeue;UseNativeQuery=1;UID=" + $s_ID + ";PWD=" + $s_PWD + ";"
$conn = New-Object System.Data.Odbc.OdbcConnection
 
$sql = "Show Tables" 
 
$conn.ConnectionString = $conn_string
$conn.Open()
$cmd = New-Object System.Data.Odbc.OdbcCommand
$cmd.Connection = $conn
$cmd.CommandText = $sql
$execute = $cmd.ExecuteReader()
 
while ( $execute.read() )
{
    $execute.GetValue(0)
}
{% endhighlight %}




