---
layout: post
title:  "NiFi Two-Way TLS/SSL with Powershell and Windows"
date:   2019-05-24 15:57:29 -0400
categories: nifi security
author: Thomas Kreutzer
---
Recently I worked on a project were a Windows platform needed to communicate to a Restful API hosted on NiFi. Part of the security requirements involved implementation of two-way or mutual TLS and the application running on windows was restricted to using Powershell. This blog article assumes you have basic knowledge with configurations for JAVA keystores, truststores and creation of certificates, they will not be covered in detail. I will cover how Powersehll was able to connect and send secured requests to our target application. It is assumed the client's certifi

In this example I was provided a keystore from the client server, from this I will export the required formats we will use for our configurations. 

### Convert .JKS into .p12 and .crt:

{% highlight shell %}
keytool -importkeystore -srckeystore mykeystore.jks -destkeystore client.p12 -srcalias youralias -
srcstoretype jks -deststoretype pkcs12
openssl pkcs12 -in client.p12 -out client.pem
openssl x509 -outform der -in client.pem -out client.crt
{% endhighlight %}

### Import the .p12 file into windows certificate store. 
On your windows machine click search and type in cert
Click on Manage user certificates

![Certificate Manager](https://i.imgur.com/5fhbtrY.png)

This opens up the certificate manager, click on **Personal --> Certificates**

![Personal Certificate](https://i.imgur.com/A4OGXx9.png)

With Certificates highlighted click on **Action --> All Tasks --> Import..**

![All Tasks](https://i.imgur.com/JBi9bJr.png)

The next screen will have Current User defaulted, it cannot be changed as we are editing for current user. Just click **Next**.

![Certificate Import](https://i.imgur.com/iquzNhZ.png)

On the next screen click browse, you will select your .p12 file that was created in the earlier steps. In order to do so you must also change the filter so that it shows files with the .p12 extension. 

![Browse File](https://i.imgur.com/LExId7X.png)
![iFile Format](https://i.imgur.com/VMEqOOv.png)

With the proper **.p12** file selected, click next. 

![File Selected]https://i.imgur.com/m7VFJ8p.png)

On the following screen enter the password and click next.

![Password](https://i.imgur.com/I2l3Y6t.png)

On the following screen accept the defaults by clicking next.

![Accept Defaults](https://i.imgur.com/3rpAouG.png)


Click finish on the last screen

![Finish](https://i.imgur.com/vJkS9jd.png)


Your .p12 certificate is installed. 

![p12 Installed](https://i.imgur.com/tj0rZyA.png)



## NiFi Configuration:
The NiFi configuration of the processor assumes you have a properly set up truststore with the certificate the client will be sending installed. We are using a StandardRestrictedSSLContextService defined as follows. 

![StandardRestrictedSSLContextService](https://i.imgur.com/WFinG4G.png)


We also use a StandardHTTPContextMap in NiFi with the following default configurations. 
![StandardHTTPContextMap](https://i.imgur.com/4BzkVwd.png)

Finally we have the HandleHTTPRequest processor with a port that is not being used, in our case 9092 for this test. We configured our Rest Endpoint and finally we have selected Need Authentication which will required two-way SSL. 

![StandardHTTPContextMap](https://i.imgur.com/ZikLem0.png)


## Execution of powershell:
We have two different ways that we can execute the powershell command. The first method is to use the .crt file which would need to be deployed on the Windows machine. The first method is as follows. 

{% highlight shell %}
$Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2
$Cert.Import("D:\path\to\tls\client.crt")

[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest `
  -UseBasicParsing https://yourhost:9092/endpoint/hbase `
  -ContentType "application/json" `
  -Method POST -Body "{'name':'Thomas','email':'none@gmail.com'}" `
  -Certificate $Cert
{% endhighlight %}

The second method is to use the thumbprint, first we will demo how to get the thumbprint in powershell. 

{% highlight shell %}
$Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2
$Cert.Import("D:\path\to\tls\client.crt")
Write-Output "$cert"
{% endhighlight %}


![Thumbprint Display](https://i.imgur.com/MIlrzYt.png)


Using the thumbprint in your powershell command.

{% highlight shell %}
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest `
  -UseBasicParsing https://yourhost:9092/endpoint/hbase `
  -ContentType "application/json" `
  -Method POST -Body "{'name':'Thomas','email':'none@gmail.com'}" `
  -CertificateThumbprint 19XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
{% endhighlight %}


You can also get the thumbpring and pass it right from the cert. 

{% highlight shell %}
$Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2
$Cert.Import("D:\path\to\tls\client.crt")

[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest `
  -UseBasicParsing https://yourhost:9092/endpoint/hbase `
  -ContentType "application/json" `
  -Method POST -Body "{'name':'Thomas','email':'none@gmail.com'}" `
  -CertificateThumbprint $($Cert.thumbprint)
{% endhighlight %}

All of the above methods will give you success. 

![200 OK](https://i.imgur.com/KzIzoBz.png)


You can also see the file has gone through NiFi.

![Nifi Success](https://i.imgur.com/d3hNa6L.png)


