---
layout: post
title: "Memory optimisation in Sitecore"
description: "How to lower memory consumption in Sitecore."
date: "2018-01-22 +0100"
tags: [Sitecore, Sitecore 8.2 Update-3, Memory, Internig, Performance]
image: /assets/images/posts/012/memory-consumption-over-time-small.jpg
---
If you host your application on 64 bits environment and if you have a lot of RAM then you can consider disabling cache size limits by changing this setting:

``` xml
<!--  CACHING - DISABLE CACHE SIZE LIMITS
      If true, Sitecore does not limit cache size growth and ignores any maximum cache sizes 
      specified in the web.config file.            
      Enabling this setting can improve the application's performance in 64-bit environments 
      by allowing Sitecore to take full advantage of the available memory.
      After setting this value to true, monitor the system at regular intervals, as this 
      configuration can cause Sitecore to consume too much memory and cause Out Of Memory errors.
      It is only recommended to set the setting to true in 64-bit environments.
      Default value: false
-->
<setting name="Caching.DisableCacheSizeLimits" value="false" />
```

Without cache size limits the memory consumption will grow higher and higher over time. How high it will go depends on the size of your solution (mainly number of items), so your memory diagram can look something like this:

![Memory consumption over time](/assets/images/posts/012/memory-consumption-over-time.jpg)

While changing this setting can improve your application's performance, on the other hand, you should monitor your system to prevent Sitecore from consuming too much memory. New stuff will be added to the cache, but the old one will not be removed, so at some point, you can run out of free RAM and get `OutOfMemory` errors. In the worst case, your application can be restarted.

**Proactive AutoHeal on Azure:**
Your application can be automatically restarted by Azure Platform if your application's process’ private bytes exceeds 90% of the limit for over 30 seconds. [More info](https://learn.microsoft.com/en-us/azure/app-service/overview-diagnostics#auto-healing)

Following is the real world example from one of my projects. I restarted the application, started smart publish of the whole content tree and at the end more than 1M elements have been published. Memory consumption reached 12GB. Sometimes it can go even higher and then Azure restarts the application:

![Memory consumption after smart publish](/assets/images/posts/012/memory-consumption-after-smart-publish.jpg)

Here is another print screen from dotMemory. Snapshots have been taken after application restart but before smart publish, then after 53k, 104k, 151k published elements. As you can see the number of objects stored in memory increase about 4,5M for every 50k published elements:

![Memory snapshot comparison from dotmemory](/assets/images/posts/012/memory-snapshot-comparison-from-dot-memory.jpg)

Below is the screenshot of the details of "after 151k" snapshot. The memory is used mostly by three `SqlServerDataProvider` objects. Those three objects are for three Sitecore databases: core, master and web. There is also a lot of wasted memory by duplicated string values. For example, if you have 1M product items and half of them is in Approved workflow state and other half is in Draft state, then the IDs of those two workflow states can be duplicated by 0,5M times for each one.

![Memory snapshot details](/assets/images/posts/012/memory-snapshot-details.jpg)

Fortunately, there are out of the box ways to improve memory consumption in **Sitecore**.

### #1 Enable Sitecore.Interning

Following setting enables interning mechanisms that should reduce memory consumption. This is done by reusing immutable objects like strings or IDs instead of creating new ones:

``` xml
<!-- INTERNING ENABLED
     If enabled, Sitecore would re-use same immutable object instances, and enable InternManager<T> API.
     This can reduce memory consumption, and simplify Garbage Collection.
     The tradeoff is additional CPU cost of putting an object to intern pool.         
     Default value: false.
-->
<setting name="Interning.Enabled" value="false"/>
```

Additionally, you can enable following setting, to also intern field values:

``` xml
<!-- INTERNING KNOWN FIELD VALUES 
     If enabled, Sitecore would ensure to use intern pool for the known field values. 

     EXAMPLE: Every item under workflow would have one among known limited values.
     Since a field value is cached as string, a lot of duplicated strings represeting same workflow would present in memory.

     This setting is useful on large solutions where memory consumption is high.
     Default value: false.
-->
<setting name="Interning.InternKnownFieldValues" value="false"/>
```

If you enable it, Sitecore will try to reuse values for fields from the following list. Of course, you can add your own fields here.

``` xml
<fieldIdsToIntern>
    <workflowState>{3E431DE1-525E-47A3-B6B0-1CCBEC3A8C98}</workflowState>
    <workflow>{A4F985D9-98B3-4B52-AAAF-4344F6E747C6}</workflow>
    <updatedBy>{BADD9CF9-53E0-4D0C-BCC0-2D784C282F6A}</updatedBy>
    <createdBy>{5DD74568-4D4B-44C1-B513-0AF5F4CDA34F}</createdBy>
    <neverPublish>{9135200A-5626-4DD8-AB9D-D665B8C11748}</neverPublish>
    <mimeType>6F47A0A5-9C94-4B48-ABEB-42D38DEF6054</mimeType>
    <owner>{52807595-0F8F-4B20-8D2A-CB71D28C6103}</owner>
    <type>{AB162CC0-DC80-4ABF-8871-998EE5D7BA32}</type>
    <isShared>{BE351A73-FCB0-4213-93FA-C302D8AB4F51}</isShared>
    <isUnversioned>{39847666-389D-409B-95BD-F2016F11EED5}</isUnversioned>
    <sortOrder>{BA3F86A2-4A1C-4D78-B63D-91C2779C1B5E}</sortOrder>
</fieldIdsToIntern>
```

Getting back to my real world example, this how memory consumption chart looked like after I enabled those two settings. As you can see the memory consumption is lower by about 4GB this time.

![Memory consumption after smart publish with interning enabled](/assets/images/posts/012/memory-consumption-after-smart-publish-with-interning-enabled.jpg)

And this is how it looks in dotMemory. After 150k it has 9M objects stored in memory less than before:

![Memory snapshot comparison from dotmemory with intrning enabled](/assets/images/posts/012/memory-snapshot-comparison-from-dot-memory-with-interning-enabled.jpg)

The interning mechanism is very fast so you shouldn't see any performance issues when you enabled it. The price we pay is one `ConcurrentDictionary` lookup per field and it is not even visible during the dotTrace profiling. I also heard that Sitecore considers including interning by default for future versions of product.

### #2 Use MemoryHealthMonitor

If you enabled interning mechanism and you still worry about memory consumption you can configure `MemoryHealthMonitor`. In case that your memory consumption exceeds defined Threshold memory monitor can clear your Sitecore cache and then force garbage collection. It is better than application restart, right?

This is how your patch can looks like:

``` xml
<hook type="Sitecore.Diagnostics.MemoryMonitorHook, Sitecore.Kernel">
    <param desc="Threshold">10GB</param>
    <param desc="Check interval">00:01:00</param>
    <param desc="Minimum time between log entries">00:00:20</param>
    <ClearCaches>true</ClearCaches>
    <GarbageCollect>true</GarbageCollect>
    <AdjustLoadFactor>false</AdjustLoadFactor>
</hook>
```

Remember that `MemoryHealthMonitor` works only if you enable Performance Counters, but you can easly customize this behaviour.

Happy Sitecoring!