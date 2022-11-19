---
layout: post
title: "The best SIF modification for local development"
description: "My custom SIF installation (and uninstallation) scripts for local development."
date: "2018-02-28 +0100"
tags: [Sitecore, Sitecore 9.0 Update 1, Square, Solr, Solr 6.6.2, SIF, SIF 1.2]
image:
---
If you are reading this, then the title did the job, especially this "the best" part :). There are many blog posts about SIF nowadays and many approaches to Sitecore 9 installation. That's why I thought that if I title this article "another SIF installation script" nobody would ever read it. So if I have your attention, then let me tell you why I wrote this post.

Some time ago I created a new repository on GitHub called [Square](https://github.com/bartlomiejmucha/Square). It is going to be a place with all my Sitecore ideas. It was shortly after Sitecore 9 release and I was playing with SIF framework so I started implementing installation script. First I copied a script from official documentation, but since then, the script evolved.

### How is it different than original one?

Compared to official scripts, Square installation script do additional things like:

- Installs Solr from zip file,
- Generates certificate for Solr,
- Configures Solr to work over HTTPS,
- Configures Solr to have custom hostname like https://square.solr,
- Installs Solr as a windows service with NSSM,
- Generates certificate for XConnect host,
- Configures XConnect host to work over HTTPS,
- Keeps clean installation folder structure with separate subfolders for Sitecore, XConnect, Solr and certificates.
- And if you don't need it or would like to reinstall, there is uninstall.ps1 script that removes everything from your PC including certificates, databases, hosts entries from window's host file, sites on IIS, Solr service, and all installation folders from disk.

It has been tested to work with Sitecore 9.0 update-1, SIF 1.2, Solr 6.6.2 and Windows 10.

After installation you will get root installation folder (like C:\Websites\Square) with subfolders structure like this:

- certificates subfolder like C:\Websites\Square\certificates, 
- xconnect subfolder like C:\Websites\Square\xconnect, 
- solr subfodler like C:\Websites\Square\solr,
- sitecore subfolder like C:\Websites\Square\sitecore.

### How to use it?

If you are interested, go to my [Square](https://github.com/bartlomiejmucha/Square) repository and read the README file. 

Happy Sitecoring!