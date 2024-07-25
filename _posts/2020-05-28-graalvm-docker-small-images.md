---
layout: post
title: 'Tiny docker images for Java with GraalVM native image'
author: Christopher Batey
comments: true
date: '2020-05-28'
tags:
- docker 
- jvm 
- kubernetes
- graalvm
---

Building small Docker images when using Java is hard. 
Even with Alpine and a cut down JVM you're still looking
at a 70MiB image. Good compared to a 500MiB JDK image or a 200MiB debian slim image but still large compared to what
native languages can produce.

GraalVM native image promises to improve this situation. 

With GraalVM native image sizes can be as little as 7Mib for a
Java application.

Here are the steps to get a 7MiB Java docker image.

### Install GraalVM

At easy way to install GraalVM and switch between multiple versions is via the [Java version manager (Jabba)](https://github.com/shyiko/jabba). To install jabba:

`curl -sL https://github.com/shyiko/jabba/raw/master/install.sh | bash && . ~/.jabba/jabba.sh`

Then use jabba to install GraalVM, at the time of writing 20.0.0 is the latest:

`jabba install graalvm@20.0.0`

Then finally select it as your JVM:

`jabba use graalvm@20.0.0`

You can validate that GraalVM is being used via:

```
java -version
openjdk version "11.0.6" 2020-01-14
OpenJDK Runtime Environment GraalVM CE 20.0.0 (build 11.0.6+9-jvmci-20.0-b02)
OpenJDK 64-Bit Server VM GraalVM CE 20.0.0 (build 11.0.6+9-jvmci-20.0-b02, mixed mode, sharing)

```

### Install native image

Current versions of GraalVM don't include native image by default, you need to install it with gu:

`gu install native-image`

Check you have `native-image` on your path:

```
native-image
```

### Compile a native image

If you're on Linux you can do this on your laptop, otherwise you will have to do it in a VM or in a Docker Container. 
Native image builds a binary for your current platform. For Docker images that should be Linux. 

Compile the following Java application:


```java
public class Main {
   public static void main(String[] args) {
       System.out.println("Hello from GraalVM");
   }
}
```

Compile:

```
javac Main.java
```

For real applications you'd likely be working with a jar created by a build tool so create a jar for Mian.class

```
jar --create --file=app.jar --main-class=Main  Main.class
```

Finally build a native image for the jar file:

```
native-image -jar ./app.jar -H:Name=output
Build on Server(pid: 28002, port: 54771)*
[output:28002]    classlist:   1,085.23 ms,  1.00 GB
[output:28002]        (cap):   2,945.86 ms,  1.29 GB
[output:28002]        setup:   4,094.41 ms,  1.29 GB
[output:28002]   (typeflow):   3,933.15 ms,  1.29 GB
[output:28002]    (objects):   3,689.07 ms,  1.29 GB
[output:28002]   (features):     157.09 ms,  1.29 GB
[output:28002]     analysis:   7,942.96 ms,  1.29 GB
[output:28002]     (clinit):     152.25 ms,  1.67 GB
[output:28002]     universe:     385.89 ms,  1.67 GB
[output:28002]      (parse):     630.83 ms,  1.67 GB
[output:28002]     (inline):   1,475.64 ms,  1.67 GB
[output:28002]    (compile):   4,549.69 ms,  1.67 GB
[output:28002]      compile:   7,062.23 ms,  1.67 GB
[output:28002]        image:   1,050.34 ms,  1.67 GB
[output:28002]        write:     323.95 ms,  1.67 GB
[output:28002]      [total]:  22,154.28 ms,  1.67 GB
```

This will create a native image for your application. We can run it to make sure it works:

```
./output
Hello from GraalVM
```

### Putting it in a Docker container

Let's start with the following `Dockerfile`:

```dockerfile
FROM debian:buster-slim
COPY output /opt/output
CMD ["/opt/output"]
```

Building it gives us a docker image of:

```
docker build . -t debian-graal
docker images | grep debian-graal
debial-graal-intro latest 75.9MB
```

If you are not running on Linux and you try the above you'll get an exec format error of some kind as the native image
will be your host operating system.

### Getting it down to 7MiB

The image is still so large as it contains most of Debian. 
If we compile a static native image then we won't need anything in our image to run it which will really get the size down.

```
native-image --static -jar ./app.jar -H:Name=output
```

Then change the final image to:

```dockerfile
FROM scratch

COPY output /opt/output

CMD ["/opt/output"]
```

If we build this and check its size, it is now ~7MiB.

And there you have it. Java in a container with a 7MiB image.

{% include jd-course.html %}


