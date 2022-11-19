---
layout: post
title: "How to organize your Sitecore custom configuration files"
description: "See how to organize your Sitecore custom configration files."
date: "2016-08-24 +0100"
tags: [Configuration Files, Sitecore]
image: /assets/images/posts/004/appconfig-small.png
---
Do you remember times when clean Sitecore installation had only a few config files inside Include folder?
For example Sitecore 6.4 rev. 120113 has only 10 files and only one file has a *.config extension so only one will be processed.

![Sitecore 6.4 rev. 120113 include folder](/assets/images/posts/004/sitecore-6.4-rev.-120113-include-folder.png)

For comparison: Include folder in the latest version of Sitecore: 8.1 upd3 has 189 files (including files in subfolders).

### Why is this a problem?

You probably experienced an issue where your custom configuration was overwritten by default Sitecore files. You probably fixed that by adding "Z." prefix to your file name and then your file was processed as a last one.

By default Sitecore process configuration files in alphabetical order. Next, it processes subfolders in alphabetical order. Files inside subfolders are also processed in alphabetical order.

![Sitecore patch files load order](/assets/images/posts/004/sitecore-patch-files-load-order.png)

### Benefits of correct config organisation

When you organise your custom configuration files in the right way it will give you two advantages:

- It will be easier to upgrade project in the feature because all your custom configurations will be placed in your custom files. You will not change any of original Sitecore files so you will be able to just copy and paste them from installation zip during the upgrade. No need to manually apply all config changes.
- Your configuration will be processed in correct order. Ideally, we want the default Sitecore configuration files (and all default module files) to be processed first, and then apply our customisations on top of this.

### How I organized my config files

Recently I was working on upgrading project to latest Sitecore version and one of the first thing that I did was moving all config customisations to separate files.

First I created my subfolder:

![My z.WebsiteConfig folder](/assets/images/posts/004/sitecore-my-zwebsiteconfig-folder.png)

I called it `Z.WebsiteConfig`. Thanks to this files from inside that folder will be processed as a last one. All default Sitecore config files (module's files too) will be processed first. And my config files will add customisations on top of this.Inside Z.WebsiteConfig folder I tried to recreated structure similar to default Sitecore config structure.

![Content of my own custom configuration subfolder](/assets/images/posts/004/content-of-my-own-custom-config-folder.png)

My aim is to have the individual file that corresponds to the original Sitecore file and add customisations to the configuration from inside that original file. So in the example above I have two folders and three files:

#### Commands.config
It corresponds to original Commands.config file. I added my custom command here.

#### Sitecore.config
It also corresponds to original Sitecore.config file. For example, I added a patch for InstanceName here

#### Prefetch folder

I added patches with my custom items to prefetch here.

![Content of my custom prefetch folder](/assets/images/posts/004/my-content-prefetch-folder.png)

#### Include folder

Inside Include folder I have following files:

![Inside include folder](/assets/images/posts/004/inside-include-folder.png)

And inside Dianoga folder I have:

![Inside include folder](/assets/images/posts/004/content-of-my-custom-dianoga-folder.png)

Again, all these files correspond to original Sitecore files or module files.
We can, of course, have files that do not correspond to any original file, for example, our custom index definition.

### A few additional files
At the end I also added a few additional files:

![Inside include folder](/assets/images/posts/004/additional-files.png)

#### AppSettings.config

This file does not correspond to Sitecore config file but to ASP.NET config file. I have my custom Sitecore settings here. I think it's a good idea to use Sitecore settings instead of asp.net ones because you can overwrite them with patch file if, for example, some settings should be disabled on developer's machine.

#### Sitecore.Environment.config

Here I have settings that should be changed based on hosting environment. For example, website hostname is different on production and different on developers machine.

#### Z.Developer folder

This is the last piece. Files inside that folder will be parsed at the end so you can override any setting with that file. This files should not be deployed to live servers. This folder looks like this:

![Inside include folder](/assets/images/posts/004/inside-zdevelopers-folder.png)

I have two files here. First one is disabling scheduling at all because I don't need to run all jobs on my developer machine and the second file does some configuration changes that improve performance. I got that file from here: Sitecore 8.0/8.1 Performance Config

### All is on GitHub

That's right. I added all files to repo here for you: [Sitecore Custom Configuration Structure][my-blog].

[my-blog]: https://github.com/bartlomiejmucha/blog