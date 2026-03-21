---
title: 'Printing labels via an external service'
date: '2024-05-31'
author: 'lukaszj'
image: '/images/CloudPrinter-1.jpg'
categories:
  - Azure
  - Dynamics365
slug: 'printing-labels-via-an-external-service'
draft: false
---

In one of the last Tech Talks, we can watch how to print warehouse labels via an external service (cloud label printing).

The Microsoft engineers also presented that it is possible to print labels from the cloud Dynamics 365 F&O in the on-premises software such as BarTender or NiceLabel in many ways but the official documentation doesn't show this in detail.
Link to the TechTalk video: [Beyond the DRA Cloud Label Printing for Dynamics 365 Supply Chain Management | TechTalk](https://www.youtube.com/watch?v=DuUcwo5ufdo&t=2479s)

One of the many ways to print labels in on-premises software is to use the Azure App Service hybrid connection.
In this post, I will present how to print labels from Dynamics 365 in on-premises BarTender using the hybrid connection. I described how to configure a hybrid connection in my [previous blog post](/azure-appservice-hybrid-connection/).

## Setup on Azure portal

The first step is to create a new Hybrid Connection to my BarTender service. I'm not going to show how to configure BarTender because I'm not an expert in that area and it's beyond the scope of this post.

My BarTender service is exposed on the local address:
**http://WIN2022TEST:8081/Integration/LabelPrint/Execute**

In the Azure Portal and Hybrid Connection Manager, I need to create a new hybrid connection to this service:

![Azure Portal - new Hybrid Connection setup](/images/HC1.png)

![Hybrid Connection Manager - new connection](/images/HC2.png)

The next step is to create a new environment variable in my Azure Function app, which I'm using as middleware. It's a connection string to my on-premises BarTender service:

![Azure Function App - new environment variable](/images/AzFunc_1.png)

Remember that the variable's name must be the same as the name in the function code.

Now I have the connection between Azure and my on-premises print service.
I also added the Azure function to API Management as a new API – but this is an optional step:

![API Management - adding Azure Function as a new API](/images/APIM1.png)

So now this is the full address of my API to print the labels on the local BarTender from the internet:
<https://apimlujatest.azure-api.net/func-hybrid-tst/Execute>

My API is secured with a subscription key because I have selected the "Subscription required" option.

## Setup in Dynamics 365 F&O

Now I can configure my Dynamics 365 cloud to use this URL. First I need to create a new external service instance and definition.
I go to: **Warehouse management -> Setup -> External Services -> External service definitions**

I create a new service definition with the parameters:

- **HTTP method:** POST
- **Operation timeout:** e.g. 5000
- **Request body type:** Raw
- **Relative URL:** /func-hybrid-tst/Execute
- **Request HTTP headers:** I put the APIM subscription-key there.

**Request body (raw)**
- Content type: `application/json`
- Request body:

```json
{
    "PrinterName": "$label.printer$",
    $label.body$
}
```

Here is how it looks:

![D365 - External service definition](/images/D365_1.png)

Then, I go to **Warehouse management -> Setup -> External Services -> External service instances** form and create a new instance with parameters:

- **Base URL:** URL of my API Management — <https://apimlujatest.azure-api.net>
- **Authentication secret:** The subscription key for my API — I can find it in my APIM definition.
- **External service definition:** The definition that I have already created.
- **Logging:** Select *Successes and errors* to log all messages.

![D365 - External service instance configuration](/images/D365_2.png)

Next, I configure the Document Routing setup and Document Routing Layout. I will try to print the purchase order label from Warehouse *24*.

Document routing setup:

![D365 - Document routing configuration](/images/D365_3.png)

![D365 - Document routing layout](/images/D365_4.png)

Next, I create a new label printer to print from my service.
The connection type must be set to: *External label service* and the label print service instance must be set to the service instance that I have already created.

![D365 - new label printer setup](/images/D365_5.png)

Finally, I set up a default printer for my user.

![D365 - default printer for user](/images/D365_6.png)

Next, I print the label and I can check the results in the **External Service Request Log** form.

![D365 - External Service Request Log](/images/D365_7-1.png)

I can check the BarTender message log to see if the label was sent to the printer.

![BarTender message log](/images/Bartender1.png)

I can also monitor this integration in Application Insights.

![Application Insights - integration monitoring](/images/AppIns1.png)

## A few final thoughts

I presented one of many ways to print labels in on-premises software. In my opinion, it is easy to configure and the advantage of using Azure solutions like App Service hybrid connection and API Management gives us e.g. the ability to monitor this solution.

The total cost of this solution is the cost of the App Service plan because the hybrid connection feature is not available in the Consumption plan.
