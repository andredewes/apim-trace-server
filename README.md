# API Management: extract real requests traces temporarily by using FrontDoor or Application Gateway

## Introduction

One of the most useful debug features of API Management is [request tracing](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-api-inspector). The problem is: we need to reproduce the same client request with an extra header (Ocp-Apim-Trace) or tell the consumers to do so.

What if we could just generate the traces from the APIM clients without telling them to add such header or do that ourselves? Is there an easier way to have APIM log the request traces coming from the real consumers temporarily for a certain period without much effort?

## Pre-requisites

- [API Management](https://learn.microsoft.com/en-us/azure/api-management/get-started-create-service-instance) instance
- [Azure FrontDoor](https://learn.microsoft.com/en-us/azure/frontdoor/create-front-door-portal) instance or [Azure Application Gateway](https://learn.microsoft.com/en-us/azure/application-gateway/quick-create-portal) instance

## Azure FrontDoor or Application Gateway in combination with API Management

If you read the API Management [documentation](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/apis/protect-apis) recommended about the best Security practices to protect APIs, [WAF](https://learn.microsoft.com/en-us/azure/web-application-firewall/overview) (Web Application Firewall) is one of them.

Now, APIM itself doesn't support WAF. We need to have another Azure service in front of it that supports WAF. Two of the main recommendations is to either use [Azure FrontDoor](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview) or [Application Gateway](https://learn.microsoft.com/en-us/azure/application-gateway/overview) for this purpose (in some cases, both). If your API Management deployment was assisted by a Microsoft or partner team, you probably have one of them already working. If you don't, I highly encourage you to reconsider the design of your APIM instance to include WAF protection.
You can find the official Microsoft documentation:
- [Configure Front Door Standard/Premium in front of Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/front-door-api-management)
- [Protect APIs with Application Gateway and API Management](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/apis/protect-apis)

The solution proposed by this article for easy request tracing requires one of them. The whole point of it is that either FrontDoor or Application Gateway will be configured to temporarily insert the required HTTP headers for tracing towards the API Management backend. That means we don't need to do that directly from the client devices!

![frontdoor-apim](/images/frontdoor-architecture.png)

## The API Management tracing feature

In case you are not familiar what is APIM request tracing, I encourage you to read [this](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-api-inspector). 
In short, if you configure a Subscription key to allow request tracing and the incoming request in APIM contains a HTTP header named "**Ocp-Apim-Trace**", APIM response will contain another HTTP header named "**Ocp-Apim-Trace-Location**", which value will be the location of a JSON file that you can download in your browser and view all the inner details of what is happening inside APIM policies. This feature is very useful if you want to investigate any error happening in your APIs and you have a hard time to understand if it is because of a buggy policy, backend timeout, incoming HTTP request details, etc. Here's a sample of a trace JSON file that is generated:

![trace-location](/images/trace-location.png)

![trace-contents](/images/trace-contents.png)

You can either manually generate a trace request using the built-in Portal trace feature or by adding the headers yourself in a third-party tool like Postman. But the main problem doing this is that you need to reproduce the exact request contents that arrived in APIM and sometimes that is not easy to have if you don't have a full payload log for HTTP POST requests for example.
So, let's move on to the "best way" to do this!

## Configuring APIM to log the trace location file if it is present

Next step is to configure APIM to log the trace file if it was generated, so we can access the location of the JSON for a given request.
We have two ways to do this:
- [Enabling Azure Monitor](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor#modify-api-logging-settings)
- [Enabling Application Insights](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-app-insights)

I am choosing Application Insights for this article. By default, no response headers are logged in Application Insights for APIM after you enable it. Therefore, let's configure to log the "**Ocp-Apim-Trace-Location**" header in its responses:

![trace-settings](/images/trace-settings.png)

After that is done, all your Application Insights request tracing (check [here](https://learn.microsoft.com/en-us/azure/azure-monitor/app/asp-net-exceptions#diagnose-failures-using-the-azure-portal) for more details on how you can browse request errors in AppInsights for example)

![appinsights-trace](/images/appinsights-trace.png)

Note the "*Response-Ocp-Apim-Trace-Location*" property in the Application Insights attributes. It contains the location of the JSON trace file from APIM responses! You can go ahead and look in the file to investigate what went wrong with that request. That means, if tracing is enabled, it will be captured in the logs.

**Another important thing**: the APIM subscription must be enabled to allow tracing to be captured. More details [here](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-api-inspector#verify-allow-tracing-setting)

## Injecting the 'Ocp-Apim-Trace-Location' header from FrontDoor or Application Gateway

The next step is to configure the FrontDoor or Application Gateway to inject the required tracing header towards the APIM backend.
If you are using FrontDoor, just add this ruleset to your route that points APIM:

![ruleset](/images/ruleset.png)

This ruleset will make all requests to generate the trace file in APIM.
You can also play around with the "Condition" and apply this rule conditionally to some endpoints/clients only.

If you are using Application Gateway, a similar rule can be done. Check the documentation [here](https://learn.microsoft.com/en-us/azure/application-gateway/rewrite-http-headers-url).

## Conclusion

That's all we need to do! Every time we want to generate real request traces we add that rule in FrontDoor or Application Gateway. Also, make sure that the target subscription key is enable in APIM. By the way, APIM disable the tracing flag automatically for a given subscription after 1 hour. But even if does that for you, my recommendation is that you also remove the header injection even if the subscription has tracing disabled, because you don't want to unnecessarily have extra routing rules in your flow.

Going further, to do everything quicker you can create scripts using for example [Runbooks](https://learn.microsoft.com/en-us/azure/automation/automation-runbook-types?tabs=lps51%2Cpy27#powershell-runbooks) that will automatically add the routing rules and enable tracing in a subscription using Az CLI or Powershell and remove them after some time.