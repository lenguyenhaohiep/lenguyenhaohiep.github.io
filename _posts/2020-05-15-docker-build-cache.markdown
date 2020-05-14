---
layout: post
title: Some reasons why docker build does not use cache
date: 2020-05-13 00:00:00
categories: Docker
---
Sometimes we do not understand why docker does not use cache when it builds images. There are some reasons that I know from my experience and to be honest it took me a lot of time to debug from the very first time I used docker.


####  Copy everything into docker container workspace
```
project
│   README.md
│   Dockerfile    
│		.gitignore
|   build.sh
└───src
│   │   file011.txt
│   │   file012.txt
```

Dockerfile
```
FROM image:latest
COPY . /workspace // --> BAD
```
Every modification will lead to rebuild the whole image because docker detects changes in the root working directory even on the files that are not related to the projects.
```
FROM image:latest
COPY /src /workspace/src // --> GOOD
COPY /build.sh /workspace/build.sh // --> GOOD

```

####  Put Dockerfile in the same project directory
```
project
│   README.md    
│   .gitignore
|   build.sh
└───src
│   │   file011.txt
│   │   file012.txt
|   |   Dockerfile // --> BAD
```
Dockerfile
```
FROM image:latest
COPY /src /workspace/src
```
If we put Dockerfile in the same workspace, and then we copy the whole workspace into Dockerfile. So, when we want to modify a line or even add a comment in the Dockerfile, this action will trigger docker not to use cache anymore because it detects changes.

####  Do not respect stage dependency
```
FROM image as stage1
RUN build.sh

FROM image as stage2
COPY --from=stage1 /workspace/file.txt /home

FROM image as stage3
COPY --from=stage2 /home/file.txt /target

FROM image
COPY --from=stage3 /home/file.txt /target
```
If the objective is just to build only one final image, there is no problem because docker will handle by itself. Docker will use caches from each stage to optimize image building. Remember that for each stage, we should avoid cases that trigger docker to rebuild everything (as described in [**1**](#copy-everything-into-docker-container-workspace) and [**2**](#put-dockerfile-in-the-same-project-directory)).

If we want to build both 3 images to reuse.
```
docker build -t stage1_img --target=stage1 .
docker build -t stage2_img --target=stage2 --cache-from=stage1_img .
docker build -t stage3_img --target=stage3 --cache-from=stage1_img,stage2_img .
docker build -t final_img --cache-from=stage1_img,stage2_img,stage3_img .
```
Whereas `--cache-from` will indicate caching sources which are images (`name:version`, in the example `latest` is implicit). It's important that we should not break stage dependency, because the next stage depends on the previous stage. For example, you could build the last image use only `stage1_img`, `stage1_img,stage2_img` or all of them `stage1_img,stage2_img,stage3_img`, however you should not do `stage2_img`, `stage2_img,stage3_img` without `stage1_img` because `stage2_img` depends on `stage1_img`.
If you run all builds in the same machine, docker could handle using local cache but if you build in different machines, docker will not find caching sources and that leads to rebuild everything.
