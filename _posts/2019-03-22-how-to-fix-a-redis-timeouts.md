---
layout: post
title: "How to fix a Redis timeouts"
description: "I have seen Redis timeouts in all my projects. In this article I provide a list of most probably causes and solutions."
date: "2019-03-22 +0100"
tags: [Sitecore, Azure, Redis]
image:
---
If you have ever faced Redis timeouts issue, you probably know that there is no single reason for it. For this, you have to investigate and understand your particular situation. I have seen Redis timeouts in all my projects and every time it was a different cause and a different fix needed. In this article, I provide the most probable causes and solutions. I'm going to update this article if I find something new.

### 1. Burst of traffic

In my opinion, this one is the most common reason and it's related to the way how ThreadPool is implemented in .NET. The ThreadPool provides two types of threads: Worker and IOCP. Each has defined minimum. By default, the minimum is set to the number of processors on a system. So for example on Azure P2v2 which has two cores, the minimum is set to 2.

ThreadPool provides new threads on demand until it reaches a minimum. Then it will throttle the rate at which it injects new threads to one thread per 500 milliseconds. The StackExchange.Redis has dedicated internal pool of threads, and if it's failing to keep up, additional work will be offered to the global ThreadPool.

Redis exceptions contain useful information about a number of minimum and busy threads, for example:

```
Timeout performing EVAL, inst: 187, mgr: Inactive, err: never, queue: 74, qu: 1, qs: 73, qc: 0, wr: 0, wq: 0, in: 65536, ar: 0, IOCP: (Busy=1,Free=999,Min=2,Max=1000), WORKER: (Busy=80,Free=32687,Min=2,Max=32767), clientName: RD281878737F80
```

The number of busy worker threads is 80 and the minimum is set to 2. So 78 threads * 500 ms = 39s. 80th thread waited at least 39s to be created. This particular exception has been thrown because Redis operation waited too long for the free thread. Additionally, when a new thread is added to the ThreadPool, or existing one becomes free to process work, there is no guarantee that it will pick the Redis operation. It can pick the incoming ASP.NET request instead.

For all these reasons, when application gets burst of traffic that needs more threads than the minimum, Redis can hit timeouts.

#### Solution

We can change the minimum Worker and IOCP threads for the system. We can do that in Web.config under `<processModel>` according to [this page](https://docs.microsoft.com/en-us/previous-versions/dotnet/netframework-4.0/7w2sway1(v=vs.100)) but on Azure, we need to call `ThreadPool.SetMinThreads` API during application startup. In Sitecore, the best place is to put a processor to the `<initialization>` pipeline.

Fortunately, Sitecore provided a [patch](https://kb.sitecore.net/articles/464570) with a processor that allows configuring these values. It also reduces the max size of ThreadPool to prevent CPU over-usage and it limits the max concurrent requests at one time, to prevent ThreadPool clogging.

#### What values are correct?

The Microsoft documentation recommends to set a minimum to 200 or 300 threads and then test and tweak as needed. These values are split across all cores. On the other hand, Sitecore's patch set's the minimum to 15 Worker threads per core and 10 IOCP threads per core. My preference is to use Sitecore's default and then tweak as needed.

### 2. High CPU

If your CD server has periods of time with CPU around 100% utilization, then there could be not enough CPU power to process Redis threads in those periods of time. In one of my projects where we use Dianoga with jpegoptim processor, and at that time we were at S2 plan, the CPU during image optimization with jpegoptim went to 100% for even 60s for some very large, high-quality images. After looking at logs and reading metrics, we have found a correlation between high CPU, image optimization and Redis timeouts.

#### Solution

In our case, we scaled up the plan from S2 to P1V2 (which costs the same amount of money) and our problems disappeared. The P1V2 has only one core, but the CPU is much better, so it handles image optimization much more quickly.

### 3. Bandwidth limit on Redis

If you transfer a large number of data to and from Redis you can hit a bandwidth limit. Microsoft provides a [nice table](https://docs.microsoft.com/en-us/azure/azure-cache-for-redis/cache-faq#cache-performance) with observed bandwidth limit on a various pricing tiers. To see how many data is transferred to and from Redis, browse to your cache instance in the Azure portal, go to the Metrics blade and add two metrics: Cache Read and Cache Write with Sum aggregation.

![Redis cache read and write](/assets/images/posts/027/redis_cache_read_and_write.jpg)

#### Solution

The obvious solution is to scale your Redis up to have more bandwidth. Another solution is to review your application and check if you can store fewer data in Redis.

#### Links

- [https://kb.sitecore.net/articles/464570](https://kb.sitecore.net/articles/464570)
- [https://docs.microsoft.com/en-us/azure/azure-cache-for-redis/cache-faq#recommendation](https://docs.microsoft.com/en-us/azure/azure-cache-for-redis/cache-faq#recommendation)
- [https://github.com/StackExchange/StackExchange.Redis/blob/master/docs/Timeouts.md](https://github.com/StackExchange/StackExchange.Redis/blob/master/docs/Timeouts.md)
- [https://docs.microsoft.com/en-us/azure/azure-cache-for-redis/cache-faq#cache-performance](https://docs.microsoft.com/en-us/azure/azure-cache-for-redis/cache-faq#cache-performance)

### 4. RPS limit on Redis

I described it in [another article]({{ site.baseurl }}{% post_url 2020-02-17-are-you-hitting-redis-requests-per-second-rps-limit %}).