---
layout: post
title: "Dont forget to clear your session storage during Sitecore upgrade"
description: "A short story how to fix ERROR ProcessExpiredItems => System.Runtime.Serialization.SerializationException: Cannot get the member '<.ctor>b__2'. excetpion."
date: "2019-04-27 +0100"
tags: [Sitecore, Sitecore 8.2 Update-7, Azure, Redis, Session]
image:
---
Recently I have been helping with the migration of one project to Azure. In the same time, the project has been upgraded to Sitecore 8.2 Update-1 and then again 8.2 Update-7. Before we deployed our site to Azure, the clean version of Sitecore 8.2 Update-1 was installed there.

After deployment to staging and production, we noticed the huge number of exceptions like this:

``` cs
ERROR ProcessExpiredItems => System.Runtime.Serialization.SerializationException: Cannot get the member 'b__2'.
   at System.Reflection.MemberInfoSerializationHolder.GetRealObject(StreamingContext context)
   at System.Runtime.Serialization.ObjectManager.ResolveObjectReference(ObjectHolder holder)
   at System.Runtime.Serialization.ObjectManager.DoFixups()
   at System.Runtime.Serialization.Formatters.Binary.ObjectReader.Deserialize(HeaderHandler handler, __BinaryParser serParser, Boolean fCheck, Boolean isCrossAppDomain, IMethodCallMessage methodCallMessage)
   at System.Runtime.Serialization.Formatters.Binary.BinaryFormatter.Deserialize(Stream serializationStream, HeaderHandler handler, Boolean fCheck, Boolean isCrossAppDomain, IMethodCallMessage methodCallMessage)
   at Sitecore.SessionProvider.Redis.RedisUtility.GetObjectFromBytes(Byte[] dataAsBytes)
   at Sitecore.SessionProvider.Redis.StackExchangeClientConnection.GetSessionDataStatic(Object rowDataFromRedis)
   at Sitecore.SessionProvider.Redis.RedisConnectionWrapper.TakeWriteLockAndGetExpiredData(String sessionId, DateTime signalTime, DateTime lockTime, Int32 expirationTimeout, ISessionStateItemCollection& data, Int32& sessionTimeout)
   at Sitecore.SessionProvider.Redis.RedisSessionStateProvider.OnProcessExpiredItems(DateTime signalTime)
```

It took me a full day of work to figure out what class exactly is to be deserialized. I had to implement a custom Redis provider and a lot of C# Reflection, but at the end of the day I saw this:

![Reflected type](/assets/images/posts/028/reflected-type.jpg)

I reached to Sitecore support and they quickly provided me a solution. The reason for the exceptions was 'old' sessions created by the earlier version of Sitecore (8.2 u1) that cannot be processed by the new version (8.2 u7). The solution for this is to just clear the session storage. It is mentioned in the [Sitecore upgrade guide](https://dev.sitecore.net/~/media/B50CA65AA6844B4B81BF36A01E9DD269.ashx) at the topic '1.4.7 Upgrade the Session Database'.

Alex form Sitecore support gave me the following steps to clear Redis session storage on Azure:

- Stop all Sitecore XP instances that use the same session database
- Clear the Redis: this can be done by using Redis Cache console - Please note that the unprocessed data (sessions added by Sitecore 8.2 u7) will be cleaned from the Redis Cache
    - In the Azure go to the Redis Cache and click "Console" option. 
    - Execute flushall command to clean up all records from the session store; [https://stackoverflow.com/questions/37227696/clear-azure-redis-cache](https://stackoverflow.com/questions/37227696/clear-azure-redis-cache)
- Start all XP instances.

Here is a [CustomRedisProvider.cs](https://gist.github.com/bartlomiejmucha/4e979d7d22b72da7358c062a245b8ec1) that I used for debugging.