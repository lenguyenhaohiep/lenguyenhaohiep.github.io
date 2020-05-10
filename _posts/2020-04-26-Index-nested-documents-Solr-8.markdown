---
layout: post
title:  "Index nested documents with Solr 8"
date:   2020-04-26 10:24:30 -0500
categories: Solr
---

#### Table of contents
1. [Schema Configuration](#schema-configuration)
2. [Index nested document](#index-nested-document)
3. [Query without using nest path](#query-without-using-nest-path)
4. [Query without using nest path](#query-using-nest-path)


This article will show how to configure solr schema and index nested documents.

Nested schema is supported from Solr 8, along with [Rudimentary Root-only Schemas](https://lucene.apache.org/solr/guide/8_0/indexing-nested-documents.html#rudimentary-root-only-schemas). So named relationships are defined between parent and nested documents instead of one keyword `_childDocuments_`. Hence, we could now have multiple relationships.

#### Schema Configuration
```
<!-- ## Nested documents -->
<field name="_root_" type="string" indexed="true" stored="true" docValues="false" />
<fieldType name="nest_path" class="solr.NestPathField" />
<field name="_nest_path_" type="nest_path" />
```

#### Index nested document
Let’s walkthrough an example of Ocean solr collection. There are a lot of species in the ocean such fish, lobster, shrimp, crab.... Of course, you could group all these species and put all of them in one single nested document list in solr but we want to keep data structure cleaner.

This query will index an ocean with 2 species: crab and fishe. Remember that `id` is required for both parent and nested documents. Do not use the same `id` in parent root document and a child document, and do not use the same child document id across relationships. This will violate integrity that solr expects (more details [here](https://lucene.apache.org/solr/guide/8_0/indexing-nested-documents.html#important-maintaining-integrity-with-updates-and-deletes)).

All examples will be POST request to the local solr.
```
POST http://localhost:8983/solr/example/update
```

```
[
	{
		"id": "ocean1",
		"doc_type": "ocean",
		"name_str": "Pacific",
		"fishes": [
			{
				"id": "fish1",
				"doc_type": "fish",
				"type_str": "Clown Fish",
				"name_str": "Nemo"
			},
			{
				"id": "fish2",
				"doc_type": "fish",
				"type_str": "Blue Tang",
				"name_str": "Dory"
			}
		],
		"crabs": [
			{
				"id": "fish1",
				"doc_type": "crab",
				"type_str": "King Crab"
			},
			{
				"id": "fish2",
				"doc_type": "crab",
				"type_str": "Blue Crab"
			}
		]
	}
]
```

#### Query without using nest path
Retrieve all oceans with their species, use `[child]` function to return all nested documents.
```
{
	"query": "*:*",
	"filter": "doc_type:ocean",
	"fields": "*,[child]"
}
```
Retrieve fishes only, use `childFilter`
```
{
	"query": "doc_type:ocean",
	"fields": "*,[child childFilter=doc_type:fish]"
}
```

Retrieve names of oceans and types of fishes.
```
{
	"query": "doc_type:ocean",
	"fields": "name_str,type_str,fishes,[child childFilter=doc_type:fish fl=type_str]"
}

```
The rule here is that we have to provide all fields including parent and child fields in `fields` (or `fl`). In the example, `name_str` is both parent and child field, and `type_str` is exclusively a child field.
Because fishes also have names, so the second `fl` has to be passed in `[child]` function to filter only `type_str` so the query will not return `name_str` of nested document. There is a limit here is that we could not return only child field if parent uses this field as well. It's to say that we could not return `name_str` of fishes but hide the one of `ocean` parent.

There is a trick to do that by putting more metadata in field names and query with wildcards.
So, let index the second ocean
```
[
	{
		"id": "ocean2",
		"doc_type": "ocean",
		"name_str": "Atlantic",
		"fishes": [
			{
				"id": "fish3",
				"doc_type": "fish",
				"type_fish_str": "Garfish",
				"name_fish_str": "Bob"
			}
		]
	}
]
```
Retrieve names of fishes from the ocean withh `id=ocean2.
```
{
	"query": "doc_type:ocean AND id:ocean2",
	"fields": "*_fish_*,fishes,[child childFilter=doc_type:fish fl=name_fish_str]"
}
```
Of course, wildcard `*_fish_*` will not work properly if there are also parent fields matching this wildcard.

#### Query using nest path
`_nest_path_` is defined as `stored=false` in sole schema, however, for the demo purpose of using nest path to query, it is stored.

It is obviously that we use `doc_type` to differentiate parent and children documents. There is also another way to do that without using `doc_type`, however, I personally prefer using `doc_type`.
Let's see how it looks like

Retrieve all oceans
```
{
    "query": "-_nest_path_:*",
    "fields": "_nest_path_,*,[child fl=_nest_path_]"
}
```
Solr will return
```
{
  "response":{
		"numFound":2,
		"start":0,
		"docs":[
      {
        "id":"ocean2",
        "doc_type":"ocean",
        "name_str":"Atlantic",
        "_root_":"ocean2",
        "_version_":1666219523517186048,
        "fishes":[
          { "_nest_path_":"/fishes#0" }
				]
			},
      {
        "id":"ocean1",
        "doc_type":"ocean",
        "name_str":"Pacific",
        "_root_":"ocean1",
        "_version_":1666220098844622848,
        "crabs":[
          { "_nest_path_":"/crabs#0" },
          { "_nest_path_":"/crabs#1" }
				],
        "fishes":[
          { "_nest_path_":"/fishes#0" },
          { "_nest_path_":"/fishes#1" }
				]
			}
		]
  }
}
```
So `_nest_path_` will indicate the relationships, and we could see that there is no `_nest_path_` in the parent. The criteria `-_nest_path_:*` is equal to `doc_type:ocean` in this case. `-_nest_path_:*` means there is no `_nest_path_` field.

Actually, `_nest_path_` is useful for solr to handle relationships, however using this field is optional if we introduce `doc_type`.

All codes and examples could be found [in my github](https://github.com/lenguyenhaohiep/solr8-nested-documents).
