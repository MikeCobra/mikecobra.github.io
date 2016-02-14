---
title: "Dockerizing Minecraft, for Better or for Worse"
excerpt_separator: <!-- more -->
layout: post
---

I had been considering running a Minecraft server for a while, but there were
various reasons why I hadn't for the last couple of years. However, the stars
had aligned and it was time for me and my gaming friends to venture into the
land of blocks once again.

There was something else to consider as well. Docker. It interested me. It was
starting to become more relevant at work. I knew very little about it.

So, I thought, I could learn about Docker by running Minecraft using it.

<!-- more -->

I needed to increase the complexity though as I knew enough to be aware that
running a single process in docker wouldn't be enough to satisfy me. No, I
needed a few processes that depended on each other to see how a multi process
tech stack could look in the land of Docker.

There are plenty of tools in existence for generating web-based maps for
Minecraft. These include [Overviewer](https://overviewer.org/), which
generates a set of static files which can be presented over HTTP to produce a
web-based map. It can run while the Minecraft server is running and doesn't
need direct access to the server process, unlike some other mapping tools.

Three processes were needed:

1. A Minecraft server, for playing Minecraft
2. An Overviewer process, generating maps
3. A HTTP server, presenting those maps

So the project began.

## The Minecraft Server

The first hurdle was to install Docker. I was running Ubuntu 15.10 so I could
follow the [official Docker documentation for installing on Ubuntu](https://docs.docker.com/engine/installation/linux/ubuntulinux/), which I
found rather good and easy to follow.

Initially, due to lack of wisdom, I did fail to read through the "Optional
configurations" section and found that DNS resolution did not work inside the
containers. I solved this by turning off dnsmasq in NetworkManager before
spotting that section and then changing to use a default opt setting the dns
inside the containers to use Google's DNS server.

Next, I had to worry about what base image to base my containers off. My current
personal distro preference has been Ubuntu but one of my colleagues informed me
that [Docker may be planning on moving to Alpine Linux as their
default](http://siliconangle.com/blog/2016/02/09/docker-gets-minimalist-with-plan-to-migrate-images-to-alpine-linux/).
I formerly ran Arch Linux and so the minimalist aspect appealed to me,
especially when the contents of the container were unlikely to be something a
human would have direct contact with.

From here, it was reasonably easy to complete the first two statements of the
Dockerfile:
{% highlight docker %}
FROM alpine:3.3

MAINTAINER Mike Clarke <michaelclarkecs@gmail.com>
{% endhighlight %}

This was all well and good, but still a little way from actually being a
functional Minecraft server.

Minecraft runs on Java so some form of runtime was going to be needed. To keep
it minimalist, I went for a JRE[^1]. This exists in the community repository for
Alpine Linux which is activated by default in the official docker image. To
install the JRE, the Dockerfile needed:

{% highlight docker %}
RUN apk --update add openjdk8-jre && rm -rf /var/cache/apk/*
{% endhighlight %}

The `apk` command is equivalent to running `apt-get update && apt-get install`,
updating the package lists and installing the desired package. The `rm` cleans
up the package listing after the package is installed to keep the container file
system free of data unnecessary for running. Doing both these commands in one
`RUN` command keeps the number of Docker file system layers down to a minimum as
only states after full statements have run are saved. As these two commands are
performing one task and cleaning up this task, effectively making one logical
change, it makes sense to keep them as a single layer.

Minecraft worlds are stored on disk and need persisting. I was aware that the
container should be treated as an ephemeral unit. When it stops, it is
destroyed. When it was needed again, it would be recreated. For persisting my
world data, I read up on [Docker
volumes](https://docs.docker.com/engine/userguide/containers/dockervolumes/).
This would allow the data to live on past the destruction of the container.

To create the volume and use it for my Minecraft data was simply a case of
adding:

{% highlight docker %}
VOLUME /minecraft
WORKDIR /minecraft
{% endhighlight %}

`WORKDIR` means that commands will be run as if they are in the specified
directory, which is useful in this case as Minecraft stores all of it's data in
(and loads all of it's configuration from) the current directory.

A Minecraft server would be a very lonely place without players, so I exposed
the default Minecraft port.

{% highlight docker %}
EXPOSE 25565
{% endhighlight %}

I also added the command to be run when the container runs, the actual Minecraft
server process.

{% highlight docker %}
ENTRYPOINT ["java", "-jar", "/minecraft_server.jar", "nogui"]
{% endhighlight %}

I needed to build my docker image by running:

{% highlight bash %}
docker build -t mikecobra/minecraft .
{% endhighlight %}

At this point, we are ready to load up our Minecraft server. "Wait, you haven't
add the Minecraft JAR yet!", I hear you say. Unfortunately, Minecraft's EULA
doesn't allow me to distribute the Minecraft executable, even within a Docker
container. So a Minecraft Server JAR must be downloaded separately and mounted
at `/minecraft_server.jar`. Yes, what I have done is made a Docker container to
run arbitrary jars that accept `nogui` as an argument.

So, I downloaded a Minecraft server JAR and created a directory for my
persistent data to live and accepted the Minecraft EULA:

{% highlight bash %}
wget http://wherethereisaminecraftjar.test/minecraft_server.jar
mkdir minecraft-data/
echo "eula=true" > minecraft-data/eula.txt
{% endhighlight %}

Then I span up my server using:

{% highlight bash %}
docker run -v /path/to/jar:/minecraft_server.jar -v /path/to/minecraft-data:/minecraft -p 25565:25565 mikecobra/minecraft
{% endhighlight %}

I could connect to my Minecraft server on localhost, success!

At this point, I showed my work to my aforementioned colleague, who also happens
to spend some of his time playing Minecraft. He pointed out that my processes
probably shouldn't be running as root inside my container for best practice.

So, I added to the Dockerfile to add a user and group for the process to run as
and instructed the processes to be run as that user, resulting in a final
Dockerfile of:

{% highlight docker linenos %}
FROM alpine:3.3

MAINTAINER Mike Clarke <michaelclarkecs@gmail.com>

RUN apk --update add openjdk8-jre && rm -rf /var/cache/apk/*
RUN addgroup minecraft && adduser -s /bin/ash -G minecraft -D minecraft

VOLUME /minecraft
WORKDIR /minecraft
EXPOSE 25565
USER minecraft

ENTRYPOINT ["java", "-jar", "/minecraft_server.jar", "nogui"]
{% endhighlight %}

## Overviewer

When I first tackled Overviewer, I did try to use an Alpine Linux base image. I
got stumped by my lack of python knowledge when trying to compile to project for
Alpine.

In the end, I gave up and went back to the comfort of deb land. I went for a
Debian base image for no particular reason, if I did this again, I'd probably
use a Ubuntu one.

A lot of what was done for this image was similar to the Minecraft server,
installing packages, creating users and volumes. I won't go into detail about

I needed to allow my users to configure Overviewer, I didn't want their
configuration options to be limited because they were using my container. At the
same time, I wanted the container to run without any configuration, as I hadn't
managed that for the server container.

To achieve this, I included a very basic configuration file in the image. This
meant it could run without any user configuration but a more custom config file
could be mounted over it, in a similar manner to how the Minecraft JAR was
mounted for the server container.

Using two volumes allowed me to separate the two different sets of persistent
data as they are not needed by the same other processes. This container would be
the only one of the three needing access to both the world and the map data.

The final Dockerfile and the Overviewer configuration I used:

{% highlight docker linenos %}
FROM debian:8
MAINTAINER Mike Clarke <michaelclarkecs@gmail.com>

RUN echo "deb http://overviewer.org/debian ./" > /etc/apt/sources.list.d/overviewer.list \
    && apt-get update \
    && apt-get install -y wget \
    && wget -O - http://overviewer.org/debian/overviewer.gpg.asc | apt-key add - \
    && apt-get update \
    && apt-get install -y minecraft-overviewer
RUN useradd -ms /bin/bash overviewer
RUN wget -O /texturepack https://dl.dropboxusercontent.com/u/75066972/Tiny\ Pixels\ -\ 25th\ October.zip
COPY overviewer.config /

VOLUME ["/world"]
VOLUME ["/map"]
USER overviewer

ENTRYPOINT ["overviewer.py", "--config=/overviewer.config"]
{% endhighlight %}

{% highlight python linenos %}
worlds["World"] = "/world"

renders["normalrender"] = {
    "world": "World",
    "title": "World",
}

texturepath = "/texturepack"
outputdir = "/map"
{% endhighlight %}

## HTTP Server

For the HTTP server, I wanted to keep it lightweight. After all, It was only
going to be serving static files. The only web servers I had really used much
previously were Apache and Nginx, which I was aware were quite heavy.

[Lighttpd](https://www.lighttpd.net/) seemed to fit my purposes quite well.
Alpine also had a package for it, which meant I could avoid any of the
compilation problems I had with Overviewer.

The actual implementation was very similar to the other two Dockerfiles. Add
packages, add users, copy a simple config that could be mounted over to
customize. I needed to use the `-D` option for starting lighttpd in the
foreground so that STDOUT would be useful from the Docker host and so the
container would die properly if the web server failed.

I encountered one issue when first testing it due to not having added required
MIME types to the lighttpd configuration. The final configuration used
*should* be enough to serve a lot of completely static content websites.

The final lighttpd Dockerfile and configuration:

{% highlight docker linenos %}
FROM alpine:3.3
MAINTAINER Mike Clarke <michaelclarkecs@gmail.com>

RUN addgroup www && adduser -G www -s /bin/ash -D www
RUN apk add --update lighttpd && rm -rf /var/cache/apk/*
COPY lighttpd.conf /etc/lighttpd/lighttpd.conf

VOLUME ["/data"]

EXPOSE 80

ENTRYPOINT ["lighttpd", "-D", "-f", "/etc/lighttpd/lighttpd.conf"]
{% endhighlight %}

{% highlight lighty linenos %}
server.document-root = "/data"

server.port = 80

server.username = "www"
server.groupname = "www"

mimetype.assign = (
  ".html" => "text/html",
  ".js" => "text/javascript",
  ".css" => "text/css",
  ".txt" => "text/plain",
  ".jpg" => "image/jpeg",
  ".png" => "image/png"
)

index-file.names = ( "index.html" )
{% endhighlight %}

## Coordinating with Compose

At this point, I had three containers which worked together to run a
functioning Minecraft server and generate and serve a Map of it. I had three
`docker run` commands which I could use to start up my Minecraft environment.

However,my commands effectively contained configuration, about where the
Minecraft and map data should live and which ports should be mapped. From my
experience, this data would be easier to manage if it lived in a configuration
file, where it can be easily stored in source control or managed using a
configuration management tool.

This is where [Docker Compose](https://docs.docker.com/compose/) is very
useful. It allows me to create one configuration file that contains all the
configuration needed to run the three containers my stack needs. Most of the
file is just the same data that went into the previously mentioned
`docker run` commands:

{% highlight yaml linenos %}
server:
  image: mikecobra/minecraft
  ports:
    - 25565:25565
  volumes:
    - /home/ubuntu/minecraft-data:/minecraft
    - /home/ubuntu/minecraft_server.16w06a.jar:/minecraft_server.jar
mapper:
  image: mikecobra/minecraft-overviewer
  volumes:
    - /home/ubuntu/minecraft-data/world:/world
    - /home/ubuntu/minecraft-map:/map
  restart: always
  mem_limit: 1G
  cpuset: "0"
web:
  image: mikecobra/lighttpd
  ports:
    - 80:80
  volumes:
    - /home/ubuntu/minecraft-map:/data
{% endhighlight %}

I added a bit of extra stuff into the configuration for the mapper image. A
restart policy of always meant I could keep regenerating the map. Every time
the process exits, having finished generating a map, it will restart and work
on updating the map.

This process is very CPU intensive though and the Minecraft server can be
quite memory intensive. As the performance of the Minecraft server was, to me,
more important than the frequency of map generation, I put limits on the CPU
and memory usage of the Overviewer process. The mem_limit is a simple cap on
the amount of memory that the container can use and the cpuset determines
which host CPUs the container is allowed to use.

Finally, to ease my interaction with the stack, I wrote an init-style wrapper
script around the docker-compose commands:

{% highlight bash linenos %}
 #!/bin/bash
COMPOSE_FILE=/home/ubuntu/minecraft-server/docker-compose.yml

case "$1" in
        start)
                COMPOSE_FILE=$COMPOSE_FILE docker-compose up -d
                ;;
        stop)
                COMPOSE_FILE=$COMPOSE_FILE docker-compose stop
                ;;
        reload|restart)
                $0 stop
                $0 start
                ;;
        logs)
                COMPOSE_FILE=$COMPOSE_FILE docker-compose logs
                ;;
        \*)
                echo "Usage: $0 start|stop|restart|reload|logs"
                exit 1
esac
exit 0
{% endhighlight %}

## Conclusion

It was reasonably easy for me to get started using Docker, I was pleased with
the results. However, the server was still running on my local desktop. Next
time, I'll talk about tackling AWS as a hosting platform for my Minecraft
server and in particular some performance problems I encountered.

Ultimately, it might've been easier for me to just run my server normally.
However, the learning experience was very useful and now I can easily
replicate my Minecraft server wherever it needs to live.

All of the Dockerfiles I created can be found on my Github and there are
automated Docker Hub builds putting the images onto the public Docker Hub.

[^1]: This is not strictly true. Originally I went for a JDK and realised I only needed a JRE while writing this blog post.
