---
layout: post
title: "How to resize and compress images (with Dianoga) during upload to Sitecore media library"
description: "I describe here I implemented functionality to resize and compress images during upload to media library."
date: "2017-11-14 +0100"
tags: [Sitecore, Sitecore 8.2 Update-3, Dianoga, Media Library, compression, resize]
image:
---
### A bit of theory, TL;DR;

Images optimisation is the hot topic for me recently. Last week I debugged one issue related to media cache: for URL without a `?mw` parameter, an image was always served from a database even if a cache file existed inside the `MediaCache` folder. Obviously this, with Dianoga module (where image optimisation for some large images can take a few seconds) and a large number of visitors, can lead to performance problems.

It turned out that we have a processor inside the `getMediaStream` pipeline that was intended to resize image during upload if the image width is greater than some specified max width. The issue was tough to spot because it really looked like it works correctly: the width and height fields of a media item were set correctly during upload, and when you downloaded the image you could not get it larger than a max width (even without any parameters in URL). But it obviously didn’t work because when I turned the processor off, cleared the media cache folder and download the image again, I got it in full-size. We spotted that issue only because we installed Dianoga and saw in logs that for some images Dianoga adds lines every time you request an image.

When you upload the image to media library then an `uiUpload` pipeline is executed. Inside that pipeline, there is a `Sitecore.Pipelines.Upload.Save` processor and inside that processor, for each uploaded image (single or unpacked form zip) the `MediaCreator.CreateFromStream` method is executed:

``` cs
public virtual Item CreateFromStream(Stream stream, string filePath, bool setStreamIfEmpty, MediaCreatorOptions options)
{
    Assert.ArgumentNotNull((object) stream, nameof (stream));
    Assert.ArgumentNotNullOrEmpty(filePath, nameof (filePath));
    Assert.ArgumentNotNull((object) options, nameof (options));
    string itemPath = this.GetItemPath(filePath, options);
    return this.AttachStreamToMediaItem(stream, itemPath, filePath, options);
}
```

It gets item path and then calls `MediaStream.AttachStreamToMediaItem`:

``` cs
public virtual Item AttachStreamToMediaItem(Stream stream, string itemPath, string fileName, MediaCreatorOptions options)
{
    Assert.ArgumentNotNull((object) stream, nameof (stream));
    Assert.ArgumentNotNullOrEmpty(fileName, nameof (fileName));
    Assert.ArgumentNotNull((object) options, nameof (options));
    Assert.ArgumentNotNull((object) itemPath, nameof (itemPath));
    Sitecore.Resources.Media.Media media = MediaManager.GetMedia((MediaItem) this.CreateItem(itemPath, fileName, options));
    media.SetStream(stream, FileUtil.GetExtension(fileName));
    return (Item) media.MediaData.MediaItem;
}
```

First, the media item is created then the `MediaStream.SetStreammethod` is executed:

``` cs
public virtual void SetStream(MediaStream mediaStream)
{
    Assert.ArgumentNotNull((object) mediaStream, nameof (mediaStream));
    this.MediaData.SetStream(mediaStream);
    this.UpdateMetaData(mediaStream);
}
```

The stream is set to the media item, and then metadata is updated inside `ImageMedia.UpdateMetaData` method:

``` cs
public override void UpdateMetaData(MediaStream mediaStream)
{
    base.UpdateMetaData(mediaStream);
    this.UpdateImageMetaData(mediaStream);
}
```

And finally `UpdateImageMetaData` looks like this:

``` cs
protected virtual void UpdateImageMetaData(MediaStream mediaStream)
{
    Assert.ArgumentNotNull((object) mediaStream, nameof (mediaStream));
    if (!mediaStream.AllowMemoryLoading)
    {
        Tracer.Error((object) "Could not update image meta data as the image is larger than the maximum size allowed for memory processing. Media item: {0}", (object) mediaStream.MediaItem.Path);
    }
    else
    {
        Item innerItem = this.MediaData.MediaItem.InnerItem;
        using (new EditContext(innerItem, SecurityCheck.Disable))
        {
            using (Image image = this.GetImage())
            {
                if (image == null)
                    return;
                innerItem["Width"] = image.Width.ToString();
                innerItem["Height"] = image.Height.ToString();
                innerItem["Dimensions"] = string.Format("{0} x {1}", (object) image.Width, (object) image.Height);
            }
        }
    }
}
```

As you can see the `UpdateImageMetaData` sets Width and Height of the image, but it does not use originally uploaded stream. Instead, it calls `GetImage` method which in the end tries to get an image from `MediaCache` folder or directly from Sitecore by calling the `getMediaStream` pipeline. That pipeline returned image transformed by MaxWidth parameter added by our custom processor I mentioned at the beginning. Width and Height fields were set from the size of the transformed image, not the original one. I looked correct during upload and when you requested image in the browser, but in the database, the stream with original size has been uploaded.

### A Solution

After a lot of digging into Sitecore assemblies with DotPeek, I decided to put my code into `MediaCreator.AttachStreamToMediaItem` method. `MediaCreator.CreateFromStream` would be better (because of the name and not only), but Dianoga processors use `MediaStream` and `MediaItem` objects, so we need `MediaItem` to be created first. So I created `CustomMediaCreator` class and put my code after `MediaItem` creation and before the stream is set onto the item:

``` cs
public class CustomMediaCreator : MediaCreator
{
    public override Item AttachStreamToMediaItem(Stream stream, string itemPath, string fileName, MediaCreatorOptions options)
    {
        Assert.ArgumentNotNull((object)stream, nameof(stream));
        Assert.ArgumentNotNullOrEmpty(fileName, nameof(fileName));
        Assert.ArgumentNotNull((object)options, nameof(options));
        Assert.ArgumentNotNull((object)itemPath, nameof(itemPath));

        var media = MediaManager.GetMedia((MediaItem)this.CreateItem(itemPath, fileName, options));

        // Customisation
        try
        {
            var mediaManager = ServiceLocator.ServiceProvider.GetRequiredService();

            var extension = FileUtil.GetExtension(fileName);
            var mimeType = mediaManager.MimeResolver.GetMimeType(extension);

            var args = new UploadMediaStreamPipelineArgs(stream, fileName, extension, mimeType, media.MediaData.MediaItem);
            CorePipeline.Run("uploadMediaStream", args);

            stream = args.Stream;
        }
        catch (Exception ex)
        {
            Log.Error("Could not run the 'uploadMediaStream' pipeline for '" + fileName, ex, this);
        }
        // Customisation ends here

        media.SetStream(stream, FileUtil.GetExtension(fileName));
        return (Item)media.MediaData.MediaItem;
    }
}
```

My customisation basically creates `UploadMediaStreamPipelineArgs` pipeline args object and then runs custom `uploadMediaStream` pipeline. Inside that pipeline, I decided to have two processors: the first one that resizes the image if it's greater than the maximum width and the second one that optimises the image with **Dianoga**. Configuration patch for media creator and pipeline looks like this:

``` xml
<sitecore>   
  <mediaLibrary>
    <mediaCreator type="Namespace.CustomMediaCreator, AssemblyName" />
  </mediaLibrary>
  <pipelines>
    <uploadMediaStream> <!-- custom pipeline -->
      <processor type="Namespace.UploadMediaStream.Resize, AssemblyName">
        <MaxWidth>1920</MaxWidth>
      </processor>
      <processor type="Namespace.UploadMediaStream.DianogaOptimize, AssemblyName " />
    </uploadMediaStream>
  </pipelines>
</sitecore>
```

Definition of the `UploadMediaStreamPipelineArgs` looks like this:

``` cs
[Serializable]
public class UploadMediaStreamPipelineArgs : PipelineArgs
{
    public Stream Stream { get; set; }
    public string FilePath { get; }
    public string Extension { get; }
    public string MimeType { get; }
    public MediaItem MediaItem { get; }

    public UploadMediaStreamPipelineArgs(Stream stream, string filePath, string extension, string mimeType, MediaItem mediaItem)
    {
        Stream = stream;
        FilePath = filePath;
        Extension = extension;
        MimeType = mimeType;
        MediaItem = mediaItem;
    }
}
```

The code of `Resize` processor is copied from `Sitecore.Resources.Media.ResizeProcessor` and modified a bit because the original one takes media stream directly from media item and at this stage that stream is not set yet:

``` cs
public class Resize
{
    public int MaxWidth { get; set; }

    public virtual void Process(UploadMediaStreamPipelineArgs args)
    {
        if (args.Stream != null && args.MimeType.StartsWith("image/", StringComparison.Ordinal) && args.Stream.CanRead && args.Stream.Length <= Settings.Media.MaxSizeInMemory)
        {
            if (args.Stream.CanSeek)
            {
                args.Stream.Seek(0L, SeekOrigin.Begin);
            }

            var mediaManager = ServiceLocator.ServiceProvider.GetRequiredService();

            var imageFormat = mediaManager.Config.GetImageFormat(args.Extension, null);
            if (imageFormat != null)
            {
                var transformationOptions = new TransformationOptions { MaxSize = new Size(MaxWidth, 0) };

                // input stream is closed inside TransformImageStream
                args.Stream = mediaManager.Effects.TransformImageStream(args.Stream, transformationOptions, imageFormat);
            }
        }
    }
}
```

And finally the `DianogaOptimize` processor. It creates a `MediaOptimizer` object and a `MediaStream` object and passes them into Process method. If the stream is returned, the input one is closed and replaced by optimised one. The code looks like this:

``` cs
public class DianogaOptimize
{
    public virtual void Process(UploadMediaStreamPipelineArgs args)
    {
        if (args.Stream != null && args.Stream.CanRead && args.Stream.Length <= Settings.Media.MaxSizeInMemory)
        {
            if (args.Stream.CanSeek)
            {
                args.Stream.Seek(0L, SeekOrigin.Begin);
            }

            var dianogaOptimizer = new MediaOptimizer();
            var mediaStream = new MediaStream(args.Stream, args.Extension, args.MediaItem);

            var optimizedOutputStream = dianogaOptimizer.Process(mediaStream, new MediaOptions { });
            if (optimizedOutputStream?.Stream != null)
            {
                args.Stream.Close();
                args.Stream = optimizedOutputStream.Stream;
            }
        }
    }
}
```

Then I did a print screen of my 4k monitor and saved it as a png file. It was around 5mb. After Resize to 1920 it was 3mb, and after `DianogaOptimize` compressed it lossless, it was 1.6MB. Quite nice results.