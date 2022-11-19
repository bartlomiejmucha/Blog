---
layout: post
title: "Are you hitting Redis requests per second (RPS) limit?"
description: "In this article I show how to check if you are hitting Redis RPS limit."
date: "2020-02-17 +0100"
tags: [Sitecore, Azure, Redis]
image:
---
Azure [documentation for Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-faq#azure-cache-for-redis-performance) contains a table with maximum bandwidth values observed for various sizes of the cache. It was collected using *redis-benchmark.exe* with 1-KB value size like it's described [here](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-faq#how-can-i-benchmark-and-test-the-performance-of-my-cache). So for example, for size C1 it is 38,000 RPS Non-SSL and 20,720 RPS SSL.

These values are not hard upper limit, rather an estimation of what the underlying VM can handle. Meaning that once you have hit your limit, you are not automatically throttled or bound to it. It does mean that there is no guarantee high latency beyond that number of requests and any issues you have can escalate exponentially the more requests you perform beyond this limit.

Also if your average key size is higher than 1-KB then Redis can handle fewer requests. Unfortunately, Microsoft does not provide numbers, but I did my own quick test using *redis-benchmark.exe* for different key values. I executed a benchmark only once for each key size and type (GET, SET) using Redis C1. Here are the results for SSL:

Size | SET | GET
---- | --- | ---
1-KB | 22616 | 27656
10-KB | 2893 | 2758
20-KB | 1385 | 1384
30-KB | 940 | 932
40-KB | 699 | 698
{: .table .table-bordered .table-striped }

The test results show that the higher key size, the fewer requests Redis can handle. The results show that on average if the key size is 10 times higher then the request limit is 10 times smaller. In other words, you can divide 1-KB RPS provided by Microsoft in the table by your average key size and you will get a number of requests for that key size.

### How to check if you are hitting the RPS limit?

Azure Redis provides a metric that we can use for that. First, let's pick some point in time when you experienced some timeouts or just a high number of *Total Operations*.

I had such a situation recently. Below images are from real project:

![Redis RPS total operations](/assets/images/posts/035/redis-rps-total-operations.png)

The chart displays **Total Operations (Sum)** and **Operations Per Second (Max)** with 1-minute aggregation. For this particular point in time, there were 69430 operations (in one minute). If we divide it by 60 seconds we will get 1157 operations per second on average for that particular minute. Operations Per Second (Max) is 1220. So these two metrics show quite the same number of requests per second.

Now let's calculate the average key size. To do this we can use **Used Memory (Max)** and **Total Keys (Max)** metrics.

![Redis RPS used memory](/assets/images/posts/035/redis-rps-used-memory.png)

Used Memory (Max) is 247.9 MB. Multiply that by 1024 to get value in KB: 247.9 * 1024 = 253849 KB. Now divide it by Total Keys (Max): 253849 / 8190 = 31 KB.

As you can see the average key size is 31KB and we had 1157 operations on average in that minute.

Are we hitting the Redis limit? Yes, because as you can see in the table above, for 30-KB key size Redis is capable of handling 940 RPS.