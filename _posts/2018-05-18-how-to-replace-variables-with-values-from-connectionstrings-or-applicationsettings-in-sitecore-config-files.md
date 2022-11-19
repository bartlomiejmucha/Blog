---
layout: post
title: "How to replace variables with values from ConnectionStrings or ApplicationSettings in Sitecore config files"
description: "I created an extension to Sitecore Config Reader class that allows to replace variables with values form Connection Strings or Application Settings."
date: "2018-05-18 +0100"
tags: [Sitecore, Sitecore 9.0 Update-1, Square, Configuration Files]
image:
---
In Sitecore's config files, you can use variables that are replaced at config loading time. The `DoGetConfiguration` method of `ConfigReader` merges all your include files and then passes the result to these two methods:

``` cs
protected internal virtual XmlDocument DoGetConfiguration()
{
    ....
    this.ReplaceGlobalVariables((XmlNode) xmlDocument.DocumentElement);
    this.ReplaceEnvironmentVariables((XmlNode) xmlDocument.DocumentElement);
    ....
}
```

Global variables are replaced first and environment variables next.

### Global variables

To define global variable you create `<sc.variable name="variable_name" value="variable_value" />` element in config file. Take a look at the example of `dataFolder` variable from inside `Sitecore.config` file:

``` xml
<configuration>
  <sitecore>
    <sc.variable name="dataFolder" value="/App_Data" />
...
    <scheduling>
      <agent type="Sitecore.Tasks.CleanupAgent" >
        <files>
          <remove folder="$(dataFolder)/logs" />
        </files>
      </agent>
    </scheduling>
  </sitecore>
</configuration>
```

The `dataFolder` variable is defined and then it is used to define path for `CleanupAgent`. It is defined at the top however, you can put variables anywhere inside `<sitecore>` node.

### Environment variables

Sitecore can also replace variables with values returned by `System.Environment.GetEnvironmentVariable` method. To use this kind of variable you have to prefix the variable name with `env` like this `$(env:variable_name)`. For example: `$(env:COMPUTERNAME)` would be replaced with the local computer name.

### Connection Strings and Application Settings

If you are like me, you probably don't want to hardcode any connection string or application setting directly in config files, so you can Build Once, Deploy Anywhere!.

A good example of setting, that is often hardcoded in config file, is `ContentSearch.Solr.ServiceBaseAddress` setting. If you have different url per environment, the one option is to commit all possible urls and pick the correct one using config layers and roles.

To help in such situation I extended `ConfigReader` with possibility to replace variables with Connection Strings and Application Settings. On Azure, Connection Strings and Application Settings can be defined per enrionment in Portal. You can also use VSTS or Octopus to replace those settings.

### How to use it:

Similar like for environment variables, you have to use prefix. For connection strings it's cs and for application settings it is as. For example: `$(cs:core)` or `$(as:DictionaryFolder)`

The code is committed to my [Square](https://github.com/bartlomiejmucha/Square) repository on GitHub into **Square.Foundation.Configuration** module. Here is a [link](https://github.com/bartlomiejmucha/Square/tree/master/src/Foundation/Configuration/code).

Take care!