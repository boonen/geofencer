## Table of Contents

* [Introduction](#introduction)
* [What is InfluxDB](#what-is-influxdb)
* [What is Grafana](#what-is-grafana)
* [The Dockerfile](#the-dockerfile)
* [Application Configuration Files](#application-configuration-files)
* [External Volumes](#external-volumes)
* [Building the Container](#building-the-container)
* [Running the Container](#running-the-container)
* [Accessing Grafana and InfluxDB](#accessing-grafana-and-influxdb)
* [Reading & Writing Data in InfluxDB](#reading-and-writing-data-in-influxdb)
* [Next Steps](#next-steps)

## Introduction

What I love about containers is how fast you can get up and running using a Dockerfile to automate the build of your image. This tutorial will walk you through the process of creating a Dockerfile that will utilize *supervisord* to run a combined install of InfluxDB and nginx for Grafana. At the end of this tutorial, you should have a container configured to accept server metrics information from *collectd*. 

## What is InfluxDB? 

InfluxDB is a time-series database that has been built to work best with metrics, events, and analytics. The solution is written completely in Go and relies on no external dependencies to run. It is being maintained [here](http://influxdb.com). 

## What is Grafana? 

Grafana is a metrics dashboard which plugs into solutions such as Graphite, InfluxDB, and OpenTSDB. You would use Grafana to visualize the various metrics data you are pushing into InfluxDB. In our case the type of metrics we're preparing for are system metrics we hope to gather using [collectd](http://www.collectd.org/).

## The Dockerfile

You can either create a folder and place your Dockerfile and all configuration files in it or you can clone these files from our repository [here](https://github.com/StackPointCloud/docker-influxdb). We recommend cloning so you capture all required configuration files, plus a pre-built Dockerfile. If you spot an issue or have an improvement, feel free to issue a PR, too!

The first lines of your Dockerfile should always be: 

    FROM ubuntu
    MAINTAINER Thomas Pynchon (pynchon@gravityrainbow.com)

This indicates what base image you are using and who maintains the Dockerfile. 

Next, you'll want to prepare the environment with all requisite software. We do this: 

    RUN \
      apt-get update && apt-get -y --no-install-recommends install \
        ca-certificates \
        software-properties-common \
        python-django-tagging \
        python-simplejson \
        python-memcache \
        python-ldap \
        python-cairo \
        python-pysqlite2 \
        python-support \
        python-pip \
        gunicorn \
        supervisor \
        nginx-light \
        nodejs \
        git \
        curl \
        openjdk-7-jre \
        build-essential \
        python-dev

Our next RUN block will install Grafana, InfluxDB, and do some basic configuration:

    WORKDIR /opt
    RUN \
      grafana_url=$(curl http://grafanarel.s3.amazonaws.com/latest.json | python -c 'import sys, json; print json.load(sys.stdin)["url"]') && \
      curl -s -o grafana.tar.gz $grafana_url && \
      curl -s -o influxdb_latest_amd64.deb http://s3.amazonaws.com/influxdb/influxdb_latest_amd64.deb && \
      mkdir grafana && \
      tar -xzf grafana.tar.gz --directory grafana --strip-components=1 && \
      dpkg -i influxdb_latest_amd64.deb && \
      echo "influxdb soft nofile unlimited" >> /etc/security/limits.conf && \
      echo "influxdb hard nofile unlimited" >> /etc/security/limits.conf

This downloads the latest version of Influx and Grafana (at the time of writing). 

Now we need to copy over the configuration files we've staged: 

    ADD config.js /opt/grafana/config.js
    ADD nginx.conf /etc/nginx/nginx.conf
    ADD supervisord.conf /etc/supervisor/conf.d/supervisord.conf
    ADD config.toml /opt/influxdb/current/config.toml

Finally, we map volumes, expose ports, and setup the run command: 

    VOLUME ["/opt/influxdb/shared/data"]

    EXPOSE 80 8083 8086 2003

    CMD ["supervisord", "-n"]

Putting it all together you should have a file similar to this: 

    FROM ubuntu
    MAINTAINER Thomas Pynchon (pynchon@gravityrainbow.com)

    RUN \
      apt-get update && apt-get -y --no-install-recommends install \
        ca-certificates \
        software-properties-common \
        python-django-tagging \
        python-simplejson \
        python-memcache \
        python-ldap \
        python-cairo \
        python-pysqlite2 \
        python-support \
        python-pip \
        gunicorn \
        supervisor \
        nginx-light \
        nodejs \
        git \
        curl \
        openjdk-7-jre \
        build-essential \
        python-dev


    WORKDIR /opt
    RUN \
      grafana_url=$(curl http://grafanarel.s3.amazonaws.com/latest.json | python -c 'import sys, json; print json.load(sys.stdin)["url"]') && \
      curl -s -o grafana.tar.gz $grafana_url && \
      curl -s -o influxdb_latest_amd64.deb http://s3.amazonaws.com/influxdb/influxdb_latest_amd64.deb && \
      mkdir grafana && \
      tar -xzf grafana.tar.gz --directory grafana --strip-components=1 && \
      dpkg -i influxdb_latest_amd64.deb && \
      echo "influxdb soft nofile unlimited" >> /etc/security/limits.conf && \
      echo "influxdb hard nofile unlimited" >> /etc/security/limits.conf

    ADD config.js /opt/grafana/config.js
    ADD nginx.conf /etc/nginx/nginx.conf
    ADD supervisord.conf /etc/supervisor/conf.d/supervisord.conf
    ADD config.toml /opt/influxdb/current/config.toml

    VOLUME ["/opt/influxdb/shared/data"]

    EXPOSE 80 8083 8086 2003

    CMD ["supervisord", "-n"]

## Application Configuration Files

We are using the following configuration files:

| Used by | Config File |
| --- | --- |
| Grafana | config.js |
| Supervisor | supervisord.conf | 
| Nginx | grafana.conf | 
| InfluxDB | config.toml |

If you cloned the repo then you should have these files; however, note, that some of the files must be edited with your IP address, desired database name, etc. 

### Grafana Config

The important one that needs to be altered is Grafana's *config.js*. You will need to update the HTTP endpoints to be an IP you can reach from the location where you will be using Grafana. 

      datasources: {
        influxdb: {
          type: 'influxdb',
          url: "http://yourpublicIP:8086/db/exampledb",
          username: 'root',
          password: 'root',
        },
        grafana: {
          type: 'influxdb',
          url: "http://yourpublicIP:8086/db/grafana",
          username: 'root',
          password: 'root',
          grafanaDB: true
        },
      },

In our demo we are connecting to the Grafana via the public Internet. 

Grafana is configured to use InfluxDB for its configurtion. While you can use the same db for both it is not recommended. 

### InfluxDB Config

We have enabled *graphite* in InfluxDB's *config.toml* by doing this: 

    [input_plugins]
      [input_plugins.graphite]
      enabled = true
      port = 2003
      udp_enabled = true
      database = "exampledb"

We tell InfluxDB to start the listener on 2003/UDP and set the database for the metrics.

## External Volumes

It is very important to understand the ephemeral nature of containers. This means that data created and stored within the container can disappear if the container restarts or is stopped. 

We expose the InfluxDB database path on this line: 

    VOLUME ["/opt/influxdb/shared/data/db"]

This allows us to pass the *-v* switch to *docker* to tell it to map the path in the container to a local path in our host system. When data is written to InfluxDB the it is now not stored within the container itself and is persistent beyond a restart. 

## Building the Container

Now that you have put your Dockerfile and configuration files together it is time to build your container. You will need to ensure you're working directory has your Dockerfile and configuration files in it. To build a container from your Dockerfile you would run the following command: 

    docker build -t influx .
    
The *-t* parameter tags the image with the name *influx*. The build will execute the Dockerfile. Inspect the output for any errors. If all looks good you should be ready to start the container up. 

## Running the Container

Building a container will not automatically start the container. You will need to do that next. Ensure the **/opt/influxdb** path exists on your host file system then run the following command. 

    docker run --name="influx" --hostname="influx" -d -v /opt/influxdb/:/opt/influxdb/shared/data -p 80:80 -p 8083:8083 -p 8086:8086 -p 2003:2003/udp influx

This tells Docker to:

1. Start the *influx* image;
2. Name it *influx*;
3. Map the container path of */opt/influxdb/shared/data* to your local */opt/influxdb*; 
4. Map your local port 80, 8083, and 8086 to the exposed ports in the container. 
5. Map port 2003 to UDP. *Collectd* will use UDP to communicate the data it is collecting. 

We need to explicitly instruct Docker to map 2003 to UDP so that *collectd* can communicate with the *graphite* listener. 

You can validate the container is running by issuing the command: 

    docker ps

## Accessing Grafana and InfluxDB

You should now be able to reach your Grafana site by plugging your into your browser: 

    http://yourpublicip
    
You can reach the InfluxDB management interface by going here: 

    http://yourpublicip:8083/

The default credential is *root* with the password *root*. You will want to change this as soon as you can. 

Before Grafana can pull data out of InfluxDB you will need to first create the databases that were defined in *config.js*. These would be *exampledb* and *grafana*. The Influx interface is relatively easy to use so you should be able to do this pretty quickly. 

## Reading and Writing Data in InfluxDB

You will need to test your InfluxDB installation by pre-populating some dummy data. Use the following command to push some data points into your *exampledb* database. Vary the values so you can create some nice graphs with peaks and troughs. You should be able to run these from the server where the container is running or remotely if you're using a public IP. 

    curl -X POST -d '[{"name":"foo","columns":["val"],"points":[[33]]}]' 'http://localhost:8086/db/exampledb/series?u=root&p=root'
    
This information is populating into our *exampledb* database which is the same one we defined in our Grafana *config.js* file. You can now define your Grafana queries and dashboard settings. 

To validate you can read data from InfluxDB you could do this: 

    curl -G 'http://localhost:8086/db/exampledb/series?u=root&p=root' --data-urlencode "q=select * from foo"

## Next Steps

At this point, you should have a single container using supervisor to run nginx for Grafana and InfluxDB as your backend data store. You can now setup and configure collectd to feed information into Influx. We will cover this in a different tutorial. We encourage you to play with this setup in your Data Center. 
