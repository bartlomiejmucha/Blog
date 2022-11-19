---
layout: post
title: "Application Insights Logs Exporter"
description: "Sometimes you need to export more than 10000 of logs from Application Insights. I wrote a tool that can help."
date: "2018-11-27 +0100"
tags: [Sitecore, Sitecore PowerShell, Azure, Application Insights]
image: /assets/images/posts/024/sitecore-power-shell-small.png
---
It happens from time to time that I have to export more than 10k logs from the **Application Insights**, especially when I have to provide the logs to Sitecore Support. As you may know, there is a limit of 10k rows that you can export at once in Application Insights UI. 

When using Application Insights REST API the limit is much higher and it's 500k of rows. The API returns logs with JSON so we have to somehow format it, and in case we need more than 500k of log lines, we have to do another request and then concatenate the results.

I wrote a tool that just does that. It has three versions:

- Standalone Windows application,
- Windows PowerShell script,
- Script for Sitecore PowerShell Extension.

### The Tool

To use the tool you have to generate REST API Key. Here is a [link to documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/api/overview) where you can read how to do it. To see the source code and installation instructions go to my [GitHub repository](https://github.com/bartlomiejmucha/Application-Insights-Logs-Exporter).

The standalone version of the tool:

![Standalone version of the tool](/assets/images/posts/024/standalone.jpg)

The SPE version of the tool:

![SPE version of the tool](/assets/images/posts/024/spe.jpg)
