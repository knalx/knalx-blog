---
layout: post
title:  "Elasticsearch 5.3.0 in Docker on transport port."
date:   2017-04-09 14:07:36 +0300
categories: elasticsearch
---

If you want to run integration test with Elasticserach in Java application you need to do this.

[Here](http://www.hascode.com/2016/08/elasticsearch-integration-testing-with-java/) is amazing article about ElasticSearch test running.



## Test with Docker

If you want elastic in docker container you would get some problems:
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


## Test with ElasticSearch test framework

Much more easier is to use special framework

Add dependencies to pom.xml
```
 <dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>transport</artifactId>
    <version>${elasticsearch.version}</version>
</dependency>

<dependency>
<groupId>org.elasticsearch.test</groupId>
    <artifactId>framework</artifactId>
    <version>${elasticsearch.version}</version>
    <scope>test</scope>
</dependency>
```

Write a test class

 ```
 @Ignore
 public class ElasticSearchTest {
 
     private static String INDEX = "test";
     private static String TYPE = "party";
 
     private TransportClient client;
 
     private NodeClient nodeClient;
 
     @Before
     public void before() throws UnknownHostException, InterruptedException {
         Settings settings = Settings.builder()
                 .put("client.transport.ignore_cluster_name", true)
                 .build();
         client = new PreBuiltTransportClient(settings)
                 .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("localhost"), 9300));
     }
 
     @Test
     public void testEs() throws IOException {
         Party party = ThriftGenerator.buildRandomParty();
         String json = ThriftHelper.convertToJson(party);
         IndexResponse indexResponse = client.prepareIndex(INDEX, TYPE).setSource(json, XContentType.JSON).get();
         System.out.println("Insert :  " + indexResponse.status());
     }
 
 
     @Test
     public void testSearch() {
         SearchResponse response2 = client.prepareSearch(INDEX)
                 .setTypes(TYPE)
                 .setSearchType(SearchType.DFS_QUERY_THEN_FETCH)
                 .setQuery(QueryBuilders.matchQuery("_all", "ХвоСт"))                 // Query
                 .setFrom(0).setSize(60).setExplain(true)
                 .get();
         System.out.println("Search: " + response2.getHits().getHits().length);
     }
 
     @After
     public void after() {
         client.close();
     }
 
 
 }
 ```
