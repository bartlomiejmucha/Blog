---
layout: post
title: "Quantus - the library to handle plurals of nouns for Sitecore Dictionary and Habitat Dictionary"
description: "I created library that helps with handling plurals of nouns for different languages in Sitecore."
date: "2017-11-09 +0100"
tags: [Sitecore, Helix, Habitat, Quantus]
image:
---
### Why?

It happens very often. There is some static label on a website that displays a number of something like a number of products or number or results. It looks fine until someone finds out that it does not for some edge case. For example, your label is like this `{0} products` and it works fine for `1.5 products`, `10 products`, `15 products` or even for `101 products`, but for some reason, it does not work for `1 products`.

It's easy to fix for English. The one can add another field or dictionary entry `{0} product` and use it only when number equals `1`. However, for Polish, the rules are more complicated. There are `0 produktów`, `1 produkt`, `1,5 produktu`, `2-4 produkty`, `5-21 produktów`, `22-24 produkty` and so on. Each language has its own rules. There is no native implementation for this in Sitecore, that's why I created the Quantus library.

### How?

Fortunately, there is a page where rules for cardinal and ordinal numbering are described for all languages: [Language Plural Rules](https://www.unicode.org/cldr/cldr-aux/charts/25/supplemental/language_plural_rules.html). There are up to six plural categories for each language: `zero`, `one`, `two`, `few`, `many` and `other`. English, for example, uses only two of them: `one` and `other`. Polish uses `one`, `few`, `many` and `other`.

The rules for English cardinal numbering looks like this:

![English cardinal rules](/assets/images/posts/009/english-cardinal-rules.png)

`if number == 1 || number == -1` then use `one` category otherwise use `other` category. For Polish, rules are more complicated:

![Polish cardinal rules](/assets/images/posts/009/polish-cardinal-rules.png)

### The library

The library consists of two assemblies: Quantus and Quantus.Sitecore. The first one contains providers for different languages and the second one provides basic integration of the first one with Sitecore. At the moment library supports only a few languages, but the aim is to support all of them.

The provider is a very simple class that implements `IPluralProvider` interface, and for English, it looks like this:

``` cs
public class EnglishPluralProvider : ProviderBase, IPluralProvider
{
    public PluralCategory GetPluralCategory(decimal n)
    {
        return n == 1 || n == -1 ? PluralCategory.One : PluralCategory.Other;
    }
}
```

**Quantus.Sitecore** assembly implements a repository for providers and adds Sitecore config section:

``` xml
<quantus>
  <patch:attribute name="defaultProvider">fallback</patch:attribute>
  <providers>
    <add name="fallback" type="Quantus.Providers.OtherPluralProvider, Quantus" />
    <add name="en" type="Quantus.Providers.EnglishPluralProvider, Quantus" />
    ...
  </providers>
</quantus>
```

Provider name is two letter language iso code. Default `fallback` provider is used when there is no provider for the specified language, and it always returns `PluralCategory.Other`. The `PluralService` class has one method, and it returns plural category for a given language and quantity:

``` cs
PluralCategory GetPluralCategory(string language, decimal quantity);
```

### The Sitecore Dictionary

I don't want to force the way one can use that library. That's why helper methods are not included in **Quantus.Sitecore** assembly. I'm going to change that when I or the community figure out the best way to implement it. However here is an example how to use it with Sitecore Dictionary.

First create a `Dictionary Folder` (Month) and six `Dictionary Entries` (Month Zero, Month One …) like in the following picture (the item names don't matter):

![Sitecore dictionary example](/assets/images/posts/009/sitecore-dictionary-example.png)

For each dictionary entry set the key to `month + . + {category}` so for Month Zero set it to `month.zero`, for Month One: `month.one` etc. Then you can use following helper method to render correct translation:

``` cs
public string PluralText(string key, decimal quantity)
{
    var category = PluralService.GetPluralCategory(Context.Language.Name, quantity);
    return Sitecore.Globalization.Translate.Text(key + "." + category.ToString().ToLower());
}
```

And use it like this:

``` cs
PluralText("month", 10);
```

### The Habitat Dictionary

I also created an example integration of Quantus library with custom dictionary implementation in Habitat. I created a pull request for this here: [Pull request to Habitat repository](https://github.com/Sitecore/Habitat/pull/353). I created a new `DictionaryPluralEntry` template with fields like this:

![Habitat dictionary example](/assets/images/posts/009/habitat-dictionary-example.png)

I also created helper methods for Habitat dictionary. The example usage is in `SearchResultHeader.cshtml` view file:

``` csharp
@string.Format(Html.Sitecore().DictionaryPlural("/search/header/Title With Results", totalResults, "Your search for '{0}' yielded '{1}' results:"), Model.Context.Query, totalResults)
```

### Source code

Source code for both [Quantus](https://github.com/bartlomiejmucha/Quantus) and [Quantus.Sitecore](https://github.com/bartlomiejmucha/Quantus.Sitecore) is available on my [GitHub](https://github.com/bartlomiejmucha).

### What next?

The aim is to implement providers for all languages and add support for ordinal numbering. I would also like to figure out the best way to use that library in Sitecore with the community. And if you wish to contribute, **please do!**