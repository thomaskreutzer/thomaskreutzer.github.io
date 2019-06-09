---
layout: post
title:  "Kerberos and Firefox"
date:   2019-06-09 07:27:29 -0400
categories: security
author: Thomas Kreutzer
---
### Configuration:

Install the MIT Kerberos tools for windows and get a proper ticket. [MIT Kereberos](http://web.mit.edu/kerberos/dist/)
In the firefox URL type about:config

![about:config](https://i.imgur.com/y2e7htw.png)

Click on accept the risk that you might void your warranty. 

![Warranty](https://i.imgur.com/c2ffs7Z.png)

Search on **negotiate-auth**
Under **network.negotiate-auth.delegation-uris** and **network.negotiate-auth.trusted-uris** we added **yourdomain.com ,.yourdomain.com**

![about:config](https://i.imgur.com/3iFLrIk.png)

Search on **network.auth.use-sspi** and set it to **false**:

![about:config](https://i.imgur.com/S23LpvW.png)

Search on **network.negotiate-auth.gsslib** and set it to the value for the MIT Kerberos you installed, in our case it was 64 bit with Firefox 64 bit so we set it to **C:\Program Files\MIT\Kerberos\bin\gssapi64.dll**:

![about:config](https://i.imgur.com/yN1qKZM.png)

After this we had successfully configured firefox for Kerberos.