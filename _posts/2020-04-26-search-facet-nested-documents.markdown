---
layout: post
title:  "Search and Faceting on nested documents with Solr 8"
date:   2020-04-26 10:24:30 -0500
categories: Solr
---
#### Table of contents
1. [Search parents with children filtering](#search-parents-with-children-filtering)
2. [Search parents with both parent and children filtering](#search-parents-with-both-parent-and-children-filtering)
3. [Faceting on nested documents](#faceting-on-nested-documents)
4. [Faceting on nested documents and parent together](#faceting-on-nested-documents-and-parent-together)
5. [Faceting and Excluding](#faceting-and-excluding)


This article will show you how to build solr queries which perform search and faceting on nested documents (children).

Let’s walkthrough an example of [Ocean solr collection](https://lenguyenhaohiep.github.io/solr/Index-nested-documents-Solr-8/)). There are some fishes and crabs in the Pacific Ocean.

#### Search parents with children filtering
Find all oceans where Clown Fishes live.
```
POST http://localhost:8983/solr/example/select
```
```
{
	"query": "{!parent which=doc_type:ocean}(type_str:\"Clown Fish\")",
	"fields": "*,[child]"
}
```
Let's analyze this query. We use a [parent query parser](https://lucene.apache.org/solr/guide/8_0/other-parsers.html#block-join-parent-query-parser). The `!parent` function will return the parents which match the children documents’ condition (Clown Fish).

The same, find all oceans which have a Clown Fish named Nemo.
```
{
	"query": "{!parent which=doc_type:ocean}(type_str:\"Clown Fish\" AND name_str:Nemo)",
	"fields": "*,[child]"
}
```

#### Search parents with both parent and children filtering
Find Pacific Ocean where Nemo lives.
```
POST http://localhost:8983/solr/example/select
```
```
{
	"query": "name_str:Pacific AND {!parent which=doc_type:ocean}name_str:Nemo",
	"fields": "*,[child]"
}
```
We could combine normal parent query with [block join query parser]( https://lucene.apache.org/solr/guide/8_0/other-parsers.html#block-join-query-parsers) provided by `!parent` function. It's recommended that we should use brackets to avoid misunderstanding. So if we inverse the condition, `{!parent which=doc_type:ocean}name_str:Nemo AND name_str:Pacific`, without brackets, Solr will understand that `name_str:Pacific` is also a condition on nested documents.

#### Faceting on nested documents
Facet all types of crabs and fishes in Pacific Ocean.
```
POST http://localhost:8983/solr/example/select
```
```
{
  "query": "*:*",
  "filter": [
    "name_str:Pacific"
  ],
  "limit": 0,
  "facet": {
    "fish_types": {
      "type": "terms",
      "field": "type_str",
      "domain": {
      	"blockChildren": "doc_type:ocean",
      	"filter": "doc_type:fish"
      }
    },
    "crab_types": {
      "type": "terms",
      "field": "type_str",
      "domain": {
      	"blockChildren": "doc_type:ocean",
      	"filter": "doc_type:crab"
      }
    }
  }
}
```
Solr will return
```
{
  "response": {
    "numFound": 1,
    "start": 0,
    "docs": []
  },
  "facets": {
    "count": 1,
    "fish_types": {
      "buckets": [
        {
          "val": "Blue Tang",
          "count": 1
        },
        {
          "val": "Clown Fish",
          "count": 1
        }
      ]
    },
    "crab_types": {
      "buckets": [
        {
          "val": "Blue Crab",
          "count": 1
        },
        {
          "val": "King Crab",
          "count": 1
        }
      ]
    }
  }
}
```
Facet computation operates on a `domain` which is the result of main query. In this example, the main query will define an original domain for parent which does not include nested documents. As we could see that, there is a `domain` option added to the query. This is called (block join domain change)[https://lucene.apache.org/solr/guide/8_0/json-faceting-domain-changes.html#block-join-domain-changes]. This option will include nested documents into the original domain.
```
"domain": {
  "blockChildren": "doc_type:ocean",
  "filter": "doc_type:fish"
}
```

Whereas `blockChildren` indicates the condition to match all parents consisting of the nested documents. It means matching all parents with `doc_type=ocean`. However, there are 2 types (crabs and fishes), so `filter` will narrow down the result set to have only fishes before faceting.

#### Faceting on nested documents and parent together
Facet ocean names along with facets on fish and crab types.
```
{
  "query": "*:*",
  "limit": 0,
  "facet": {
    "ocean_names": {
      "type": "terms",
      "field": "name_str",
      "domain": {
      	"blockParent": "doc_type:ocean"
      }
    },
    "fish_types": {
      "type": "terms",
      "field": "type_str",
      "domain": {
      	"blockChildren": "doc_type:ocean",
      	"filter": "doc_type:fish"
      }
    },
    "crab_types": {
      "type": "terms",
      "field": "type_str",
      "domain": {
      	"blockChildren": "doc_type:ocean",
      	"filter": "doc_type:crab"
      }
    }
  }
}
```
Again the `domain` option is used for `ocean_names` facet, it's very important to indicate this option to avoid name conflict. Especially when the parent and nested documents use same fields which is `name_str` in this case.
```
"domain": {
  "blockParent": "doc_type:ocean"
}
```
Whereas `blockParent` indicates the condition that matches parent documents. Remember that `blockParent` or `blockChildren` are conditions on parent documents. They work as similarly as block join query parsers `{!parent which=<allParents>}` and `{!child of=<allParents>}` respectively.
If `domain` is not added to `ocean_names` facet, the result will contain all facet counts of fishes and crabs.

#### Faceting and Excluding
Facet all types of fishes in Pacific Ocean with excluding filters which are Clown Fish and Pacific Ocean.
```
{
  "query": "*:*",
  "filter": [
  	"{!tag=ocean_name}name_str:Pacific",
  	"{!tag=fish_type parent which=doc_type:ocean}type_str:\"Clown Fish\""
   ],
  "limit": 10,
  "facet": {
    "ocean_names": {
      "type": "terms",
      "field": "name_str",
      "domain": {
      	"blockParent": "doc_type:ocean",
    	  "excludeTags": "ocean_name"
      }
    },
    "fish_types": {
      "type": "terms",
      "field": "type_str",
      "domain": {
      	"blockChildren": "doc_type:ocean",
      	"filter": "doc_type:fish",
      	"excludeTags": "fish_type"
      }
    }
  }
}
```
The `excludeTags` is used to exclude filters from facets. Unlike normal documents without nested documents where only filters related to the facet field have to be excluded ([more details](https://lucene.apache.org/solr/guide/8_2/faceting.html#tagging-and-excluding-filters)), here we have to exclude also `fish_type` tag from `ocean_names` facet. It's because the parent condition will give the same partial result as done by the children condition. In this example, Pacific Ocean (`!tag=ocean_name`) contains Clown Fish, so Clown Fish is filtered out if it's not excluded in `fish_types` facet. The same, `!parent` function will find Pacific Ocean (`!tag=fish_type`) which is also filtered out if it's not excluded from `ocean_names` facet.
