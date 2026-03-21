---
title: 'Azure App Service hybrid connection'
date: '2024-05-04'
author: 'lukaszj'
image: '/images/HC_Title.jpg'
categories:
  - Azure
slug: 'azure-appservice-hybrid-connection'
draft: false
---

In my very first post, I would like to talk about Azure App Service Hybrid Connection.

During Dynamics 365 implementation, we usually have to design and develop many integrations. The integrations are various – web services, importing data from sources such as SQL databases using e.g. Recurring Integration, or Odata.
The technologies are different, but so are the services we need to connect to. If they are exposed "to the world" then establishing communication with cloud-based D365 should not be a major problem.

However, there are times when a service cannot be exposed to the world and is only available on the customer's internal (on-premises) network. Connecting D365 in the cloud to an on-premises web service is not possible by default – because cloud D365 does not have access to the on-premises network.

In this case, the Azure App Service hybrid connection solution comes to the rescue. Hybrid connection is available as a feature of App Service – note that it is not available in the consumption plan.

Hybrid Connection creates a tunnel between the App Service and the on-premises application/service, communicating over port 443. Therefore, it can be used for communication with applications on any network where port 443 is open for communication.
Link to the documentation: <https://learn.microsoft.com/en-us/azure/app-service/app-service-hybrid-connections>

![Azure App Service Hybrid Connections - Microsoft documentation](/images/MS_documentation.png)

In the solution presented next, I will use Azure Function. The Azure Function must be hosted in App Service. I will choose the plan with the lowest price – Basic.
The Basic plan allows you to create a maximum of 5 hybrid connections.

![App Service plan pricing tiers](/images/Plans.png)

In the first step, I create a new App Service with a Basic price plan.

![Creating a new App Service Plan](/images/AppServicePlan-1.png)

Next, I create a new Function App and add it to the previously created Application Service:

![Creating a new Function App](/images/FunctionApp-1.png)

I have the function app, so now I need to deploy a function that is responsible for forwarding my requests to the particular endpoint. The function code would look like this:

![Azure Function code for request forwarding](/images/Function-1.png)

I will pass the connection string to my local API as a parameter of the Function App:

![Function App connection string configuration](/images/ConnectionString.png)

After creating the function app, I can configure the hybrid connection. To do this, I need to open:
**Function App -> Networking -> Hybrid Connections.**
Click on: Add hybrid connection

![Azure Portal - Hybrid Connection configuration](/images/AzPortal_HybridConnection.png)

I have to fill in some information:
- **Hybrid connection Name:** Unique name of the hybrid connection
- **Endpoint host:** This is the name of the host on which our on-premises API is installed
- **Endpoint port:** The port of the on-premise API
- **Service Bus Namespace:** Because hybrid connection is built using service bus namespace, we need one

OK. I have created a new hybrid connection in Azure Portal, but I also need to install the Hybrid Connection Manager on the server that hosts our on-premises Web service.
I can download the connection manager installation file from **Function App -> Networking -> Hybrid connections** page:

![Azure Portal - Hybrid Connection Manager download](/images/AzPortal_HybridConnection2-1.png)

After installation of Hybrid Connection Manager (HCM) I have to add my connection to the manager. To do this I have to click Enter Manually and copy the connection string from Azure Portal:

![HCM - adding connection manually](/images/HCM1-1.png)

The new connection in HCM is usually created with the status ***Not connected***

![HCM - connection status Not connected](/images/HCM2.png)

To fix the connection status to Connected I need to restart the **"Azure Hybrid Connection Manager Service"** service.
After that, I can see that HCM is now connected to Azure.

![HCM - connection status Connected](/images/HCM3-1.png)

After configuring HCM, I have created a kind of tunnel between the App Service and the on-premises service.
The App Service will work as middleware in our scenario.
In the App Service, I have deployed the Azure function that will forward our request to the selected address.

My local service is a very simple service with the address: *http://win2022test:3000/Customers*
I can test connections using e.g. Postman. I will send a request to Azure Function.

![Postman - test request to Azure Function](/images/Postman.png)

The new record has been created.

![Insomnia - result verification](/images/Insomnia.png)

I have presented the solution that helps me to connect from Dynamics 365 to the on-premises API.
In the next post, I would like to present how to use this in the real scenario in Dynamics 365 and also how we can monitor the request using Azure API Management and Application Insights.
