---
layout: post
date: 2013-09-16
title: Advanced Cookie-based session affinity with Application Request Routing
---

As you probably know, the Windows Azure Platform has a built-in load-balancing service that you can typically use to create "farms" of Web servers for high availability and performance scale-out. You can find some detailed information on how to set this up in the Windows Azure documentation, like this article: [Load Balancing Virtual Machines](http://www.windowsazure.com/en-us/manage/windows/common-tasks/how-to-load-balance-virtual-machines/).

One well-known characteristic of the Windows Azure load balancer is that it only provides a round-robin algorithm, and does not support so-called "sticky" sessions. This is not a problem for most Web sites, however it can prove challenging when migrating more complex projects to the Cloud, where the architecture may rely on sticky sessions provided by an on-premises load balancing solution.

We typically recommend using our [IIS Application Request Routing (ARR) module](http://www.iis.net/downloads/microsoft/application-request-routing) to implement sticky session if they are required for the application. There are several resources describing this solution, here are a few interesting links:

- [Using the Application Request Routing Module](http://www.iis.net/learn/extensions/planning-for-arr/using-the-application-request-routing-module) : this is the standard installation documentation from the IIS team, that you could for example follow to deploy ARR on Windows Virtual Machines.
- The excellent [Richard Astbury](http://coderead.wordpress.com/) from our partner [Two10Degrees](http://www.two10degrees.com/) has an [AzureARR project hosted on Github](https://github.com/richorama/AzureARR) that packages ARR in a Web Role for use in Cloud Services.

However, I recently came across a partner who required more control over the sticky-session algorithm that what ARR provides. In order to better understand the requirements, let's first look at how ARR implements session stickyness.

The technique ARR uses is usually known as a "Cookie Insertion" method. The idea is that ARR, standing in between a client browser and the server farm, will intercept incoming requests, choose a server to "stick" the session to, and will insert in the server response a custom HTTP Cookie that contains the name of the server. To be precise, ARR does not insert the server name in clear text, but rather a hash of the name, but the effect is the same. In subsequent requests, the browser will "honor" the Cookie and send it back to ARR. ARR will then keep on sending the requests in that "session" to the server whose name is in the Cookie.

To illustrate, let's look at some HTTP requests using curl:

	$ curl -i http://arriis.cloudapp.net/
	HTTP/1.1 200 OK
	Content-Type: text/html
	Last-Modified: Mon, 29 Jul 2013 10:35:50 GMT
	Accept-Ranges: bytes
	ETag: "7c57e964478cce1:0"
	Server: Microsoft-IIS/7.5
	X-Powered-By: ASP.NET
	X-ServerName: iis1
	Date: Mon, 16 Sep 2013 13:06:44 GMT
	Content-Length: 10

	hello iis1

This first request is made directly to the load-balanced endpoint of a Cloud Service consisting of two IIS Virtual Machines. As you can see, the request was served by a VM named "iis1". If I run this same command again, the next request will be served by "iis2", the second server in my Cloud Service: (response headers edited for brevity)

	$ curl -i http://arriis.cloudapp.net/
	HTTP/1.1 200 OK
	X-ServerName: iis2

	hello iis2

The "X-ServerName" header is not standard in IIS, I added it manually in order to facilitate troubleshooting the load-balancing setup. You would typically remove it in a production environment! Adding the header can be done using the custom [HTTP Response Headers section in IIS Manager](http://www.iis.net/configreference/system.webserver/httpprotocol/customheaders).

Now that we have seen the standard round-robin algorithm offered by Windows Azure, let's look at the HTTP responses when ARR is added as a front-end to the IIS servers:

	$ curl -i http://arrarr.cloudapp.net/
	HTTP/1.1 200 OK
	Content-Length: 10
	Content-Type: text/html
	Last-Modified: Mon, 29 Jul 2013 10:34:59 GMT
	Accept-Ranges: bytes
	ETag: "d092b446478cce1:0"
	Server: Microsoft-IIS/7.5
	Set-Cookie: ARRAffinity=6e04a77b7c6917600d9ff7ee980a4d4a8a0dfb135247289a9c58cf88587bec23;Path=/;Domain=arrarr.cloudapp.net
	X-Powered-By: ASP.NET
	X-ServerName: iis2
	X-Powered-By: ARR/2.5
	X-Powered-By: ASP.NET
	Date: Mon, 16 Sep 2013 13:12:43 GMT

	hello iis2

As you can see, we now have a couple additional HTTP headers in the response: the "X-Powered-By: ARR/2.5" confirms that we are going through ARR, and the Set-Cookie header asks the client to add a Cookie named "ARRAffinity" to all subsequent requests.

To simulate a well-behaved browser that honors Cookie headers, we can manually add the Cookie in curl:

	$ curl -i --cookie ARRAffinity=6e04a77b7c6917600d9ff7ee980a4d4a8a0dfb135247289a9c58cf88587bec23 http://arrarr.cloudapp.net/
	HTTP/1.1 200 OK
	X-ServerName: iis2
	X-Powered-By: ARR/2.5

	hello iis2

As long as we keep on sending the Cookie header, all our requests will now go to the same server in our farm. Fantastic!

For most use cases, Cookie Insertion Just Works and is very easy to setup using ARR. However, here is one very specific case where it falls short!

Where Cookie Insertion falls short
----------------------------------

My partner has the following situation: he has a number of customers who use a library, or SDK, to connect to his Web services. Session affinity is required for his services to function, because a good amount of customer data is cached in memory in order to accelerate his processing algorithms. Because the client is a code library that does not maintain client-side state, it will not honor Cookies that are sent by the application or ARR: it only sends the header values (and Cookies) as hard-coded in the library.

They currently implement session affinity using a hardware load-balancing device that provides "rule-based load balancing". This load-balancing technique does not require inserting an additional Cookie: it uses the value of an existing application Cookie to calculate a hash that points to a given server. The load-balancing device is configured to inspect the value of a specific application Cookie which is sent by the library, let's call it "CUserID".

How can we reproduce this behavior using ARR?

Since we are looking for a hash-based load balancing algorithm, let's look at what ARR offers, extracted from the [TechNet documentation for ARR](http://technet.microsoft.com/en-us/library/dd443524(v=ws.10).aspx):

- Server variable hash – Distributes traffic based on a hashed value of a server variable.
- Query string hash – Distributes traffic based on the hashed value of the query string value. When more than one query string name is specified, the concatenated string of the corresponding query string value is used for the hash.
- Request hash – Distributes traffic based on the hashed value of the configured server variable or URL. For example, if the server variable is QUERY_STRING, the hashed value is based on the names in the request query string.

Unfortunately, none of these options can be directly used to hash on a Cookie value! 

But here is what we can do by adding a simple Rewrite Rule: we can capture the value of the Cookie using a Condition, and then create a custom Server Variable that contains the value of the Cookie. We can then use the Server Variable Hash algorithm on this custom variable to bind the session based on the Cookie value!

Let's look at the ARR configuration step by step.

First, you should disable Client Affinity if it was enabled.

Then, using IIS Manager, in the URL Rewrite module at the server level, add a new rule before the ARR rule (in this example, "myfarm_with_cookie"):

![List of rewrite rules](/images/arr/arr1.png)

Configure the rule to match regular expressions (not wildcards - this is important for the next steps). Use .* as the pattern to match.

![Configure a new rule](/images/arr/arr2.png)

In the Conditions panel, add a new condition, where you will match a pattern on the HTTP_COOKIE server variable. For the pattern, use something like this:

	CookieName=([0-9.A-Za-z]+)

This will match the Cookie value and store it in {C:1}.

![Condition](/images/arr/arr3.png)

Now in the Server Variables panel, create a new variable, e.g. LB_COOKIE in my example, and assign to it the value {C:1} to retrieve the value of the Cookie matched by the condition.

![Server Variable](/images/arr/arr4.png)

Finally, in the Action panel, just route the request to the server farm like the default rule does.

![Routing](/images/arr/arr5.png)

Now you have a brand new server variable you can use to base your load-balancing on!

Just go in the ARR module configuration and select the "Server variable hash" algorithm, and type the name of your variable in the text box, e.g. LB_COOKIE in my case.

![Load-balancing algorithm](/images/arr/arr6.png)

You should now have a brand new Cookie-based load balancing setup, with no Cookie insertion.
