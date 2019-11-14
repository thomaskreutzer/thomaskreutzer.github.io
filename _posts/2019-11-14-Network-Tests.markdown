---
layout: post
title:  "Network Tests"
date:   2019-11-14 07:20:29 -0400
categories: network
author: Thomas Kreutzer
---
### Description:
Every once in a while you will run into an issue and the support team is going to ask for a set of network tests. Due dilligence is required because network problems can compound very quickly. Here is a quick script you can run to create some logs for testing. 



{% highlight bash %}
#!/bin/bash

#cleanup any prior files
rm /tmp/*_ifconfig.log
rm /tmp/*.yourdomain.com.log

# Declare an array of string with type

declare -a StringArray=("hst01.yourdomain.com" \
"hst02.yourdomain.com" \
"hst03.yourdomain.com" \
"hst04.yourdomain.com" \
"hst05.yourdomain.com" \
"hst06.yourdomain.com" \
"hst07.yourdomain.com" \
"hst08.yourdomain.com" ) 

# Iterate the string array using for loop

for val in ${StringArray[@]}; do
   echo $val
   ping -c 3600 $val >> /tmp/from_${HOSTNAME}_to_${val}.log
   dig $val >> /tmp/$from_${HOSTNAME}_to_${val}.log
done
 
# Network interfaces
ifconfig >> /tmp/${HOSTNAME}_ifconfig.log
{% endhighlight %}


Once you have completed execution of these scripts on every host, collect the logs into one location Search through the ping logs to see if you have times exceeding 1ms. Ping times generally should not exceed 1ms, especially if they are located on the same network or rack. 

{% highlight bash %}
grep "time=[1-9]" ./*
grep "time=[1-9][0-9]" ./*
{% endhighlight %}

You can also check the dig to see if the hosts have a single A record pointing to a single IP Address. You could use nslookup to execute forward and reverse lookups. 

{% highlight bash %}
grep ";; ANSWER SECTION:" -A 2 ./*
{% endhighlight %}

