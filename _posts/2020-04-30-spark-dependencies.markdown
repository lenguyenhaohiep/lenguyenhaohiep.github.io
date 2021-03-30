---
layout: post
title:  "Modify Spark Job’s dependencies folder"
date:   2016-08-07 00:24:30 -0500
categories: Spark
---

By default, Spark uses the folder `/root/.ivy2/` to store extra dependencies. When spark is running, you can see

```
Ivy Default Cache set to: /root/.ivy2/cache
The jars for the packages stored in: /root/.ivy2/jars
```

In order to change it, you can add `--conf spark.jars.ivy=</path/to/folder>` as a submit option.
