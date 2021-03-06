---
layout: post
title: Exclude classes from Jacoco report with gradle
date: 2020-05-08 00:00:00
categories: Gradle Jacoco
---
When running tests with JUnit, JUnit will create a test report in binary format and Jacoco will generate a readable report (html, xml). If you want to exclude modules, classes, methods from jacoco report. You could define 2 tasks `test` and `jacocoTestReport` in `build.gradle` (version 6.3 is used) as follow.

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
Whereas:

**(1)** will exclude test coverage result of excluded components such as modules, classes, inner classes, methods given **their java class names** from the binary format.

**(2)** will exclude test coverage result of excluded components such as modules, classes, inner classes (**method is not supported**) given **their file paths** from the jacoco readable report.

If you use **(2)** only, your test coverage will not include any result of excluded components in the readable report. However, if you use **(1)** only, your Jacoco readable report will also include coverage result of excluded components, with all 0%. So you could define both **(1)** and **(2)** or only **(2)** is enough.
