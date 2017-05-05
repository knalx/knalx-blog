---
layout: post
title:  "Elasticsearch 5.3.0 in Docker on transport port."
date:   2017-04-09 14:07:36 +0300
categories: elasticsearch
---

If you want to tun integration test with Elasticserach in Java application you need to do this.


Run it on transport port (9300) - its not so easy as you think
1. You need set correct hosts
```
 docker run  -p 9300:9300 -p 9200:9200 \
 -e "xpack.security.enabled=false"  -e "network.host=_site_" -e "network.publish_host=_local_" \ 
 docker.elastic.co/elasticsearch/elasticsearch:5.3.0
 ```
 
2. You need to set virtual memory config on container host. [More here](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html)
 ```
 sysctl -w vm.max_map_count=262144
 ```
Because if you not - you will get the error.
```
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```
And you cant change this policy becouse dev mod is triggered by binding transport to an external interface.
[Doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html)

Just use port 9200 - its much more simple.
