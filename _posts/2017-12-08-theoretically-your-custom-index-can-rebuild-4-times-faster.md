---
layout: post
title: "Theoretically your custom index can rebuild 4 times faster"
description: "It'a theoretical example of how much faster index rebuild can be if you exclude not nesesery fields."
date: "2017-12-08 +0100"
tags: [Sitecore, Sitecore 8.2 Update-3, Solr, Solr 6.1, Indexing, Performance]
image:
---
Recently I thought a lot about the reasons to create a custom index instead of using master/web one. I asked about it on [stack exchange](https://sitecore.stackexchange.com/questions/8253/when-and-why-to-create-a-custom-index). Everybody made a good point, but I still felt the need to convince myself.

In my current project, there are around 50k product items in the web database. Each product can have language versions so after rebuild we get around 150k documents in Solr. So I decided to create the smallest possible index that will rebuild our products, and compare rebuild times with our current custom index.

### The Thin Index Configuration

First I created a new index configuration. I called it `thinSolrIndexConfiguration`. It references `defaultSolrIndexConfiguration` but clears `documentOptions` and `virtualFields` and sets `indexAllFields` to `false`. It looks like this:

``` xml
<sitecore>
  <contentSearch>
    <indexConfigurations>

      <thinSolrIndexConfiguration type="Sitecore.ContentSearch.SolrProvider.SolrIndexConfiguration, Sitecore.ContentSearch.SolrProvider"
        ref="contentSearch/indexConfigurations/defaultSolrIndexConfiguration">

        <indexAllFields>false</indexAllFields>

        <!-- replace default document options with new empty object-->
        <documentOptions type="Sitecore.ContentSearch.SolrProvider.SolrDocumentBuilderOptions, Sitecore.ContentSearch.SolrProvider"/>

        <!-- replace default virtual fields with new empty object-->
        <virtualFields type="Sitecore.ContentSearch.VirtualFieldProcessorMap, Sitecore.ContentSearch"/>

      </thinSolrIndexConfiguration>

    </indexConfigurations>
  </contentSearch>
</sitecore>
```

Then I configured new index using the `thinSolrIndexConfiguration`. The config looks like this:

``` xml
<index id="thin_products_search_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
  <param desc="name">$(id)</param>
  <param desc="core">$(id)</param>
  <param desc="propertyStore" ref="contentSearch/indexConfigurations/databasePropertyStore" param1="$(id)" />
  <configuration ref="contentSearch/indexConfigurations/thinSolrIndexConfiguration">
    <documentOptions type="Sitecore.ContentSearch.SolrProvider.SolrDocumentBuilderOptions, Sitecore.ContentSearch.SolrProvider">
                                
      <!-- comment fields you need -->
      <exclude hint="list:AddExcludedField">
        <!-- These two fields are required for content search api-->
        <!--<Datasource>_datasource</Datasource>-->
        <!--<IndexName>_indexname</IndexName>-->
        <Hash>_hash</Hash>
        <DocumentType>_documenttype</DocumentType>
        <Database>_database</Database>
        <!--<Language>_language</Language>-->
        <Template>_template</Template>
        <Parent>_parent</Parent>
        <!--<LatestVersion>_latestversion</LatestVersion>-->
        <Version>_version</Version>
        <Group>_group</Group>
        <IsClone>_isclone</IsClone>
        <FullPath>_fullpath</FullPath>
        <!--<Name>_name</Name>-->
        <DisplayName>_displayname</DisplayName>
        <Creator>_creator</Creator>
        <Editor>_editor</Editor>
        <TemplateName>_templatename</TemplateName>
        <Created>_created</Created>
        <Updated>_updated</Updated>
        <!--<Path>_path</Path>-->
        <Content>_content</Content>
        <Tags>_tags</Tags>
      </exclude>

      <include hint="list:AddIncludedTemplate">
        <ProductTemplateId>{6CE1B316-7E96-4153-AA94-A00308CFEE17}</ProductTemplateId>
      </include>
    </documentOptions>
  </configuration>
  <strategies hint="list:AddStrategy">
    <strategy ref="contentSearch/indexConfigurations/indexUpdateStrategies/onPublishEndAsync" />
  </strategies>
  <locations hint="list:AddCrawler">
    <crawler type="Sitecore.ContentSearch.SitecoreItemCrawler, Sitecore.ContentSearch">
      <Database>web</Database>
      <Root>/sitecore/content/Website/Products/</Root>
    </crawler>
  </locations>
</index>
```

As you can see, there is only one template included (product template), and there is a list of excluded fields. Those are special Sitecore fields added by default to index, so I have to exclude them explicitly. For example, I excluded `_templatename` field because there is only one template indexed so I don't need that information to be repeated for each document in the index. In the end, you should decide for yourself which fields you really need in the index.

For this example I only left a few fields like `_language`, `_latestversion`, `_name` and `_path`. There are also `_datasource` and `_indexname` fields left. Those two are used by Content Search Api, so if you exclude them, your LINQ queries will stop working because Sitecore by default filter by those two fields. It's probably in case you have all indexes in a single core.

The `_version_` field is added by Solr and `_uniqueid` field is set to required in `schema.xml` so I left them too. After a rebuild, the Solr document looks like this:

``` json
"response":{"numFound":151337,"start":0,"docs":[
  {
    "_language":"es",
    "_datasource":"sitecore",
    "_latestversion":true,
    "_indexname":"thin_products_search_index",
    "_name":"productD35397111099555",
    "_uniqueid":"sitecore://web/{4d46474e-96d4-4d3a-99c8-ba80bbd7abf9}?lang=es&ver=1&ndx=thin_products_search_index",
    "_version_":1585204787342737408,
    "_path":["0de95ae441ab4d019eb067441b7c2450",
      "11111111111111111111111111111111",
      "4d46474e96d44d3a99c8ba80bbd7abf9",
      "82a8c3328c9641d9b77cd7869d1378c2",
      "ad447b9f560f4a989e0f79f7ef878861",
      "df6d71dc4c2e4e65bccc463210bc447b",
      "e2c221ff9d8349bdbbdd24c101042fee",
      "f2c36c9737dd4a4888ef111456f979d9"]
  },
  ...
  ```

### The Original Index Configuration

The original index configuration which inherits from the `defaultSolrIndexConfiguration` looks like this:

``` xml
<index id="original_products_search_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
  <param desc="name">$(id)</param>
  <param desc="core">$(id)</param>
  <param desc="propertyStore" ref="contentSearch/indexConfigurations/databasePropertyStore" param1="$(id)" />
  <configuration ref="contentSearch/indexConfigurations/defaultSolrIndexConfiguration" />
    <strategies hint="list:AddStrategy">
      <strategy ref="contentSearch/indexConfigurations/indexUpdateStrategies/onPublishEndAsync" />
    </strategies>
    <locations hint="list:AddCrawler">
      <crawler type="Sitecore.ContentSearch.SitecoreItemCrawler, Sitecore.ContentSearch">
        <Database>web</Database>
        <Root>/sitecore/content/Website/Products/</Root>
      </crawler>
    </locations>
</index>
```

### The Results

I rebuilt thin and original index three times on my local PC and on average it took **59 seconds** to rebuild the thin one and **228 seconds** to rebuild original one, so it's almost 4 times faster. It proves that you should put only the fields you really need into an index. The smaller index also eats less disk space, and it's easier to maintain.