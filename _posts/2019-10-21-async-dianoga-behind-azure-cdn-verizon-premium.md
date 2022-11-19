---
layout: post
title: "Async Dianoga behind Azure CDN (Verizon Premium)"
description: "How to configure Dianoga to compress images asynchronously behind CDN."
date: "2019-10-21 +0100"
tags: [Sitecore, Azure, CDN, Compression, Dianoga]
image:
---
Dianoga documentation says that behind CDN the optimized version of the image must be sent immediately (synchronously). That makes sense because you want the optimized version to be cached by CDN and then served to the front end users.

However, asynchronous optimization has a huge advantage: front end visitors will get the uncompressed version of the image immediately, while the high intensive optimization process can be run in the background, and then after optimization is completed, new requests will get a compressed version of the image.

This is especially beneficial if you run your website on slow servers. I saw in one project which was run on Azure Standard (S2) tier, that Dianoga compression killed the server after fresh deployment. The server was unresponsive for 30 to 60 minutes. It was because of a slow CPU. The compression of single image took sometimes a few minutes. After scaling up to Premium (P1) tier it was much much better. That's why it is beneficial to store MediaCache in external location shared between slots or copy MediaCache from production slot to deployment slot in release pipeline as I described [here]({{ site.baseurl }}{% post_url 2019-01-23-how-to-copy-mediacache-from-prod-to-stage-slot-during-deployment-with-azure-devops %}).

A few months back I configured Azure CDN (Verizon Premium) for a website and managed to set up asynchronous Dianoga behind it.

The idea is to send the uncompressed version of the image immediately but set the max-age header to 2 minutes, then kick the optimization in the background and when the optimized version of the image is available, serve it with correct max-age (in my case it was 30 days) for the next requests.

The CDN will store the image internally according to the max-age set on the origin server. So the first request will result in storing an uncompressed version of the image for 2 minutes and CDN will serve that image to the front end users.

It is very important to override max-age for the image on the CDN level to always serve it with the correct value (30 days) no matter if it's compressed or uncompressed because if the user already downloaded the image, there is no point to redownload compressed version again.

After 2 minutes, when a new request for the image comes, the CDN will revalidate its internal cache. It will do a request to the origin server again. If optimization is completed on the origin server, this time max-age should be set to a higher value (in my case it was 30 days).

### How to do it?

To achieve this result, you need to do two things. First, you have to override `DoProcessRequest` method of `MediaRequstHandler` like this:

``` cs
public class DianogaMediaRequestHandler : Sitecore.Resources.Media.MediaRequestHandler
{
    protected virtual bool DoProcessRequest(HttpContext context, MediaRequest request, Sitecore.Resources.Media.Media media)
    {
        Assert.ArgumentNotNull((object)context, nameof(context));
        Assert.ArgumentNotNull((object)request, nameof(request));
        Assert.ArgumentNotNull((object)media, nameof(media));
        if (this.Modified(context, media, request.Options) == Tristate.False)
        {
            this.RaiseEvent("media:request", request);
            this.SendMediaHeaders(media, context);
            context.Response.StatusCode = 304;
            return true;
        }
        this.ProcessImageDimensions(request, media);
        MediaStream mediaStream = this.GetMediaStream(media, request);
        if (mediaStream == null)
            return false;
        this.RaiseEvent("media:request", request);
        if (Settings.Media.EnableRangeRetrievalRequest && Settings.Media.CachingEnabled)
        {
            using (mediaStream)
            {
                this.SendTempOrCorrectMediaHeaders(mediaStream, media, context);

                new RangeRetrievalResponse(RangeRetrievalRequest.BuildRequest(context, media), mediaStream).ExecuteRequest(context);
                return true;
            }
        }
        else
        {
            this.SendTempOrCorrectMediaHeaders(mediaStream, media, context);

            this.SendStreamHeaders(mediaStream, context);
            using (mediaStream)
            {
                context.Response.AddHeader("Content-Length", mediaStream.Stream.Length.ToString());
                WebUtil.TransmitStream(mediaStream.Stream, context.Response, Settings.Media.StreamBufferSize);
            }
            return true;
        }
    }
    
    protected virtual void SendTempOrCorrectMediaHeaders(MediaStream mediaStream, Sitecore.Resources.Media.Media media, HttpContext context)
    {
        // if it's memory stream, that means the Dianoga has not compressed image yet
        // 
        if (mediaStream.Stream is MemoryStream)
        {
            this.SendTemporaryMediaHeaders(media, (HttpContextBase)new HttpContextWrapper(context));
        }
        else
        {
            this.SendMediaHeaders(media, context);
        }
    }
    
    protected virtual void SendTemporaryMediaHeaders(Sitecore.Resources.Media.Media media, HttpContextBase context)
    {
        TimeSpan delta = TimeSpan.FromMinutes(2);

        DateTime date = media.MediaData.Updated;
        if (date > DateTime.UtcNow)
            date = DateTime.UtcNow;
        HttpCachePolicyBase cache = context.Response.Cache;
        cache.SetLastModified(date - delta);
        cache.SetETag(media.MediaData.MediaId + "_temp");
        cache.SetCacheability(Settings.MediaResponse.Cacheability);
        //TimeSpan delta = Settings.MediaResponse.MaxAge;
        if (delta > TimeSpan.Zero)
        {
            if (delta > TimeSpan.FromDays(365.0))
                delta = TimeSpan.FromDays(365.0);
            cache.SetMaxAge(delta);
            cache.SetExpires(DateTime.UtcNow + delta);
        }
        Tristate slidingExpiration = Settings.MediaResponse.SlidingExpiration;
        if (slidingExpiration != Tristate.Undefined)
            cache.SetSlidingExpiration(slidingExpiration == Tristate.True);
        string cacheExtensions = Settings.MediaResponse.CacheExtensions;
        if (cacheExtensions.Length > 0)
            cache.AppendCacheExtension(cacheExtensions);
        string varyHeader = this.GetVaryHeader(media, context);
        if (string.IsNullOrEmpty(varyHeader))
            return;
        context.Response.AppendHeader("vary", varyHeader);
    }
}
```

Customization is bolded. The only change in `DoProcessRequest` method is the replacement of `SetMediaHeaders` to `SendTempOrCorrectMediaHeaders` in two places. The `SendTemporaryMediaHeaders` method works similar to `SetMediaHeaders` but it sets temporary values to *Last-Modified*, *ETag*, *Max-Age* and *Expires* headers.

Verizon CDN should work quite fine out of the box. I decided to use ADN so I had to add two rules in Rules Engine. The first rule enables caching for `/-/media/*` urls because by default cache is disabled for ADN:

![Rule 1](/assets/images/posts/033/rule-1.jpg)

 And the second rule sets Max-Age to 30 days on edge servers, so users will always get the correct Max-Age:

 ![Rule 2](/assets/images/posts/033/rule-2.jpg)

 That's all folks. If you have any questions find me on twitter. Cheers