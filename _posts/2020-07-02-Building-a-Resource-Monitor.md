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

In this series will be exploring some basic concept in docker and docker-compose.

## Learning Objectives
Define
- Docker CLI
    - network (create)
    - container (run)
- ELK
    - Elasticsearch
    - MetricBeats
    - Kibana
    - Grafana


## Walkthrough Start
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

We'll first need to create a Docker Overlay network:

```powershell
docker network create bryson-lab-net
```
breakdown network

Creating Elasticsearch container:

```powershell
docker container run -d --name elasticsearch  `
    --net bryson-lab-net  `
    -p 9200:9200  `
    -p 9300:9300  `
    -e "discovery.type=single-node"  `
    elasticsearch:7.7.1
```
breakdown run, -d, --name, --net , -p, -e, specific image versions

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
