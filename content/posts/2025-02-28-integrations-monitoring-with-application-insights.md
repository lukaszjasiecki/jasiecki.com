---
title: 'Integrations monitoring with Application Insights'
date: '2025-02-28'
author: 'lukaszj'
image: '/images/AppIns_Title-1.jpg'
categories:
  - Azure
  - Dynamics365
slug: 'integrations-monitoring-with-application-insights'
draft: false
---

Every Dynamics 365 implementation involves connections to other systems, utilizing technologies such as oData and custom web services. However, Dynamics 365 lacks built-in functionality to monitor outgoing or incoming requests, leaving no out-of-the-box solution for monitoring or alerts.

Application Insights, a feature of Azure Monitor, enables monitoring for web applications like D365. To integrate Application Insights, we also need Azure [API Management (APIM)](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts), which acts as a gateway for routing requests to Dynamics 365.

## Requirements

To implement this solution, we need to create two Azure services:

1. API Management instance
2. An instance of the Application Insights service

## Implementation

The first step is to add a new API to APIM. In this example, we'll use a standard oData service for managing customer data. Additionally, it is crucial to enable the **Subscription required** parameter to secure the API with a subscription key.

{{< figure src="/images/APIM_1.png" caption="Azure portal – creating a new API" >}}

Next on the same screen, I have to enable logging to the Application Insights.

![APIM – enabling Application Insights logging](/images/APIM_2.png)

I can choose which operations to log:

![APIM – selecting operations to log](/images/APIM_3.png)

I would like to enable:

- **Frontend Request** – Incoming request will be registered
- **Backend Response** – Outgoing response from D365 will be registered

Finally, I have two operations: POST and GET to operate on the oData endpoint:

![APIM – POST and GET operations overview](/images/APIM_4.png)

What's important – we should authenticate if we want to send requests to the D365. To avoid doing this in two steps (first send a request to get the Bearer token and then send the next request to the oData service), we can use the APIM inbound policy.

The policy would look like this:

```xml
<inbound>
    <base />
    <send-request mode="new" response-variable-name="bearerToken" timeout="20" ignore-error="true">
        <set-url>{{auth-url}}</set-url>
        <set-method>POST</set-method>
        <set-header name="Content-Type" exists-action="override">
            <value>application/x-www-form-urlencoded</value>
        </set-header>
        <set-body>@{ return "client_id={{d365-appId}}&resource={{d365-url}}&client_secret={{d365-secret}}&grant_type=client_credentials"; }</set-body>
    </send-request>
    <set-header name="Authorization" exists-action="override">
        <value>@("Bearer " + (String)((IResponse)context.Variables["bearerToken"]).Body.As<JObject>()["access_token"])</value>
    </set-header>
    <retry condition="@(context.Response.StatusCode == 429)" count="10" interval="10" max-interval="100" delta="10" first-fast-retry="true" />
</inbound>
```

It works by first sending a request to the service responsible for authentication, which returns a token. Then, a request is sent to the target service in D365 with the bearer token included in the header.

So now the address where I can request my D365 application to operate on the customers is as follows:
<https://apimlujatest.azure-api.net/d365-odata/CustomersV2>

## Sending request

Now, I will send the request to the APIM endpoint to create a new customer:

{{< figure src="/images/Postman_1.png" caption="Postman request" >}}

The response looks like this:

{{< figure src="/images/Postman_2.png" caption="Postman response" >}}

I can check the details in the Application Insights:

{{< figure src="/images/AppIns_1.png" caption="Azure Application Insights dashboard" >}}

Here are the details of the request and the response:

![Application Insights – request details](/images/AppIns_2.png)

{{< figure src="/images/AppIns_3.png" caption="Application Insights – request details" >}}

{{< figure src="/images/AppIns_4.png" caption="Application Insights – response details" >}}

I can also use the KQL to query the logs and find the requests using filters:

![Application Insights – KQL query](/images/KQL_1.png)

## A few final thoughts

I have demonstrated how we can monitor traffic between Dynamics 365 and various systems using Azure API Management and Application Insights. By implementing this solution, we gain better visibility into API traffic, allowing us to track requests, detect anomalies, and troubleshoot potential issues effectively.
