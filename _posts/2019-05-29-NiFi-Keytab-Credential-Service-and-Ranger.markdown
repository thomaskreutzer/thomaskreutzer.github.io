---
layout: post
title:  "NiFi Keytab Credential Service and Ranger"
date:   2019-05-29 08:27:29 -0400
categories: NiFi security
author: Thomas Kreutzer
---
### About the keytab credential service:

Provides a mechanism for specifying a Keytab and a Principal that other components are able to use in order to perform authentication using Kerberos. By encapsulating this information into a Controller Service and allowing other components to make use of it (as opposed to specifying the principal and keytab directly in the processor) an administrative is able to choose which users are allowed to use which keytabs and principals. This provides a more robust security model for multi-tenant use cases.

### Configure the service

Log into the NiFi UI and click on the **Configuration** icon.

![Configuration](https://i.imgur.com/Ai3ifTb.png)

This will open up a new panel for **NiFi Flow Configuration** and click on **CONTROLLER SERVICES**, this will display all the controller services defined at this level. **NOTE:** this is currently being done at the root canvas **Nifi Flow**. 

![Controller Service](https://i.imgur.com/F57hgLa.png)

On the far right hand side there will be a + sign, click this to create a new controller service. 

![New Controller Service](https://i.imgur.com/6BpfFNs.png)

In the filter type the word keytab, then hit enter or click **ADD**. 

![Filter](https://i.imgur.com/9oQCJuD.png)

This has added the service, notice the caution icon as we have not yet configured this properly. 

![Keytab Credential Service](https://i.imgur.com/Anx1xwk.png)

On the far right hand side click on the **Configure** icon. 

![Properties](https://i.imgur.com/DRH0YCL.png)

Notice the name of the service, this is very generic and should be changed to be more specific. Otherwise it starts to become very difficult in NiFi to understand which controller services are for specific purposes. 

![Keytab Credential Service](https://i.imgur.com/vn296cF.png)

Note the unique ID that has been created by NiFi, we will use this later.

![ID](https://i.imgur.com/UFAzITW.png)

Since we are configuring the generic hdpprd-ingest this will be named more appropriately. 

![Controller Service](https://i.imgur.com/YnhgaBy.png)

Next click on **PROPERTIES** to finish the configuration. 
Enter in the path to the keytab file path and the Kerberos Principal. **NOTE:** the keytab should be accessible to the user running NiFi, in this case you should **chown nifi:hadoop /path/to/keytab** and the permissions should be **chmod 600 /path/to/keytab**. 

![Credentials](https://i.imgur.com/lwD4iB6.png)

Click on OK to exit, the controller services is now configured.

![Created Service](https://i.imgur.com/mHTFa4D.png)

### Security Configuration:

Next we will configure a set of policies in Ranger to allow a test of the controller service. This next section assumes that the users already have access to the process group and will not cover these security details. 
In the flow the processor **FetchHBaseRow** requires a keytab, right click on it and click on configure. 

![FetchHbaseRow](https://i.imgur.com/Rfcb6t9.png)

For the **HBase Client Service** we currently have no value set.

![HbaseClientService](https://i.imgur.com/cyKfTm1.png)

When clicking on the dropdown of available services, we can see that no service access has been granted. 

![No Service Available](https://i.imgur.com/5Ml07E5.png)

The **HBase Client Service** will use the keytab, in this case we will create a new service. **NOTE:** This will create a service in the scope of the process group level.
I am renaming my service 

**FROM:**

![From](https://i.imgur.com/2QXymjr.png)

**TO:**

![To](https://i.imgur.com/U7xcxDY.png)

Next click on the arrow to configure, when asked to save changes click yes.  

![Configure](https://i.imgur.com/RpA8kP5.png)

This takes you to the services window, click on the configure icon to edit your controller service and notice the Kerberos Credentials Service is not configured. 

![No Value Set](https://i.imgur.com/ZsuTIi7.png)

Clicking on the drop down displays the fact that a service is configured but not available to my user. NOTE: the id number matches the one previously mentioned in this article. 

![Not Available](https://i.imgur.com/gyP2hCf.png)

In Ranger we will add the following entry to the policy we are configuring for this group. 
**Format Mask** = /controller-services/{controller unique identifier}
**Value** = /controller-services/612a6a17-0168-XXXX-XXXX-XXXXXXX506a0

![Ranger](https://i.imgur.com/v4Itj1D.png)

Once the policies have refreshed we now have access to the service. Pleas select the service. 

![Configured](https://i.imgur.com/ggnEW8A.png)

Configure the rest of the parameters required, in this case for HBase it is the location to the core-site.xml, hdfs-site.xml and hbase-site.xml. 

![Final Configurations](https://i.imgur.com/L6sUwDT.png)

Enable the service and test the processor, and we have success. 

![Success](https://i.imgur.com/1eFE38T.png)

### Other Considerations

If you provide access to a user for the keytab credential service with read/write, they do have full access to the service. This means they can view and change the configuration. If you intend to use this globally as a one-time setup and provide this to process group users it would require you to add this to a policy with read only. There is always still an option for an administrator to set these up at the process group level, however this could also become difficult to manage. Many times groups can just configure these settings within their own process group putting control in the hands of the developers. 