---
layout: post
title:  "Calling Kerberized RESTful API from Spark with Scala"
date:   2019-11-08 04:45:29 -0400
categories: spark
author: Thomas Kreutzer
---
### Description:
Recently I worked on a project where it was required to call restful API's with a spark application. The endpoint that we needed to access was using Kerberos Authentication. To make this work properly I deployed a service keytab for the application ID that would be executing the process, in addition JAVA code was created to allow get and put operations to succeed with Kerberos. The after the jar was imported into the program kerberos calls worked perfeclty. This is probably not the cleanest implementation and it was completed in a pinch. 


**Java Code:**
{% highlight java %}
package com.yoursite.scala.rest;

import java.io.IOException;
import java.io.InputStream;
import java.security.Principal;
import java.security.PrivilegedAction;
import java.util.Arrays;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Set;

import javax.security.auth.Subject;
import javax.security.auth.kerberos.KerberosPrincipal;
import javax.security.auth.login.AppConfigurationEntry;
import javax.security.auth.login.Configuration;
import javax.security.auth.login.LoginContext;

import org.apache.commons.io.IOUtils;
import org.apache.http.HttpException;
import org.apache.http.HttpResponse;
import org.apache.http.StatusLine;
import org.apache.http.auth.AuthSchemeProvider;
import org.apache.http.auth.AuthScope;
import org.apache.http.auth.Credentials;
import org.apache.http.client.HttpClient;
import org.apache.http.client.HttpResponseException;
import org.apache.http.client.config.AuthSchemes;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.client.methods.HttpUriRequest;
import org.apache.http.config.Lookup;
import org.apache.http.config.RegistryBuilder;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.auth.SPNegoSchemeFactory;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClientBuilder;

public class KerberosHttpClient {
	private String principal;
	private String keyTabLocation;

	public KerberosHttpClient() {
	}

	public KerberosHttpClient(String principal, String keyTabLocation) {
		super();
		this.principal = principal;
		this.keyTabLocation = keyTabLocation;
	}

	public KerberosHttpClient(String principal, String keyTabLocation, String krb5Location) {
		this(principal, keyTabLocation);
		System.setProperty("java.security.krb5.conf", krb5Location);
	}

	public KerberosHttpClient(String principal, String keyTabLocation, boolean isDebug) {
		this(principal, keyTabLocation);
		if (isDebug) {
			System.setProperty("sun.security.spnego.debug", "true");
			System.setProperty("sun.security.krb5.debug", "true");
		}
	}

	public KerberosHttpClient(String principal, String keyTabLocation, String krb5Location, boolean isDebug) {
		this(principal, keyTabLocation, isDebug);
		System.setProperty("java.security.krb5.conf", krb5Location);
	}

	private static HttpClient buildSpengoHttpClient() {
		HttpClientBuilder builder = HttpClientBuilder.create();
		Lookup<AuthSchemeProvider> authSchemeRegistry = RegistryBuilder.<AuthSchemeProvider>create()
				.register(AuthSchemes.SPNEGO, new SPNegoSchemeFactory(true)).build();
		builder.setDefaultAuthSchemeRegistry(authSchemeRegistry);
		BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
		credentialsProvider.setCredentials(new AuthScope(null, -1, null), new Credentials() {
			@Override
			public Principal getUserPrincipal() {
				return null;
			}

			@Override
			public String getPassword() {
				return null;
			}
		});
		builder.setDefaultCredentialsProvider(credentialsProvider);
		CloseableHttpClient httpClient = builder.build();
		return httpClient;
	}
	
	private Subject getSubject(String userId) {
		Set<Principal> princ = new HashSet<Principal>(1);
		princ.add(new KerberosPrincipal(userId));
		Subject sub = new Subject(false, princ, new HashSet<Object>(), new HashSet<Object>());
		return sub;
	}
	
	
	private Configuration getConfig() {
		Configuration config = new Configuration() {
			@SuppressWarnings("serial")
			@Override
			public AppConfigurationEntry[] getAppConfigurationEntry(String name) {
				return new AppConfigurationEntry[] {
						new AppConfigurationEntry("com.sun.security.auth.module.Krb5LoginModule",
								AppConfigurationEntry.LoginModuleControlFlag.REQUIRED, new HashMap<String, Object>() {
									{
										put("useTicketCache", "false");
										put("useKeyTab", "true");
										put("keyTab", keyTabLocation);
										// Krb5 in GSS API needs to be refreshed so it does not throw the error
										// Specified version of key is not available
										put("refreshKrb5Config", "true");
										put("principal", principal);
										put("storeKey", "true");
										put("doNotPrompt", "true");
										put("isInitiator", "true");
										put("debug", "true");
									}
								}) };
			}
		};
		
		return config;
	}

	private HttpResponse httpResponseGet(final String url) {
		// keyTabLocation = keyTabLocation.substring("file://".length());
		System.out.println(String.format("Calling KerberosHttpClient %s %s %s", this.principal, this.keyTabLocation, url));
		
		Configuration config = getConfig();
		Subject sub=getSubject(this.principal);
		
		try {
			LoginContext lc = new LoginContext("", sub, null, config);
			lc.login();
			Subject serviceSubject = lc.getSubject();
			return Subject.doAs(serviceSubject, new PrivilegedAction<HttpResponse>() {
				HttpResponse httpResponse = null;
				@Override
				public HttpResponse run() {
					try {
						HttpUriRequest request = new HttpGet(url);
						HttpClient spnegoHttpClient = buildSpengoHttpClient();
						httpResponse = spnegoHttpClient.execute(request);
						return httpResponse;
					} catch (IOException ioe) {
						ioe.printStackTrace();
					}
					return httpResponse;
				}
			});
		} catch (Exception le) {
			le.printStackTrace();
			;
		}
		return null;
	}
	
	private HttpResponse httpResponsePost(final String url, String json) {
		// keyTabLocation = keyTabLocation.substring("file://".length());
		System.out.println(String.format("Calling KerberosHttpClient %s %s %s", this.principal, this.keyTabLocation, url));
		
		Configuration config = getConfig();
		Subject sub=getSubject(this.principal);
		
		try {
			LoginContext lc = new LoginContext("", sub, null, config);
			lc.login();
			Subject serviceSubject = lc.getSubject();
			return Subject.doAs(serviceSubject, new PrivilegedAction<HttpResponse>() {
				HttpResponse httpResponse = null;
				@Override
				public HttpResponse run() {
					try {
						HttpPost request = new HttpPost(url);
						
						StringEntity entity = new StringEntity(json);
						request.setEntity(entity);
						request.setHeader("Accept", "application/json");
						request.setHeader("Content-type", "application/json");
						HttpClient spnegoHttpClient = buildSpengoHttpClient();
						httpResponse = spnegoHttpClient.execute(request);
						return httpResponse;
					} catch (IOException ioe) {
						ioe.printStackTrace();
					}
					return httpResponse;
				}
			});
		} catch (Exception le) {
			le.printStackTrace();
			;
		}
		return null;
	}
	
	
	public String getUrl(String principal, String keytab, String url) throws UnsupportedOperationException, IOException, HttpException{
		String c = "";
		KerberosHttpClient restTest = new KerberosHttpClient(principal,keytab,true);
		HttpResponse response = restTest.httpResponseGet(url);
		InputStream is = response.getEntity().getContent();
		StatusLine statusLine = response.getStatusLine();
		if(statusLine.getStatusCode() == 200) {
			c = new String(IOUtils.toByteArray(is), "UTF-8");
		} else {
			throw new HttpResponseException(statusLine.getStatusCode(), statusLine.getReasonPhrase());
		}
		
		return c;
	}
	
	public String postUrl(String principal, String keytab, String url, String json) throws UnsupportedOperationException, IOException, HttpException{
		String c = "";
		KerberosHttpClient restTest = new KerberosHttpClient(principal,keytab,true);
		HttpResponse response = restTest.httpResponsePost(url, json);
		InputStream is = response.getEntity().getContent();
		StatusLine statusLine = response.getStatusLine();
		if(statusLine.getStatusCode() == 201) {
			c = new String(IOUtils.toByteArray(is), "UTF-8");
		} else {
			throw new HttpResponseException(statusLine.getStatusCode(), statusLine.getReasonPhrase());
		}
		
		return c;
	}
}

{% endhighlight %}

**pom.xml**
{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.yoursite.kerberos</groupId>
	<artifactId>Kerberos</artifactId>
	<version>0.0.1-SNAPSHOT</version>



	<dependencies>
		<!-- https://mvnrepository.com/artifact/org.apache.httpcomponents/httpclient -->
		<dependency>
			<groupId>org.apache.httpcomponents</groupId>
			<artifactId>httpclient</artifactId>
			<version>4.5.10</version>
		</dependency>

		<!-- https://mvnrepository.com/artifact/commons-io/commons-io -->
		<dependency>
			<groupId>commons-io</groupId>
			<artifactId>commons-io</artifactId>
			<version>2.6</version>
		</dependency>

	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-assembly-plugin</artifactId>
				<version>3.1.1</version>
				<configuration>
					<descriptorRefs>
						<descriptorRef>jar-with-dependencies</descriptorRef>
					</descriptorRefs>

				</configuration>
				<executions>
					<execution>
						<id>assemble-all</id>
						<phase>package</phase>
						<goals>
							<goal>single</goal>
						</goals>
					</execution>
				</executions>
			</plugin>

			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.6.1</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>
{% endhighlight %}


Next you need to execute some scala code for this to work. In this example we will execute it from spark-shell. 

**Start spark-shell including the required jar file.**
{% highlight shell %}
spark-shell --deploy-mode client \
--jars file:///opt/Kerberos-0.0.1-SNAPSHOT-jar-with-dependencies.jar \
--executor-memory 5G --executor-cores 4 --num-executors 12
{% endhighlight %}


**NOTE:** We were pushing data to the HWX Registry server providing schema evolution information. 
The payload was properly formatted JSON. 

{% highlight scala %}
import com.yoursite.scala.rest.KerberosHttpClient

val princ = "service@YOUR.REALM"
val keytab_loc = "/etc/security/keytabs/service.keytab"
val registry_host="http://your-registry-host:7788"



  def create_registry_group(data_type:String, grp_nm:String, name:String): String = {
    val payload = StringBuilder.newBuilder
    payload.append( "{" )
    payload.append( "\"type\":\"%s\",".format(data_type) )
    payload.append( "\"schemaGroup\":\"%s\",".format(grp_nm) )
    payload.append( "\"name\":\"%s\",".format(name) )
    payload.append( "\"description\":\"Schemas for %s group\",".format(grp_nm) )
    payload.append( "\"compatibility\":\"BACKWARD\"," )
    payload.append( "\"validationLevel\":\"LATEST\"," )
    payload.append( "\"evolve\":\"true\"" )
    payload.append( "}" )
    val krb = new KerberosHttpClient()
    try {
      val r = krb.postUrl( princ,keytab_loc,"%s/api/v1/schemaregistry/schemas".format(registry_host), payload.toString() )
      return r
    } catch {
      case e: org.apache.http.client.HttpResponseException => return null
    }
  }
  
  def get_latest_schema(name:String) : String = {
    val krb = new KerberosHttpClient()
    try {
      val r = krb.getUrl(princ,keytab_loc,"%s/api/v1/schemaregistry/schemas/%s/versions/latest".format(registry_host,name))
      return r
    } catch {
      case e: org.apache.http.client.HttpResponseException => return null
    }
  }

  def register_new_schema(name:String, escaped_avro_schema:String) : String = {
    val payload = StringBuilder.newBuilder
    payload.append( "{" )
    payload.append( "\"description\":\"Schema for %s\",".format(name) )
    payload.append( "\"schemaText\":\"%s\"".format(escaped_avro_schema) )
    payload.append( "}" )
    val krb = new KerberosHttpClient()
    val url = "%s/api/v1/schemaregistry/schemas/%s/versions?branch=MASTER".format(registry_host, name)
    println(url)
    try {
      val r = krb.postUrl( princ,keytab_loc,url.format(registry_host, name), payload.toString() )
      return r
    } catch {
      case e: org.apache.http.client.HttpResponseException => return null
    }
  }
{% endhighlight %}