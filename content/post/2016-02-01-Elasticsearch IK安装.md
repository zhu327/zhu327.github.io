---
date: 2016-02-01T10:25:40+08:00
title: Elasticsearch IK安装
tags: ["elasticsearch"]
---

** 本文描述的的安装环境均为Ubuntu 14.04 64bit

#### 1. 安装Elasticsearch

参考
> https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-elasticsearch-on-ubuntu-14-04

1. 安装OpenJDK;


```shell
    sudo apt-get install openjdk-7-jre
```
<!--more-->

2. 下载并安装Elasticsearch;
    > https://www.elastic.co/downloads/elasticsearch
    
    当前最新版本为2.0,本文以2.0版本为准

    ```shell
    sudo dpkg -i elasticsearch-2.0.0.deb
    ```

    安装完成后会创建`/usr/share/elasticsearch`目录,`/etc/elasticsearch`目录为配置文件目录,`/etc/init.d/elasticsearch`启动文件.
3. 启动Elasticsearch

	```shell
    sudo /etc/init.d/elasticsearch start
    sudo service elasticsearch start
    ```

4. 验证安装

    ```shell
    curl 'http://localhost:9200/?pretty'
    ```

#### 2. 简单配置Elasticsearch

Elasticsearch的配置文件保存在`/etc/elasticsearch/elasticsearch.yml`中,配置项都有注释,这里我们只配置一些基础的项.

1. cluster.name
    集群名称,Elasticsearch会自动查找在同一网段的同一集群名的node来组成搜索集群,这里我们只有单node,所有任意配置一个需要集群名即可

2. node.name
	节点名称,在集群中的单节点名,同一集群的节点名不能一样

3. node.master
	在同一个集群中的节点,分为master与slave,master节点主要是维持集群状态健康,不负责数据存取与索引建立的工作,所以不会过载.slave负责处理数据任务,即使slave过载,只要集群中有其它的节点健康,不会影响到整个节点.
    这个配置只能设置为true或者false,这里我们使用单节点,所以不用设置,默认就为true,须知,一般我们的出口ip节点设置为master,且集群中至少要有一个master.

4. node.data
	true或者false,设置当前节点是否存储索引数据,一般我们不需要配置这项,默认为true,但是如果在集群中,出口ip节点为master,我们会设置node.data为false,这样出口node就会只负责从slave上抓取数据,聚合结果.可以称之为"search load balancer".

5. 其它
	index.number_of_shards: 索引分片数
    index.number_of_replicas: 索引副本数
    比较多的分片可以改善建索引的性能,比较多的副本可以改善搜索速度.
    path.data: 索引数据存储路径,默认为/var/lib/elasticsearch

6. 安全相关
    network.bind_host: localhost 设置服务node的ip
    script.disable_dynamic: true 禁止远程执行脚本

#### 3. 使用Elasticsearch

1. 创建索引

	```shell
    curl -X POST 'http://localhost:9200/tutorial/helloworld/1' -d '{ "message": "Hello World!" }'
    ```

2. 简单检索
	通过id检索:

    ```shell
    curl 'http://localhost:9200/tutorial/helloworld/1'
    ```

    搜索字段:

    ```shell
    curl -X GET 'http://localhost:9200/tutorial/helloworld/_search?q=message:Hello&pretty'
    ```

3. 使用DSL查询语句

    ```javascript
    {
        "query" : {
            "match" : {
                "last_name" : "Smith"
            }
        }
    }
    ```

    > https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html

待续...

```shell
export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-amd64
sudo apt-get install openjdk-7-jdk
sudo apt-get install openjdk-7-jre-headless
sudo apt-get install openjdk-7-jre-lib
```
