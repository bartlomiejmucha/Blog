---
layout: post
title: "Unexpected side effect of the EditContext"
description: "There is one unexpected side effect of the EditContext that I didn't know about. Did you?"
date: "2018-08-13 +0100"
tags: [Sitecore, Sitecore 9.0 Update-2]
image:
---
Let's say you have two languages configured in Sitecore: EN and DE. Your `Product` template has versioned field `Title` and shared field `SKU`. The product can have only EN version, only DE version or both.

Imagine that you have a product with only DE version and then for some reason you executed below code to change SKU. It's shared field, so it shouldn't matter if you edit it on the EN or DE language, right?.

``` cs
using (new LanguageSwitcher("en"))
using (new EditContext(item))
{
    item["SKU"] = newSku;
}
```

What do you think will happen?

### Let's dig into the code.

The `SaveItem` method in the `ItemProvider` class looks like this:

``` cs
public override bool SaveItem(Item item)
{
    Assert.ArgumentNotNull((object) item, nameof (item));
    if (!item.Modified)
        return false;
    if (item.RuntimeSettings.Temporary)
        return true;
    this.EnsureVersion(item);
    item.Statistics.UpdateRevision();
    if (!this.GetDataEngine(item.Database).SaveItem(item))
        return false;
    this.UpdateDependantRevisions(item);
    item.Reload();
    return true;
}
```

Yep, it calls the `EnsureVersion` method to make sure that there is at least one version of the item in the current language.

``` cs
private void EnsureVersion(Item item)
{
    if (item.Versions.Count > 0)
        return;
    ItemChanges changes = item.GetChanges(false);
    if (changes == null || changes.FieldChanges.Count == 0)
        return;
    bool flag = false;
    foreach (FieldChange fieldChange in changes.FieldChanges)
    {
        if (fieldChange.Definition == null || !fieldChange.Definition.IsShared)
            flag = true;
    }
    if (!flag)
        return;
    this.AddVersion(item, SecurityCheck.Disable);
}
```

The method does nothing when there already is any version or there are no changes or **all field changes are for shared fields**.

So we should be good as we changed only `SKU` field and it's shared one?

### NOPE!

The `EditContext` has three overloads:

``` cs
EditContext(Item item)
EditContext(Item item, SecurityCheck securityCheck)
EditContext(Item item, bool updateStatistics, bool silent)
```

The last one has the possibility to disable update of the statistics, like the updated date, updated by etc.

In our example, we use the first overload. It by default sets `updateStatistics` to `true`. Because of this, the statistics fields are updated and when the `item.GetChanges()` method is called inside the `EnsureVersion`, it returns our shared field and four additional versioned fields. The new item version is created.

### This is not what I expected

It smells to me a bit because when you edit something you do not expect to add something without warning. I spent a few hours debugging our product importer because of this.

What about you guys? Did you know about it?