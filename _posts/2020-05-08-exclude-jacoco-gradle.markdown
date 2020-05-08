---
layout: post
title: Exclude classes from Jacoco report with gradle
date: 2020-05-08 00:00:00
categories: Tech
tags: english, java, gradle
---

If you want to exclude modules, classes, methods from jacoco report. You could define 2 tasks `test` and `jacocoTestReport` in `build.gradle` (version 6.3 is used) as follow.

```
test { 
	finalizedBy jacocoTestReport

	jacoco {
		excludes = [
			'path.to.your.package.module.*',
			'path.to.your.package.module.Class',
			'path.to.your.package.module.Class.InnerClass',
			'path.to.your.package.module.Class.method',
		] // (1)
	}
} 

jacocoTestReport {
	dependsOn test 

	afterEvaluate { 
		getClassDirectories().setFrom(classDirectories.files.collect {
			fileTree(dir: it,
				exclude: [
					'**/path/to/your/package/module/*.class'
					'**/path/to/your/package/module/Class.class'
					'**/path/to/your/package/module/Class$InnerClass.class'
				] // (2)
			)
		})
	}
}
```

When running tests with JUnit, it will create a test report in binary format and Jacoco will generate a readable report (html, xml).

(1) will exclude test coverage result of components (modules, classes, inner classes, methods) from the binary format.

(2) will exclude test coverage result of components (modules, classes, inner classes (method is not supported)) from the jacoco report.

If you use (2) only, you test coverage will not include any result of excluded components in the readable format. However, if you use (1) only, your readable report with Jacoco will include coverage result of excluded components, with all 0% for all. So you could define both (1) and (2) or only (2) is enough.
