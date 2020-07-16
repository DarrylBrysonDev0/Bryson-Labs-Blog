---
title: "Docker Walkthrough: ELK Resource Monitor Part-1"
date: 2020-07-10T10:21:30-04:00
categories:
  - Resource-Monitor
tags:
  - docker
  - walkthrough
  - ELK
excerpt: "ELK Resource Monitor w/ Docker: Part-1 Docker CLI"
---

*First in the series exploring the use of docker through a practicle Use Case.*

Welcome to the first in a series of posts that will follow my recent training with Docker. Most of my experiance developing enterprise solutions has been gained in the automotive industry through varying engineering roles. During the on going Pandemic, I decided to take the extra time I had and learn Docker. Primarily with a focus on how to apply modern DevOps tools and concepts to the Monilithic/data-center based/regulated solutions I've built most my career.

Docker solves one of my biggest problems in building BI solutions, varying deployment environments. When creating BI solutions the name of the game is flexability and speed. Solutions should be customized and local to the business that needs it. Even within the same department software and hardware used can vary greatly. Docker allows a developer to build an app and without concern, mostly, for the platform it'll be deployed to.

## The Setup
The setup I'm using is docker for windows API version 1.40 on Windows 10 Home. We'll be using Linux containers. I'm also using PowerShell 7 in Windows Terminal. If you're following allong on a Linux, commands are generally the same excluding some OS specific syntax (file path, line continueation).     

In this post we'll cover some basic docker CLI concepts by building a simple resource monitoring dashboard using ELK tools. This use case uses the following design pattern:

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/ELK-resource-monitor-design-pattern-wht-bkg.png" alt="ELK-resource-monitor-design-pattern-wht-bkg">

Solution Components:
1. **Docker Server:** Docker host running all containers for this solution.
2. **Application Server:** Ubuntu server running unrelated workloads.
3. **MetricBeats:** Lightweight shipper that sends host metric data to a central data store.
4. **Elasticsearch:** Document database specilized for timeseries and log processing use cases
5. **Kibana:** Mangament panel for Elasticsearch indices
6. **Grafana:** Monitoring platform for creating dashboards

## Build Process
Process: 
1. Create Docker network
2. Create Elasticsearch container
3. Create Kibana container 
4. Install Metricbeats
    - Docker Server
    - Application Server (optional)
5. Create Grafana container
6. Config index pattern
7. Configure Grafana

### 1. Create Docker Network 
We'll first need to create a network for our containers to communicate on. By default if a network isn't specified the `docker0` network would be used. it's generaly good practice to create a project specific network to provide some isolation from other projects you may have running. If you'd like additional information on this command check Docker's documentation [here](https://docs.docker.com/engine/reference/commandline/network_create/). The command for creatinga named network is straight forward:

```powershell
docker network create resource-monitor-net
```

### 2. Create Elasticsearch Container
Elasticsearch is the core component for this solution. Elasticsearch is a highly scalable and performant document database customized for index searches. It's also been very popular option for powering log processing use cases. Many tools have created Elaticsearch interfaces and the API is simple/stable enough to create your own if needed. I've found that Elasticsearch is super simple to spin up and down in docker. When configured right Elasticsearch makes for a great testing timeseries db.

We can use a docker run command to create the Elasticsearch container:

```powershell
docker container run -d --name elasticsearch  `
    --net resource-monitor-net  `
    -p 9200:9200  `
    -p 9300:9300  `
    -e "discovery.type=single-node"  `
    elasticsearch:7.7.1
```
I've split the command across multipule lines to highlight the parameters. Let's examine each parameter: 
<!-- breakdown run, -d, --name, --net , -p, -e, specific image versions -->
- `docker container run` --> This is the base command needed to create a container
- `-d` --> Adding this flag signals docker to run the container in `detachted` mode. this keeps the container outputs in the background.
- `--name elasticsearch` --> Names the container `elasticsearch`
- `-p 9200:9200 -p 9300:9300` --> Connects the ports on the *host* (computer running docker) and the ports internal to the container. -p or -publish follows the pattern `<host_port>:<container_port>`
- `-e "discovery.type=single-node"` --> Sets the containers environment variable. Environment variables are used to pass configuration parametes into the containers. In this case we must tell elasticsearch that it's running in singal node mode.
- `elasticsearch:7.7.1` --> Is the 7.7.1 version of the elasticsearch image. It's a good habit for dev or production to specify image versions. Alternativly the images could be specified `elasticsearch:latest` this would change the image that's downloaded to the most current release. In most cases this causes  breaking changes.  

###  3. Create Kibana Container 
Next well create a Kibana container to act as an admin panel for Elasticsearch. Kibana is intended to be a visualisation tool for Elasticsearch. I find the dashboarding components to be a bit difficault to work with and isn't very deep in features. 

The command to create a Kibana container:

```powershell
docker container run -d --name kibana  `
    --net resource-monitor-net  `
    -p 5601:5601  `
    -e SERVER_NAME='kibana_bryson-labs'  `
    -e ELASTICSEARCH_HOSTS='http://elasticsearch:9200'  `
    kibana:7.7.1
```
Comparing this command with the one for Elasticseach you can see that we've connected to the same network this time on port `5601`. We've also set 2 environment variables.
- SERVER_NAME : seting the server name internal to kibana as 
breakdown kibana spcific env variables 'kibana_bryson-labs'
- ELASTICSEARCH_HOSTS : Defines what host and port to looks for Elasticsearch on.In this example you see one of the advatages to using a named network, container name resolution. Docker does some cool things with networking under the hood, one of them is a DNS that allows for container name resolution. With this a container can reference another container by name, provided they're on the same network. This is much better than the alternative of hard coding IP addresses. 

### 4. Install Metricbeats - Docker Server
After creating the Elasticsearch database and Kibana to mantain it, we can start feeding it our host's metrics. This solution makes use of MetricBeats as a simple way of sending `metric sets` (CPU, RAM, and Disk utilization) in near real-time. 

The command to create a MetricBeats container:

```powershell
docker container run -d --name=metricbeat `
    --hostname=docker-server  `
    --net resource-monitor-net  `
    -e setup.kibana.host=http://kibana:5601  `
    -e output.elasticsearch.hosts=["http://elasticsearch:9200"]  `
    --user=root `
    --volume="/var/run/docker.sock:/var/run/docker.sock:ro" `
    --volume="/sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro" `
    --volume="/proc:/hostfs/proc:ro" `
    --volume="/:/hostfs:ro" `
    docker.elastic.co/beats/metricbeat:7.7.1
```

Docker by default isolates the container's filesystem from its host's. We need to give MetricBeats some visibility to the underlying host. Otherwise it can only collect metrics from it's self. The Docker way of exposing the filesystem is to create a bind-mount using the --volume flag. Bind-mounts follow the pattern `<host/source/path> : <container/target/path> : <permissions>`. 

--- http://www.floydhilton.com/docker/2017/03/31/Docker-ContainerHost-vs-ContainerOS-Linux-Windows.html#:~:text=Container%20Host%3A%20Also%20called%20the,kernel%20with%20running%20Docker%20containers.&text=Note%20that%20windows%20containers%20require,while%20Linux%20containers%20do%20not. ---

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/bind-mount-diagram.png" alt="bind-mount-diagram">


### 4. Install Metricbeats - Application Server (Optional)
Optionally you can install MetricBeats on another computer or vm. This would approximate putting MetriBeats on an application server. Where your interested on monitoring the resouces of a remote host as it processes work loads. 

Installation of MetricBeats on our application server is a bit more involved than the Docker install. Installation is platform specific when not using Docker, you can find the offical instructions [here](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-installation.html). 

This illistrats the biggest value propissition for Docker. Before container technologies like Docker, making a solution platform agnostic was a massive undertaking. Now a solution can be built for Docker, and Docker handles the platform integration.

After installing MetricBeats the configuration file will need to be replaced. 
1. Download the example config [example.metricbeat.yml]({{ site.url }}{{ site.baseurl }}/assets/files/resource-monitor/example.metricbeat.yml)
2. Replace `<Docker-Server-IP>` with the ip address of the Docker server (Section to edit shown below)
3. Rename the file metricbeats.yml and save to your config folder, The location of the config folder depends on the platform. For my Ubuntu host, the config folder is `/etc/metricbeats`. You can find your directory layout [here](https://www.elastic.co/guide/en/beats/metricbeat/current/directory-layout.html)

```yaml
#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["<Docker-Server-IP>:9200/"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"

#----------------------------- Logstash output --------------------------------
``` 
*example.metricbeat.yml segment too edit*

### 5. Install Metricbeats - Docker Server
Create Grafana container:

```powershell
docker container run -d --name grafana `
    --net resource-monitor-net  `
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
