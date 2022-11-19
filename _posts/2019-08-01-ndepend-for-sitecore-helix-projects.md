---
layout: post
title: "NDepend for Sitecore Helix projects"
description: "How NDpend can help to keep track of Helix design principles."
date: "2019-08-01 +0100"
tags: [Sitecore, Helix, NDepend, Architecture]
image: /assets/images/posts/030/ndpend_logo.jpg
---
NDepend is a tool for static code analysis. It can tell how good or bad is the code. It can be very useful to keep track of the technical debt. It can be plugged into CI\CD pipeline to check code quality continuously. It can be also very useful for Sitecore (Helix) projects.

Helix is a set of overall design principles and conventions. Here is how we can use NDepend to keep track of them:

### Acyclic Dependencies Principle

> The dependency graph of packages must have no cycles.
>
> [http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)

NDepend already has two default rules for this: `Avoid namespace dependency cycles` and `Avoid namespaces mutually dependent`. Both rules are under Architecture group. Here are links to NDependent Rules Explorer website: [rule1](https://www.ndepend.com/default-rules/NDepend-Rules-Explorer.html?ruleid=ND1400#!) and [rule2](https://www.ndepend.com/default-rules/NDepend-Rules-Explorer.html?ruleid=ND1401#!).

![NDepend ADP rules](/assets/images/posts/030/ndepend_adp_rules.jpg)

To find dependency cycles you can also use NDepend Dependency Graph or Dependency Matrix views. Here is a nice video about it:

{% include youtube-embed.html url="https://www.youtube-nocookie.com/embed/nH-tB-HMNsE" %}

### Stable Dependency Principle

> The dependencies between packages should be in the direction of the stability of the packages. A package should only depend upon packages that are more stable than it is.
>
> [http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)

NDepend doesn’t have a default rule for this, but we can easily create it by ourselves:

``` cs
from project in JustMyCode.Assemblies
let moreStableAssembliesUsed = project.AssembliesUsed.Where(a => a.Instability > project.Instability)
where moreStableAssembliesUsed.Any()
select new {project, moreStableAssembliesUsed}
```

However, for Helix projects, a more useful can be a rule that will check dependency between layers. We can write a rule like this:

``` cs
from project in JustMyCode.Assemblies
where project.Name.Contains(".Foundation.") || project.Name.Contains(".Feature.")
let invalidReferences = project.AssembliesUsed.Where(a => a.Name.Contains(".Feature.") || a.Name.Contains(".Project."))
where invalidReferences.Any()
select new {project, invalidReferences}
```

To create a new rule, click *Create Rule* button then enter a query into Queries and Rules Edit window:

![NDepend create a rule](/assets/images/posts/030/ndepend_create_a_rule.jpg)

![NDepend QR edit](/assets/images/posts/030/ndepend_qr_edit.jpg)

### Stable Abstractions Principle

> Abstractness increases with stability.
>
> [http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)

NDepend has default rule for checking this. It’s called `Assemblies that don’t satisfy the Abstractness/Instability` principle under the Architecture group. And here is the link with the rule details: [link](https://www.ndepend.com/default-rules/NDepend-Rules-Explorer.html?ruleid=ND1407#!).

![NDepend SAP rule](/assets/images/posts/030/ndpend_sap_rule.jpg)

### More info

Of course, NDepend has much more [features](https://www.ndepend.com/features/). On the website, there are [walkthrough videos](https://www.ndepend.com/docs/videos) that explain features pretty good.