﻿---
layout: post
title: "Improve the Customer Analytic with scalability and dashboard experience for Omnitech Retail Platform"  
author: "William Dam"
author-link: "#"
#author-image: "{{site.baseurl}}/images/authors/photo.jpg"
date: 2016-12-10
categories: [IoT & App Services]
color: "blue"
#image: "{{site.baseurl}}/images/TofugearImages/ tofugear-logo.jpg" #should be ~350px tall
excerpt: Microsoft work with Tofugear to re-architect its Omnitech Retail platform solution
language: English
verticals: “Retail, Consumer Products & Services”
---

Customer profile
----------------

Company: Tofugear
---------------------------------
Location: Hong Kong
-------------------
Description:
------------
[**Tofugear**](http://www.tofugear.com/) is a new IoT Startup specialize
in providing retail, ecommerce, business engineering, logistic and
supply chain solution. **Tofugear Omnitech** solution is a fully
customized omnichannel retailing platform which offers new and exciting
opportunities ranging from capturing increased sales across channels,
enhanced brand awareness and loyalty, as well as gaining keen insight
into customer 'trying and buying' behavior. They see the opportunity to
adopt IoT Hub, Azure Stream Analytic, Azure function and PowerBI on the omnitech retail
platform to offload current API gateway and deliver better dashboard
viewing to their clients

This is the team that was involved with the project:
-   William Dam – Microsoft, Technical Evangelist
-   William Yeung – Tofugear Chief Architect

Problem statement:
--------------------

**Scenario**

Currently **Tofugear Omnitech** solution will collect both end customer
analytics and transactional information from mobiles (Android, iOS) and
web (client side JS) clients thru an API gateway built using Ruby on Rails
then store into PostgreSQL. Another Ruby worker will pull these
data from PostgreSQL to process regularly and load to Azure Machine
Learning to drive various AI functions for customer service and business intelligence analysis.
Tofugear would like to further improve the scalability of existing
architecture which limited by the API gateway and like to create a
dashboard solution based on PowerBI for better insight of the customer
data to their clients.

In our 1st meeting, we discussed the solution architecture, and we think
it’s better to offload all the end customer analytic information
(mobiles and web client) from the API gateway and only keeping the
transactional data flow to the gateway. We’ll stream all the customer
analytics and the store sensors data (in next phase) to the
IoT Hub instead, then use Azure Stream Analytics to aggerate all these
real-time streaming data and join with the Product Reference data from
Blob (snapshot from the PostgreSQL) to output to the Power BI.

To minimize the code change of the Ruby worker which processing the end
customer analytic data from PostgreSQL written from existing API
Gateway. We’ll need to allow the Ruby worker to pull the client data from
IoT hub instead of the PostgreSQL.

Key technologies:

- IoT: Azure IoTHub, Azure Stream Analytics, Blob
- Web Service: WebApp, Azure Function, Nodejs, Ruby, JS
- Device: Mobile (iOS/Android)
- Dashboard: PowerBI

![Whiteboard Architecture]({{site.baseurl}}/images/TofugearImages/Tofugear-WhiteBoard.jpg)

Solutions, steps and delivery
-------------------------------

**Data Ingestion**

To unify all web and mobile client connections to IoTHub, we decided to
use HTTP REST interface and we’ve spent some time to figure out how to use JavaScript
to generate the SAS token which then need to set in the [HTTP
authorization header](https://docs.microsoft.com/en-us/rest/api/iothub/device-identities-rest#bk_common) in order to connect to IoT Hub.

JavaScript sample in generating the SAS token for IoTHub registration

    var hubName = "xxxxxxxxxxx";        
    var myKey = "xxxxxxxxxxxxxxxxxxxxxxxxxxxx";
    var host = (hubName + ".azure-devices.net").toLowerCase()
    var hostUrl = (host + "/" + url).toLowerCase()
    var endpointUri = "https://" + hostUrl

    // Create a SAS token
    var expiry = 1500000000
    var sigSource = encodeURIComponent(host) + '\n' + expiry
    var signature, hmac;
    hmachash = CryptoJS.HmacSHA256(sigSource, CryptoJS.enc.Base64.parse(myKey))
    signature = CryptoJS.enc.Base64.stringify(hmachash)
    var sasToken = _.template("SharedAccessSignature sig=<%= sig %>&se=<%= se %>&skn=<%= skn %>&sr=<%= sr %>")
    var sas = sasToken({sr: encodeURIComponent(host), sig: encodeURIComponent(signature), se: expiry, skn: 'iothubowner'})
 
    return jQuery.ajax({
        url: endpointUri,
        method: method,
        data: JSON.stringify(data),
        dataType: 'json',
        beforeSend: function(xhr) {
            xhr.setRequestHeader("Authorization", sas);       
        }
    })   

And when we test it on browser client, we found that [IoT hub does not
support CORS](https://github.com/Azure/azure-iot-sdks/issues/1001) thus
return “not allowed access” error below from IoT Hub.
```
*jquery-3.1.1.min.js:4 OPTIONS <https://tofugeariothub.azure-devices.net/devices/webClient> send @ jquery-3.1.1.min.js:4ajax @ jquery-3.1.1.min.js:4xhrRequest @ test.js:32sendTemperature @ test.js:11onclick @ test.html:11*

*test.html:1 XMLHttpRequest cannot load <https://tofugeariothub.azure-devices.net/devices/webClient>. Response to preflight request doesn't pass access control check: No Access-Control-Allow-Origin' header is present on the requested resource. Origin '<http://localhost:3001>' is therefore not allowed access. The response had HTTP status code 405.*
```

![IoTHub CORS with WebClient]({{ site.baseurl }}/images/TofugearImages/Tofugear-WebClientCORS.jpg)

So, we look into alternative with Azure function to act as the proxy. We
create 2 Azure Functions as proxy responsible for device registration
and decide to apply for all clients (web and mobile) registration so that
we won’t expose the IoTHub SAS token for device creation on the client side, and the send
message proxy will be used by the web client only till IoTHub can
support CORS later. iOS and Android will use the same REST interface to send the
device message directly to IoTHub as we test it work on Postman.

Sample of the data ingestion from WebClient
```
{
    "deviceId" : "webClient101",
    "deviceMessage" : {
        "event": "browse_product",
        "event_value": "https://shop.xxxxxxxxxx.co.xx/ja/shop/products/162-spangle-bag?color=379",
        "size_name": "FREE",
        "color_name": "BLACK",
        "product_id": 162,
        "referer_product_id": null
    },
    "deviceKey": "gtNQ6rNF7m+rurHHh27w+i3D6NSEdCgUvfSVJnOnKys="
 }
```

![Architecture to include AzFn to overcome CORS issue]({{site.baseurl}}/images/TofugearImages/Tofugear-withWebClientProxyAzFnArch.jpg)

This following Azure Function code is used for IoTHub Device Registration for all clients:

    var connectionString = `HostName=${process.env.IOTHUB_HOSTNAME};SharedAccessKeyName=iothubowner;SharedAccessKey=${process.env.IOTHUBOWNER_SHAREDACCESSKEY}`
    var registry = iothub.Registry.fromConnectionString(connectionString)
    // var device = new iothub.device(null) //remove this code as its seems the new npm package keep complain that device is not constructor
    // replace with the following 
    var device = {
        deviceId: null
    };
    device.deviceId = req.body.deviceId
    registry.create(device, function (err, deviceInfo, res) {
        context.log('IoTHub connected......');
        if (err) {
            registry.get(device.deviceId, function(err, deviceInfo, res) { 
                context.res = {
                    status: 500,
                    body: JSON.stringify({
                        "status": 500,
                        "error": 'unable to create device',
                        "deviceInfo": deviceInfo 
                    })
                }              
                context.done()          
            });                     
        } else if (deviceInfo) {
            context.res = {
                status: 201,
                body: JSON.stringify({ "deviceInfo": deviceInfo })
            }
            context.log('Device registered......');
            context.done();
        }
    })

This is the IoTHub webClient message Proxy Azure Function code to help web client to bypass the CORS issue:

    var connectionString = `HostName=${process.env.IOTHUB_HOSTNAME};DeviceId=${req.body.deviceId};SharedAccessKey=${req.body.deviceKey}`  
    var client = clientFromConnectionString(connectionString);
    var messageSent = false;
    var connectCallback = function (err) {
      if (err) {
        context.log('Could not connect: ' + err);
      } else {
        context.log('IotHub connected');
        // Create a message and send it to the IoT Hub
        var msg = new Message(JSON.stringify({ deviceId: req.body.deviceId, Data: req.body.deviceMessage }));
        context.log('Message sending.....');
        client.sendEvent(msg, function (err) {
          if (err) {
            console.log(err.toString());
          } else {
            context.log('Message sent');
            messageSent = true;
            context.res = {
                  status: 201,
                  body: JSON.stringify({Data: req.body.deviceMessage + ' from ' + req.body.deviceId + ' sent successfully'})
              }
              context.done();
          };
        });
      }
    };
    client.open(connectCallback);


**Data Processing**

We then connect these Azure Functions to IoTHub with Stream Analytics and
also using blob storage that store the reference data product snapshot
for Stream Analytics to combine for richer PowerBI output.

![Stream Analytics]({{site.baseurl}}/images/TofugearImages/Tofugear-StreamAnalytic.jpg)

Sample of Stream Analytic combing the client data and product reference to output to PowerBI
```
SELECT
    skus.id as sku_id,
    metrics.Data.event,
    metrics.Data.event_value,
    metrics.Data.size_name,
    metrics.Data.color_name,
    metrics.Data.user_id,
    skus.price, 
    skus.product_state,	
    skus.product_code,
    skus.can_purchase
INTO
    Tofugearpowerbidataset
FROM
    [TofugearIoTHubInput] metrics 
LEFT JOIN [Tofugear-Ref-Data] skus on metrics.Data.product_id = skus.product_id
```

Since we like to separate the IoTHub consumer group for Azure Stream
Analytics and Ruby worker. We create 2 consumer groups with one consumed
by Stream Analytics to output to PowerBI and another consumer group which
will be used by the existing Ruby worker to pull the web and mobile
client telemetry analytic data for processing.

We spend quite sometimes hacking the [Apache Qpid Proton
package](http://qpid.apache.org/proton/index.html) as there’s no IoTHub
SDK support for Ruby which require AMQP1.0. We’ve no success after few
tries and it’s also take too much effort if we consider using Ruby
inline to wrap the [azure-iot-sdks
C](https://github.com/Azure/azure-iot-sdks/tree/master/c) library. To
bypass the amqps connection challenge for Ruby worker, we decide to
create another Azure Function to allow the existing Ruby worker on
demand using HTTP to pull the IoTHub (receiver side) end customer
analytic data for processing instead of direct streaming the IotHub data
to Ruby worker as its won’t able to handle the capacity.

![Architecture to add another AzFn to allow Ruby worker to pull from IoTHub]({{site.baseurl}}/images/TofugearImages/Tofugear-withRubyWorkerProxyArch.jpg)

This is the Azure Function code to allow Ruby worker to pull the message from IoTHub:

    var connectionString = `HostName=${process.env.IOTHUB_HOSTNAME};SharedAccessKeyName=iothubowner;SharedAccessKey=${process.env.IOTHUBOWNER_SHAREDACCESSKEY}`
    var printError = function (err) {
    context.log(`error occurred: ${err.message}`);
    };
    var messageList = []
    var messageBodyList = []
    var appendMessageToList = function (message) {
        messageBodyList.push({
            offset: message.offset,
            sequenceNumber: message.sequenceNumber,
            enqueuedTimeUtc: message.enqueuedTimeUtc,
            body: message.body
        })
        messageList.push(message)
        context.log(message)
        return true;
    }
    var client = EventHubClient.fromConnectionString(connectionString);
    var closeClientAndCompleteContext = function() {
        client.close();
        context.done();
    }
    client.open()
        .then(client.getPartitionIds.bind(client))
        .then(function (partitionIds) {
            context.log('Connected to IoTHub.....');
            setTimeout(function() {
                        context.res = {
                            status: 201,
                            body: JSON.stringify({'messages': messageBodyList})
                        }
                        
                        closeClientAndCompleteContext()
                
                    }, 5000)
                    
            return partitionIds.map(function (partitionId) {
                context.log('Retirving Data from queue.....');
                return client.createReceiver(process.env.MESSAGE_POLL_CONSUMERGROUP, partitionId, { 'startAfterOffset': (req.query.after_offset || 0) }).then(function(receiver) {
                    context.log(`connected. PartitionId: ${partitionId}`)                
                    receiver.on('errorReceived', printError);
                    receiver.on('message', appendMessageToList);
                    
                });
            });
        })
        .catch(printError);

We observed couple of unexpected behavior with the IoTHub on receiver
side. This 1st unexpected issue is that we found the free version of IoT
hub does not support other than $Default consumer group with error
message of the IoTHub could not be found but the paid IoTHub version is
fine. I would recommend we document its if it’s a limitation.

Because we’re using the offset to pull the data from the IoTHub queue,
we also found that the [EventHub
createReceiver](https://github.com/Azure/azure-event-hubs/blob/master/node/send_receive/lib/client.js)
function call will return invalid respond in below screenshot for those
partitions that doesn’t contain the specific offset. However, the call
still return successfully just the error prompt is a bit annoying.

```
2016-11-01T04:26:55.136 The supplied offset '336' is invalid. The last
offset in the system is '-1'
TrackingId:8c2c5345-efe2-4cf9-8952-d5ea4a62dd70\_B2,
SystemTracker:iothub-ns-tofugeario-73126-2c3dc3dc23:eventhub:tofugeariothub\~24575,
Timestamp:11/1/2016 4:26:55 AM
Reference:f1637dbe-9ec9-462b-b73e-e2c1a06488bc,
TrackingId:25df2420-84d1-47e9-a0e3-f2a4beaee67c\_B2,
SystemTracker:iothub-ns-tofugeario-73126-2c3dc3dc23:eventhub:tofugeariothub\~24575|streamanalytic,
Timestamp:11/1/2016 4:26:55 AM
TrackingId:c72526651ba4472dbb3bdb9a7fc3821a\_G0, SystemTracker:gateway2,
Timestamp:11/1/2016 4:26:55 AM.............................
2016-11-01T04:26:59.230 Function completed (Success,
Id=7a4e2e9d-5902-449c-ba4f-02c349994f0c)
```

Performance tuning
-------------------

Basically, we’ve the end-to-end flow establish and next thing that we’re
looking into is to improve the startup time for Azure Functions as its
exhibit some sort of cold start symptoms and we use Dynamic plan for
cost saving. So, we decide to setup another Timer Trigger KeepAlive HTTP
ping function to keep these 3 Azure Functions warm and also moving all
the npm package loading outside the Function call will improve the
unnecessary npm package load as long as the Functions are warm.

![Final Architecture and including a Timer Trigger AzFn to keep all AzFns warm]({{site.baseurl}}/images/TofugearImages/Tofugear-FinalArch.jpg)

And this is the Timer trigger code that ping other Azure Functions to keep them warm:

    context.log("Timer triggered at " + myTimer.next);
    var pingPaths = [
        '/api/devices?action=ping&code=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
        '/api/messages?action=ping&code=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
        '/api/message_feed?action=ping&code=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    ]
    context.log("timer passed?",myTimer.isPastDue)
    if(myTimer.isPastDue)
    {
        context.log('Node.js is running late!');     
    } else {
        pingPaths.map(function (path)  {
            var url = `https://${process.env.AF_HOST}${path}`
            context.log(`ping url: ${url}`)
            var req = https.get(url)
            req.end()
        })
        context.log("Timer triggered at " + myTimer.next);
    };
    context.done();


We observed quite good respond time with average like 400ms or sometime less except the initial start of the
Azure Function. Then we start noticed that there’s seems some
long start up occasionally may up to mins and after some investigation
we suspect some combination of the IoTHub connection setup and Azure
Function environment may contribute this unexpected result which we’re
working closely with product team to investigate. It’s important to get
this performance issue resolve to apply in production environment.

![Respond time for the message proxy, excluding the error cases]({{site.baseurl}}/images/TofugearImages/AzFnPerformance.jpg)

Conclusion
----------

This combined effort from Microsoft and Tofugear has delivered the POC
that gave the partner a glance on how IoT Suite and Azure Function can
provide easy scaling and integration and with almost real-time combined
data visualization in PowerBI. The project was implemented in a bit over
4 weeks with only 1 TE and 1 partner developer and can be shorter if not
some unexpected technical challenges.

We accomplished our goal of making a scalable and better visualization
product integration with this POC. Tofugear is willing to commit to work
with us on-going with the product team actively helping us in resolving
the performance issue and we aim to bring this POC into production
integration as soon as this performance issue been resolved.

Tofugear is setting high expectations and committing to work with us
together towards a production launch of the product. While many of our
frameworks and solutions solve the business or technical problem a
customer has, we’re appreciated the resources and bandwidth the customer
has to maintain, debug, or troubleshoot what we put together.

![Tofugear team]({{site.baseurl}}/images/TofugearImages/Tofugear-team.jpg)

Here is a quote from the customer:

“Our partnership with Microsoft on this Tofugear Omnitech project has
bring us close relationship to work as partner. This new architecture
significantly changes the way tracking end customer analytic data in
communicate with a central system that receives and stores data, while
also allowing to visualize these close to real time data in a meaningful way. It brings
performance and cost benefits and will definitively leverage our sales
in this segment. This is what the market needs: solutions that add value
while at the same time reducing the complexity of the integration to our
platform would let us more focusing to deliver more customer value and feature delivery.”
