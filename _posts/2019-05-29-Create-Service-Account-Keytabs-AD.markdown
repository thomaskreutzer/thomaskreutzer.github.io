---
layout: post
title:  "Create Service Account Keytabs from Active Directory"
date:   2019-05-29 08:27:29 -0400
categories: hadoop security
author: Thomas Kreutzer
---
### Assumptions:

It is assumed that you have already configured your cluster with a one-way trust to Active Directory and have an MIT kerberos installation in your cluster. It is also assumed that you have installed all kerberos client tools on your linux hosts. 

### Commands to create your keytab:

{% highlight shell %}
ktutil
addent -password service-account@REALM.COM -k 1 -e rc4-hmac
wkt service-account.keytab
quit
{% endhighlight %}

That's it, you know should have a keytab created in the directory you are in with the name service-account.keytab. Of course you should substitute this name and account with the proper name for you configuration.

