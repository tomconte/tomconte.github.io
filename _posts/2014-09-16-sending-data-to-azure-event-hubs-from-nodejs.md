---
layout: post
date: 2014-09-16
title: Sending data to Azure Event Hubs from Node.JS using the REST API
---

[Event Hubs](http://azure.microsoft.com/en-us/services/event-hubs/) are a highly scalable pub-sub ingestion service, part of the Microsoft Azure Service Bus, capable of elastic scale to handle millions of events per second from millions of connected devices so that you can process and analyze the massive amounts of data produced by your connected devices and applications (thanks [Scott Guthrie](http://weblogs.asp.net/scottgu/azure-virtual-machine-machine-learning-iot-event-ingestion-mobile-sql-redis-sdk-improvements) for the one-sentence description!).

One important characteristic of Event Hubs is that they will let you securely send telemetry data from any device and any environment, as long as it can deal with HTTPS REST calls. The [developer guide](http://msdn.microsoft.com/en-us/library/azure/dn789972.aspx) describes in more details the mechanisms used to scale the access control mechanisms to millions of devices: basically, each device will send its data to a unique Publisher construct, an endpoint materialized by an URI that is specific to that device. These Publisher URIs can be secured by a per-publisher token, so that only the device that possesses the right token can publish to the Publisher endpoint.

All of this can be achieved by using the REST API, from any environment. The ["Service Bus Event Hubs Large Scale Secure Publishing"](http://code.msdn.microsoft.com/windowsazure/Service-Bus-Event-Hub-99ce67ab) C# sample shows how to achieve this using only REST calls, including the generation of the Publisher SAS tokens.

I wrote a Node.JS sample to demonstrate the same from a pure JavaScript environment. It uses the [Send Event](http://msdn.microsoft.com/en-us/library/azure/dn790664.aspx) REST API to send telemetry data, and authenticates using a Service Bus [Shared Access Signature (SAS) token](http://msdn.microsoft.com/en-us/library/azure/dn170477.aspx).

The Event Hub REST API call itself is simple, the tricky part is to generate the SAS token. Here is the function that does the job:

~~~javascript
// Create a SAS token
// See http://msdn.microsoft.com/library/azure/dn170477.aspx

function create_sas_token(uri, key_name, key)
{
    // Token expires in one hour
    var expiry = moment().add(1, 'hours').unix();

    var string_to_sign = encodeURIComponent(uri) + '\n' + expiry;
    var hmac = crypto.createHmac('sha256', key);
    hmac.update(string_to_sign);
    var signature = hmac.digest('base64');
    var token = 'SharedAccessSignature sr=' + encodeURIComponent(uri) + '&sig=' + encodeURIComponent(signature) + '&se=' + expiry + '&skn=' + key_name;

    return token;
}
~~~

Here is a breakdown of the process:

- Generate an expiration value: using the [Moment.js](http://momentjs.com/) library, start from the current time, add one hour, and convert the result into a [Unix Timestamp](http://en.wikipedia.org/wiki/Unix_time) (number of seconds since the Epoch).
- URL-encode the URI and the expiry.
- Use the Node.JS crypto module to generate an SHA256 HMAC (Base64-encoded) using the secret key.
- Finally assemble all the components to create the SAS token.

Before you can use this function, you need to go in your Event Hub configuration page, and create a Shared Access Policy. In my case I created a policy named "send" and associated the "Send" permission. This will generate two secret keys associated to this policy (one primary and one secondary). These keys are used to generate the SAS tokens, and are never transmitted to the devices: only the generated tokens are required to send data.

![Event Hub Shared Access Policy](/images/event_hub_sas.png)

Then you would use these configuration values to call the function:

- `uri`: the URI for which you want to generate a SAS token. In our case, this is the full URI of the publisher associated to our device, e.g. something that looks like this:

~~~
https://mynamespace.servicebus.windows.net/myeventhubs/publishers/mydevice/messages
~~~

- `key_name`: the name of the Shared Access Policy, i.e. "send" in my case.
- `key`: the corresponding key.

Below you will find the Gist for the full source code.

<script src="https://gist.github.com/tomconte/b06f8bb1c5c036e8047c.js"></script>
