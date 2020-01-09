---
title: "Setting up an Elasticsearch, Fluentd and kibana (EFK) Stack"
date: 2020-01-05T18:09:00Z
categories: ['Code', 'Tutorials']
tags: ['Logging', 'DevOps']
author: "Michael Forde"
draft: false
---

When it comes to managing our currently small infrastructure, it became clear very early on that logging is vital.
taking the proper care in logging everything in the code base etc. was carried out, However we had one 
small problem, it was a very difficult job to go visit any or all of these logs, weather it be system logs, docker logs or
logs directly from our main application.

Heres where the journey to look into implementing a centralized logging system began. There was a few important key features we needed,

<!--more-->

1. The ability to store and collect logs of any kind by pulling from any location at any time.

2. Using open source technology to achive all this. 

3. Being able to filter, aggregate, compare and analyze the logs.


After doing a bit of research, I found a nice group of components which make up a stack called `EFK` named after
the following components respectivally,


**Elasticsearch**: An open source distributed, RESTful search and analyics engine.

**Fluentd**: An open source data collector with tones of plugins.

**Kibana**: A web UI for Elasticsearch.





__These are the steps to mannually set up the EFK stack:__


We will be setting up each component in its own Docker container. Docker enables us to deploy
each component faster and gives us more control over the behaviour of the components individually.

- Increase the `mmap` limits by running the following command as root:

This is required to stop and prevent Elasticsearch from crashing. 

```sh
sudo sysctl -w vm.max_map_count=262144
```

- Set up a docker `network`:

We must set up a docker network to allow the containers communicate with each other, this
is achieved by executing the following docker command.

```sh
sudo docker network create efk
```

- Depoly Elasticsearch using the following docker command:

```sh
sudo docker run --network=efk --name elasticsearch -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e "cluster.name=docker-cluster" -e "bootstrap.memory_lock=true" -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" --ulimit memlock=-1:-1 -v elasticdata:/usr/share/elasticsearch/data docker.elastic.co/elasticsearch/elasticsearch-oss:6.2.2
```

You can verify if this container is running using the following command.

```sh
sudo docker ps
```

This should output a response similar to this.

```sh
╰─❯ sudo docker ps 
CONTAINER ID        IMAGE                                                     COMMAND                  CREATED             STATUS              PORTS                                            NAMES
dd2add2fb581        docker.elastic.co/elasticsearch/elasticsearch-oss:6.2.2   "/usr/local/bin/dock…"   28 minutes ago      Up 28 minutes       0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp   elasticsearch

```

- Deploying Kibana using the following command

```sh
sudo docker run --network=efk --name kibana -d -p 5601:5601 docker.elastic.co/kibana/kibana-oss:6.2.2
```

Verify using the same command again to check if the container has started.

```sh
╰─❯ sudo docker ps
CONTAINER ID        IMAGE                                                     COMMAND                  CREATED              STATUS              PORTS                                            NAMES
7bcb4595347b        docker.elastic.co/kibana/kibana-oss:6.2.2                 "/bin/bash /usr/loca…"   About a minute ago   Up About a minute   0.0.0.0:5601->5601/tcp                           kibana
dd2add2fb581        docker.elastic.co/elasticsearch/elasticsearch-oss:6.2.2   "/usr/local/bin/dock…"   36 minutes ago       Up 36 minutes       0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp   elasticsearch

```

- Configuring Fluentd to collect logs

Create a directory structure for Fluentd configuration.

```sh
mkdir -p fluentd/plugins
```

Now lets create a Fluentd config file.

```sh
nano fluentd/fluent.conf
```

For example, I will use the configuration to pull logs from docker containers, paste this
example into your `fluentd/fluent.conf`. 

```xml
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<match *.**>
  @type copy
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix fluentd
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name access_log
    tag_key @log_name
    flush_interval 1s
  </store>
  <store>
    @type stdout
  </store>
</match>
```

- Setting up a `Dockerfile` for fluentd

Create another file in the `fluentd` directory called `Dockerfile`.

```sh
nano fluentd/Dockerfile
```

Adding in the following config to build a docker image.

```sh
FROM fluent/fluentd:v0.12-onbuild

RUN apk add --update --virtual .build-deps \
        sudo build-base ruby-dev \
 && sudo gem install \
        fluent-plugin-elasticsearch \
 && sudo gem sources --clear-all \
 && apk del .build-deps \
 && rm -rf /var/cache/apk/* \
           /home/fluent/.gem/ruby/2.3.0/cache/*.gem
```

Now the directory should look like the following.

```sh
╰─❯ ls fluentd
Dockerfile  fluent.conf  plugins
```

- Building docker image for fluentd

 Using the follwoing command you can build the previously made `Dockerfile`, it should create
 an `Image ID` as shown, keep note of this `Image ID` as we will use it for the next command.

 ```sh
 sudo docker build fluentd/

 ...
 ...

 Successfully built <Image ID>
 ```

- Deploying fluentd

Take note of the `Image ID` from the last step, and run the following command with the `Image ID`
included to deploy the fluentd container.

```sh
sudo docker run -d --network efk --name fluentd -p 24224:24224/tcp -p 42185:42185/udp <Image ID>
```

- Checking if it works

First go and see if kibana is up by going to the port you have set it,
in our case it is set as following.

```sh
http://0.0.0.0:5601
```

Now send a message to fluentd using docker as follows.

```sh
docker run --log-driver=fluentd --log-opt tag="docker.{.ID}}" ubuntu echo 'Hello Fluentd!'
```

This should not be visible in the Kibana Dashboard.