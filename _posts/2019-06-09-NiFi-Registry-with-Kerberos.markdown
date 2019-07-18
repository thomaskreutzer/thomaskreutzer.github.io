---
layout: post
title:  "NiFi Registry with Kerberos"
date:   2019-06-09 08:27:29 -0400
categories: NiFi security
author: Thomas Kreutzer
---
### Assumptions:

Many windows environments have Kerberos enabled behind the curtains for you as with the following scenario. In addition we have a one-way trust configured with the Active Directory domain. This article only covers issues I encountered with early versions of NiFi Registry which may be resolved today in a newer version.

### Issue:
![Bad Message](https://i.imgur.com/p0a2rC3.png)


It seems UI is sending more than 8kb of headers size but it may be working fine using curl because its request size did not go beyond 8kb. You may want to increase max request header size to 16kb. You need to update the config in **/var/lib/ambari-server/resources/common-services/REGISTRY/0.3.0/package/templates/registry.yaml.j2**, restart ambari-server and registry. This change should be reflected in **/usr/hdf/current/registry/conf/registry.yaml**.

I am trying 16KiB
{% highlight yaml %}
server:
  applicationConnectors:
    - type: http
      port: <port-no>
      maxRequestHeaderSize: 16KiB
{% endhighlight %}

After restarting NiFi registry the error message bumped up to our new max.
{% highlight none %}
WARN 2018-05-22 08:17:52.493 dw-61 o.e.j.h.HttpParser - Header is too large >16384
WARN 2018-05-22 08:17:52.494 dw-61 o.e.j.h.HttpParser - bad HTTP parsed: 413 for HttpChannelOverHttp@72106d4f
{r=1,c=false,a=IDLE,uri=null}
ERROR 2018-05-22 08:17:52.585 dw-61 - GET /favicon.ico c.h.r.c.GenericExceptionMapper - Got exception: NotFoundException / message HTTP 404 Not Found
javax.ws.rs.NotFoundException: HTTP 404 Not Found
{% endhighlight %}

### SOLUTION:
After updating it to 32KiB, I am able to get past the error.

