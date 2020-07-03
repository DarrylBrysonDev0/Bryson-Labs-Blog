---
title: "Docker Walkthrough: ELK Resource Monitor Part-1"
date: 2020-07-02T10:21:30-04:00
categories:
  - Resource-Monitor
tags:
  - docker
  - walkthrough
  - ELK
---

*First in the series exploring the use of docker through a practicle Use Case.*

Welcome to the first in a series of posts that follows my recent training with Docker. I developed a specific as an enterprise developer, having most of my experiance in automotive engineering/quality. During the on going Pandemic, I decided to take the extra time I had to learn Docker. Primarily with a focus on how to apply modern DevOps tools and concepts to the Monilithic/data-center based/regulated solutions I've built most my career.

I do assume ----------------- Assume you know what and why for Docker -----------------

----------------- My_Docker_Setup_&_Windows_Setup -----------------

In this post we'll cover the basic docker CLI concepts by building a simple resource monitoring dashboard using ELK tools. Will be using the following design pattern:

----------------- Resource_Monitor_Design_Pattern -----------------

----------------- Breakdown_Pattern_Elements ----------------------

----------------- Benefits_of_Solution_Componets = Containers -----

## Learning Objectives
The learning objectives for this post are:
1. Explore docker basics for running a sequince of interacting containers
    - network (create)
    - container (run)
2. High level introduction to the ELK stack
    - Elasticsearch
    - MetricBeats
    - Kibana
    - Grafana

## Walkthrough Contents
Process: 
1. Create Docker network
2. Create Elasticsearch container
3. Create Kibana container 
4. Install Metricbeats
    - docker container
    - Ubuntu
5. Create Grafana container
6. Config index pattern
7. Configure Grafana

We'll first need to create a network for our containers to communicate on. By default if a network isn't specified the `docker0` would be used. Check Docker documentation [here](https://docs.docker.com/engine/reference/commandline/network_create/) for more detailes. Creating a named network in docker is a simple command:

```powershell
docker network create bryson-lab-net
```

The primary component that this solution is based around is Elasticsearch. Elasticsearch is a highly scalable and performant document database customized for index searches. It's also been very popular for log processing use cases. I've found that Elasticsearch is super simple to spin up and down in docker. When configured it right Elasticsearch makes for a good testing timeseries db.

We can use a docker run command to create the Elasticsearch container:

```powershell
docker container run -d --name elasticsearch  `
    --net bryson-lab-net  `
    -p 9200:9200  `
    -p 9300:9300  `
    -e "discovery.type=single-node"  `
    elasticsearch:7.7.1
```
I've split the command across multipule lines to highlight the parameters. In this command we: <!--breakdown run, -d, --name, --net , -p, -e, specific image versions-->
- `docker container run` --> base command to create a container
- `-d` --> run the container in `detachted` mode, this runs the container in the background from the terminals perspective
- `--name elasticsearch` --> names the container elasticsearch
- `-p 9200:9200 -p 9300:9300` --> connects the ports on the *host* (computer running docker) and the ports internal to the container. -p or -publish follows the pattern *host_port:container_port*
- `-e "discovery.type=single-node"` --> sets the containers environment variable. Environment variables  pass configuration parametes into the containers.
- `elasticsearch:7.7.1` --> Is the 7.7.1 version of the elasticsearch image. It's a good habit for dev or production to specify image versions. Alternativly the images could be specified `elasticsearch:latest` this would change the image that's downloaded to the most current release. In most cases that results in breaking changes.  

Create Kibana container:

```powershell
docker container run -d --name kibana  `
    --net bryson-lab-net  `
    -p 5601:5601  `
    -e SERVER_NAME='kibana_bryson-labs'  `
    -e ELASTICSEARCH_HOSTS='http://elasticsearch:9200'  `
    kibana:7.7.1
```
breakdown kibana spcific env variables

Create MetricBeats container:

```powershell
docker container run -d --name=metricbeat `
    --hostname=docker-host-01  `
    --net bryson-lab-net  `
    -e setup.kibana.host=http://kibana:5601  `
    -e output.elasticsearch.hosts=["http://elasticsearch:9200"]  `
    --user=root `
    --volume="/var/run/docker.sock:/var/run/docker.sock:ro" `
    --volume="/sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro" `
    --volume="/proc:/hostfs/proc:ro" `
    --volume="/:/hostfs:ro" `
    docker.elastic.co/beats/metricbeat:7.7.1
```
breakdown hostname, --volume, bind mounts, --user

Install MetricBeats to ubuntu:
- https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-installation.html
- customize metricbeat.yml

```yaml
#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["<ES-Hostname-IP>:9200/"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"

#----------------------------- Logstash output --------------------------------
```

breakdown yaml comment tags, yaml use as a config for many linux services.

Create Grafana container:

```powershell
docker container run -d --name grafana `
    --net bryson-lab-net  `
    -p 3000:3000  `
    --volume lab_dash_data:/var/lib/grafana  `
    --volume "${pwd}\Grafana\grafana.ini:/etc/grafana/grafana.ini"  `
    -e GF_PATHS_CONFIG=/etc/grafana/grafana.ini  `
    grafana/grafana:7.0.3
```
breakdown ${pwd}, include grafana.ini file

## Configure
- Config Index Pattern
    - pattern name (wildcard pattern) [metricbeat-7*]
    - set timestamp field (@timestamp)
- Create "Elasticsearch" datasource (on Grafana)
    * set url http://localhost:9200/ or http://elasticsearch:9200/ or http://<IP>:9200/
    * Pattern = No pattern
    * Index name = metricbeat-7*
    * Version = 7.0+
- Import Grafana Dashboard
