---
layout: post
title: "Custom Application Insights Adaptive Sampling Processor for Sitecore"
description: "There are some log lines that you would like to store every time, without any reduction, sampling. For example AUDIT logs. It's useful to check AUDIT logs to see what happened in the system, who edited a particular item and when. In this article, I will show you how to create a custom AI adaptive sampling processor that will keep all AUDIT logs."
date: "2019-09-19 +0100"
tags: [Sitecore, Azure, Application Insights, Telemetry]
image:
---
On Azure we have three [types of sampling](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling):

- Adaptive sampling
- Fixed-rate sampling
- Ingestion sampling

In my opinion, Adaptive sampling is the best. The volume of telemetry sent from a website to the Application Insights is automatically adjusted to keep within a specified maximum rate of traffic and it is controlled via the `MaxTelemetryItemsPerSecond` setting.

To achieve the target volume, some of the generated telemetry is discarded. And this can be a problem if we don't want to reduce some type of logs like **AUDIT** logs.

Fortunately, it's easy enough to write a custom telemetry processor for adaptive sampling that will store all AUDIT log lines. Here is the source code of the processor:

``` cs
public class SitecoreAdaptiveSamplingTelemetryProcessor : AdaptiveSamplingTelemetryProcessor, ITelemetryProcessor
{
    private ITelemetryProcessor Next { get; set; }

    public SitecoreAdaptiveSamplingTelemetryProcessor(ITelemetryProcessor next) : base(next)
    {
        this.Next = next;
    }

    public SitecoreAdaptiveSamplingTelemetryProcessor(SamplingPercentageEstimatorSettings settings, AdaptiveSamplingPercentageEvaluatedCallback callback, ITelemetryProcessor next) : base(settings, callback, next)
    {
        this.Next = next;
    }

    public new void Process(ITelemetry item)
    {
        if (item is TraceTelemetry trace)
        {
            if (trace.Message.Contains("AUDIT"))
            {
                this.Next.Process(item);
                return;
            }
        }
        
        base.Process(item);
    }
}
```

It inherits from original `AdaptiveSamplingTelemetryProcessor`. You also need to specify `ITelemetryProcessor` interface. Without it won't work.

The `Process` method first checks if telemetry item is of `TraceTelemetry` type, then if it contains **AUDIT** in the message. If yes it goes to the next processor in the pipeline. If no, it executes the original adaptive sampling code.

The last thing we need to do is to remove original `AdaptiveSamplingTelemetryProcessor` from *ApplicationInsights.config* file and insert our own. We can do that using transform file:

``` xml
<ApplicationInsights xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform" xmlns="http://schemas.microsoft.com/ApplicationInsights/2013/Settings">
  <TelemetryProcessors>
    <Add Type="Microsoft.ApplicationInsights.WindowsServer.TelemetryChannel.AdaptiveSamplingTelemetryProcessor, Microsoft.AI.ServerTelemetryChannel" xdt:Transform="Remove" xdt:Locator="Match(Type)"/>
    <Add Type="Example.Portal.Website.Azure.AI.SitecoreAdaptiveSamplingTelemetryProcessor, Example.Portal.Website" xdt:Transform="Insert">
      <MaxTelemetryItemsPerSecond>3</MaxTelemetryItemsPerSecond>
    </Add>
  </TelemetryProcessors>
</ApplicationInsights>
```

To make sure it works correctly, you can execute this query in the Application Insights Analytics:

```
traces
| where message contains "AUDIT" 
| where itemCount > 1
```

If `itemCount` is greater than `1` it means that log line has been sampled. The query above should return no results.