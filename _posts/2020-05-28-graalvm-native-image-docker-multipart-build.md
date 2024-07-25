---
layout: post
title: 'GraalVM native image multistage Docker builds'
author: Christopher Batey
comments: true
date: '2020-05-28'
short: 'Use Docker multistage builds to build GraalVM native images into Docker images'
tags:
- docker 
- jvm 
- kubernetes
- graalvm
---

GraalVM native image can be used to create tiny Docker images as described [here](graalvm-docker-small-images.html).

Here's how to use a multistage Docker build to build the image and produce the final image.
Very useful for a real build pipeline or if you're on Mac or Windows machine.

We assume that your application is available as `app.jar` in the Docker context.
The Dockerfile has two stages:
 - Building the native image
 - Building the final application image

```dockerfile
FROM oracle/graalvm-ce:20.0.0-java11 AS build
RUN gu install native-image
COPY app.jar /build/
RUN cd build && native-image --static -jar app.jar -H:Name=output

FROM scratch
COPY --from=build /build/output /opt/output
CMD ["/opt/output"]
```

We start with the base image provided by Oracle. This contains a GraalVM JDK. In stage one we:
1. Install `native-image` with `gu`
1. Copy the application jar into the image
1. Run a static native image build

The next stage starts from `scratch` as we have a static binary with all its dependencies included so we don't
need an operating system. Then we:
1. Copy the native image built in the previous stage to /opt/output
1. Set the `CMD` to run the native image

Multi-stage builds allow us to do everything inside a Docker container but still end up with a final image that
doesn't container any of our build dependencies.

The image in this example for a Java application that only uses JDK libraries is `~7MiB`.

{% include jd-course.html %}

