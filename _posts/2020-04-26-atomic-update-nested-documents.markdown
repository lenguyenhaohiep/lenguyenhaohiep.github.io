---
layout: post
title:  "Use atomic update on nested document with Solr 8"
date:   2020-04-26 10:24:30 -0500
categories: Solr
---

#### Table of contents
1. [Add new nested documents](#add-new-nested-documents)
2. [Update nested documents](#update-nested-documents)
3. [Remove nested documents](#remove-nested-documents)
4. [Partial update together with parent](#partial-update-together-with-parent)


As [atomic update is now supported from Solr 8](https://lucene.apache.org/solr/guide/8_1/updating-parts-of-documents.html), this article will demonstrate how to perform atomic partial update on nested documents.

Let's say that we want to add more fishes into Pacific Ocean (read more [here](https://lenguyenhaohiep.github.io/solr/Index-nested-documents-Solr-8/) to know how to index nested documents).

#### Add new nested documents
In order to add, we use partial update with `add` modifier. The `id` parent is always required to perform any action on nested documents.
Let's add 2 more fishes into the Pacific Ocean with `id=ocean1`.
```
POST http://localhost:8983/solr/example/update
```
```
[{
	"id": "ocean1",
	"fishes": {"add": [
		{
			"id": "fish4",
			"doc_type": "fish",
			"name_str": "Doe"
		},
		{
			"id": "fish5",
			"doc_type": "fish",
			"name_str": "Hans"
		}
	]}
}]
```
It's very important to remember that Solr cannot guarantee unique id key on nested documents. By that means if we execute this query twice, there will be duplicated nested documents. So, we have to perform the partial update very carefully. We could avoid this by verifying whether the nested documents exits or just perform a query to remove them first ([more details here](#remove-nested-documents)).

#### Update nested documents
Parent root document id is always mandatory for any action on nested documents. Here, this important information is put directly in the URL with `_route_` parameter instead of inside the request body.
Let's modify some fish names from Pacific Ocean with `id=ocean1`. So, we have to add `_route_=ocean1`, it's compulsory because if not, solr will understand that we are performing actions on parent root document.
```
POST http://localhost:8983/solr/example/update?_route_=ocean1
```
```
[
	{
		"id": "fish4",
		"name_str": {"set": "Bobby"}
	},
	{
		"id": "fish5",
		"name_str": {"set": "Benny"}
	}
]
```
So now, we could apply [any modifier](https://lucene.apache.org/solr/guide/8_1/updating-parts-of-documents.html#atomic-updates) depending on the type of the field. In this example, the `set` modifier is used to update the names of fishes.

#### Remove nested documents
We use partial update with `remove` modifier to remove nested documents. As in previous section, the parent `id` is mandatory to perform any update on nested documents along with nested document ids to remove.

Let's remove 2 fishes with `ids=fish4, fish5` from Pacific Ocean with `id=ocean1`
```
POST http://localhost:8983/solr/example/update
```
```
[{
	"id": "ocean1",
	"fishes": {"remove": [
		{
			"id": "fish4"
		},
		{
			"id": "fish5"
		}
	]}
}]
```

Again, any action on nested document has to be performed within the parent scope. It could be understood that action on nested documents is the update action on the parent document. It means, if you provide nested document ids (`fish3`, `fish4`) and perform a normal delete, nothing will happen ((more details here)[https://lucene.apache.org/solr/guide/8_0/major-changes-in-solr-8.html#nested-documents]).


#### Partial update together with parent
We could combine partial update on parent together with partial updates (remove and add). However, we cannot combine multiple actions on the same nested block. It means cannot have `add` or `remove` perform in the same time.
We cannot combine partial updating on nested documents simultaneously with parent.

Let's add one more fish and update the surface information of Pacific Ocean.
```
POST http://localhost:8983/solr/example/update
```
```
[{
	"id": "ocean1",
	"surface_str": {"set": "65,250,000 km2"},
	"fishes": {"add": [
		{
			"id": "fish6",
			"doc_type": "fish",
			"name_str": "Doe"
		}
	]}
}]
```

All codes and examples could be found [in my github](https://github.com/lenguyenhaohiep/solr8-nested-documents).
