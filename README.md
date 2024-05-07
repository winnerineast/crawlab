# Craw Management to harvest data over Internet (Crawlab)


[中文](README-zh.md) | English

[Installation](#installation) | [Run](#run) | [Screenshot](#screenshot) | [Architecture](#architecture) | [Integration](#integration-with-other-frameworks)| [CHANGELOG](CHANGELOG.md) | [Disclaimer](DISCLAIMER.md)

Golang-based distributed web crawler management platform, supporting various languages including Python, NodeJS, Go, Java, PHP and various web crawler frameworks including Scrapy, Puppeteer, Selenium.

## Installation

You can follow the [installation guide](crawlab-docs/docs/en/guide/installation/README.md).

## Quick Start

Please open the command line prompt and execute the command below. Make sure you have installed `docker-compose` in advance.

```bash
cd deployment_examples/docker/basic
docker-compose up -d
```

Next, you can look into the `docker-compose.yml` (with detailed config params) and the [Documentation](crawlab-docs/docs/en/README.md) for further information. 

## Run

### Docker

Please use `docker-compose` to one-click to start up. By doing so, you don't even have to configure MongoDB database. Create a file named `docker-compose.yml` and input the code below.


```yaml
version: '3.3'
services:
  master: 
    image: crawlabteam/crawlab:latest
    container_name: crawlab_example_master
    environment:
      CRAWLAB_NODE_MASTER: "Y"
      CRAWLAB_MONGO_HOST: "mongo"
    volumes:
      - "./.crawlab/master:/root/.crawlab"
    ports:    
      - "8080:8080"
    depends_on:
      - mongo

  worker01: 
    image: crawlabteam/crawlab:latest
    container_name: crawlab_example_worker01
    environment:
      CRAWLAB_NODE_MASTER: "N"
      CRAWLAB_GRPC_ADDRESS: "master"
      CRAWLAB_FS_FILER_URL: "http://master:8080/api/filer"
    volumes:
      - "./.crawlab/worker01:/root/.crawlab"
    depends_on:
      - master

  worker02: 
    image: crawlabteam/crawlab:latest
    container_name: crawlab_example_worker02
    environment:
      CRAWLAB_NODE_MASTER: "N"
      CRAWLAB_GRPC_ADDRESS: "master"
      CRAWLAB_FS_FILER_URL: "http://master:8080/api/filer"
    volumes:
      - "./.crawlab/worker02:/root/.crawlab"
    depends_on:
      - master

  mongo:
    image: mongo:4.2
    container_name: crawlab_example_mongo
    restart: always
```

Then execute the command below, and Crawlab Master and Worker Nodes + MongoDB will start up. Open the browser and enter `http://localhost:8080` to see the UI interface.

```bash
docker-compose up -d
```

For Docker Deployment details, please refer to [relevant documentation](crawlab-docs/docs/en/guide/installation/docker.md).

## Screenshot

#### Login

![](screenshots/20210729/screenshot-login.png)

#### Home Page

![](screenshots/20210729/screenshot-home.png)

#### Node List

![](screenshots/20210729/screenshot-node-list.png)

#### Spider List

![](screenshots/20210729/screenshot-spider-list.png)

#### Spider Overview

![](screenshots/20210729/screenshot-spider-detail-overview.png)

#### Spider Files

![](screenshots/20210729/screenshot-spider-detail-files.png)

#### Task Log

![](screenshots/20210729/screenshot-task-detail-logs.png)

#### Task Results

![](screenshots/20210729/screenshot-task-detail-data.png)

#### Cron Job

![](screenshots/20210729/screenshot-schedule-detail-overview.png)

## Architecture

The architecture of Crawlab is consisted of a master node, worker nodes, [SeaweedFS](https://github.com/chrislusf/seaweedfs) (a distributed file system) and MongoDB database. 

![](screenshots/20210729/crawlab-architecture-v0.6.png)

The frontend app interacts with the master node, which communicates with other components such as MongoDB, SeaweedFS and worker nodes. Master node and worker nodes communicate with each other via [gRPC](https://grpc.io) (a RPC framework). Tasks are scheduled by the task scheduler module in the master node, and received by the task handler module in worker nodes, which executes these tasks in task runners. Task runners are actually processes running spider or crawler programs, and can also send data through gRPC (integrated in SDK) to other data sources, e.g. MongoDB.

### Master Node

The Master Node is the core of the Crawlab architecture. It is the center control system of Crawlab.

The Master Node provides below services:
1. Task Scheduling;
2. Worker Node Management and Communication;
3. Spider Deployment;
4. Frontend and API Services;
5. Task Execution (you can regard the Master Node as a Worker Node)

The Master Node communicates with the frontend app, and send crawling tasks to Worker Nodes. In the mean time, the Master Node uploads (deploys) spiders to the distributed file system SeaweedFS, for synchronization by worker nodes.

### Worker Node

The main functionality of the Worker Nodes is to execute crawling tasks and store results and logs, and communicate with the Master Node through gRPC. By increasing the number of Worker Nodes, Crawlab can scale horizontally, and different crawling tasks can be assigned to different nodes to execute.

### MongoDB

MongoDB is the operational database of Crawlab. It stores data of nodes, spiders, tasks, schedules, etc. Task queue is also stored in MongoDB.

### SeaweedFS

SeaweedFS is an open source distributed file system authored by [Chris Lu](https://github.com/chrislusf). It can robustly store and share files across a distributed system. In Crawlab, SeaweedFS mainly plays the role as file synchronization system and the place where task log files are stored. 

### Frontend

Frontend app is built upon [Element-Plus](https://github.com/element-plus/element-plus), a popular [Vue 3](https://github.com/vuejs/vue-next)-based UI framework. It interacts with API hosted on the Master Node, and indirectly controls Worker Nodes. 

## Integration with Other Frameworks

[Crawlab SDK](https://github.com/crawlab-team/crawlab-sdk) provides some `helper` methods to make it easier for you to integrate your spiders into Crawlab, e.g. saving results.

### Scrapy

In `settings.py` in your Scrapy project, find the variable named `ITEM_PIPELINES` (a `dict` variable). Add content below.

```python
ITEM_PIPELINES = {
    'crawlab.scrapy.pipelines.CrawlabPipeline': 888,
}
```

Then, start the Scrapy spider. After it's done, you should be able to see scraped results in **Task Detail -> Data**

### General Python Spider

Please add below content to your spider files to save results.

```python
# import result saving method
from crawlab import save_item

# this is a result record, must be dict type
result = {'name': 'crawlab'}

# call result saving method
save_item(result)
```

Then, start the spider. After it's done, you should be able to see scraped results in **Task Detail -> Data**

### Other Frameworks / Languages

A crawling task is actually executed through a shell command. The Task ID will be passed to the crawling task process in the form of environment variable named `CRAWLAB_TASK_ID`. By doing so, the data can be related to a task.
