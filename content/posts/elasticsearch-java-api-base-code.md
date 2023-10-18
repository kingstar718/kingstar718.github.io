---
author: "kingstar718"
title: 'ElasticSearch Java API 基本操作'
date: 2023-10-18T16:43:24Z
tags: ["java","elasticsearch"]
draft: false
---

## 前言

ElasticSearch Java API是ES官方在8.x版本推出的新java api，也可以适用于7.17.x版本的es。

本文主要参考了相关博文，自己手动编写了下相关操作代码，包括更新mappings等操作的java代码。

代码示例已上传[github](https://github.com/kingstar718/Java-Learning/blob/master/tools/src/main/java/top/wujinxing/elasticsearch/EsUtils.java)。

## 版本

1. `elasticsearch`版本：`7.17.9`，修改 `/elasticsearch-7.17.9/config/elasticsearch.yml`，新增一行配置：`xpack.security.enabled: false`，避免提示
2. `cerebro`版本：`0.8.5`（浏览器连接es工具）
3. `jdk`版本：`11`
4. `elasticsearch-java`版本：

```xml
<dependency>  
    <groupId>co.elastic.clients</groupId>  
    <artifactId>elasticsearch-java</artifactId>  
    <version>7.17.9</version>  
</dependency>  
  
<dependency>  
    <groupId>com.fasterxml.jackson.core</groupId>  
    <artifactId>jackson-databind</artifactId>  
    <version>2.12.3</version>  
</dependency>
```

## 连接

```java
public static ElasticsearchClient getEsClient(String serverUrl) throws IOException {  
    RestClient restClient = RestClient.builder(HttpHost.create(serverUrl)).build();  
    ElasticsearchTransport transport = new RestClientTransport(restClient, new JacksonJsonpMapper());  
    ElasticsearchClient esClient = new ElasticsearchClient(transport);  
    log.info("{}", esClient.info());  
    return esClient;  
}
```

## 索引

### 创建索引

```java
public static void createIndex(ElasticsearchClient esClient, String indexName) throws IOException {  
    if (existsIndex(esClient, indexName)) {  
        log.info("index name: {} exists!", indexName);  
    } else {  
        CreateIndexResponse response = esClient.indices().create(c -> c.index(indexName));  
        log.info("create index name: {}, ack: {}", indexName, response.acknowledged());  
    }  
}

// 判断索引是否存在
public static boolean existsIndex(ElasticsearchClient esClient, String indexName) throws IOException {  
    BooleanResponse exists = esClient.indices().exists(c -> c.index(indexName));  
    return exists.value();  
}
```

### 查询索引

```java
public static void getIndex(ElasticsearchClient esClient, String indexName) throws IOException {  
    GetIndexResponse getIndexResponse = esClient.indices().get(s -> s.index(indexName));  
    Map<String, IndexState> result = getIndexResponse.result();  
    result.forEach((k, v) -> log.info("get index key: {}, value= {}", k, v));  
  
    // 查看全部索引  
    IndicesResponse indicesResponse = esClient.cat().indices();  
    indicesResponse.valueBody().forEach(i ->  
            log.info("get all index, health: {}, status: {}, uuid: {}", i.health(), i.status(), i.uuid()));  
}
```

### 删除索引

```java
public static void delIndex(ElasticsearchClient esClient, String indexName) throws IOException {  
    if (existsIndex(esClient, indexName)) {  
        log.info("index: {} exists!", indexName);  
        DeleteIndexResponse deleteIndexResponse = esClient.indices().delete(s -> s.index(indexName));  
        log.info("删除索引操作结果：{}", deleteIndexResponse.acknowledged());  
    } else {  
        log.info("index: {} not found!", indexName);  
    }  
}
```

### 更新Mappings

```java
public static void updateMappings(ElasticsearchClient esClient, String indexName, Map<String, Property> documentMap) throws IOException {  
    PutMappingRequest putMappingRequest = PutMappingRequest.of(m -> m.index(indexName).properties(documentMap));  
    PutMappingResponse putMappingResponse = esClient.indices().putMapping(putMappingRequest);  
    boolean acknowledged = putMappingResponse.acknowledged();  
    log.info("update mappings ack: {}", acknowledged);  
}
```

### 使用更新Mappings方法

```java
@Test  
public void updateMappingsTest() throws IOException {  
    Map<String, Property> documentMap = new HashMap<>();  
    documentMap.put("name", Property.of(p -> p.text(TextProperty.of(t -> t.index(true)))));  
    documentMap.put("location", Property.of(p -> p.geoPoint(GeoPointProperty.of(g -> g.ignoreZValue(true)))));  
    // index 设置为 true，才可以使用 search range 功能  
    documentMap.put("age", Property.of(p -> p.integer(IntegerNumberProperty.of(i -> i.index(true)))));  
    EsUtils.updateMappings(esClient, indexName, documentMap);  
}
```

## 文档操作

### 实体类

```java
@Data  
public class Product {  
    private String id;  
    private String name;  
    private String location;  
    private Integer age;  
    private String polygon;
}
```

### 新增

```java
public static void addOneDocument(ElasticsearchClient esClient, String indexName, Product product) throws IOException {  
    IndexResponse indexResponse = esClient.index(i -> i.index(indexName).id(product.getId()).document(product));  
    log.info("add one document result: {}", indexResponse.result().jsonValue());  
}
```

### 批量新增

```java
public static void batchAddDocument(ElasticsearchClient esClient, String indexName, List<Product> products) throws IOException {  
    List<BulkOperation> bulkOperations = new ArrayList<>();  
    products.forEach(p -> bulkOperations.add(BulkOperation.of(b -> b.index(c -> c.id(p.getId()).document(p)))));  
    BulkResponse bulkResponse = esClient.bulk(s -> s.index(indexName).operations(bulkOperations));  
    bulkResponse.items().forEach(b -> log.info("bulk response result = {}", b.result()));  
    log.error("bulk response.error() = {}", bulkResponse.errors());  
}
```

### 查询

```java
public static void getDocument(ElasticsearchClient esClient, String indexName, String id) throws IOException {  
    GetResponse<Product> getResponse = esClient.get(s -> s.index(indexName).id(id), Product.class);  
    if (getResponse.found()) {  
        Product source = getResponse.source();  
        log.info("get response: {}", source);  
    }  
    // 判断文档是否存在  
    BooleanResponse booleanResponse = esClient.exists(s -> s.index(indexName).id(id));  
    log.info("文档id:{},是否存在：{}", id, booleanResponse.value());  
}
```

### 更新

```java
public static void updateDocument(ElasticsearchClient esClient, String indexName, Product product) throws IOException {  
    UpdateResponse<Product> updateResponse = esClient.update(s -> s.index(indexName).id(product.getId()).doc(product), Product.class);  
    log.info("update doc result: {}", updateResponse.result());  
}
```

### 删除

```java
public static void deleteDocument(ElasticsearchClient esClient, String indexName, String id) {  
    try {  
        DeleteResponse deleteResponse = esClient.delete(s -> s.index(indexName).id(id));  
        log.info("del doc result: {}", deleteResponse.result());  
    } catch (IOException e) {  
        log.error("del doc failed, error: ", e);  
    }  
}
```

### 批量删除

```java
public static void batchDeleteDocument(ElasticsearchClient esClient, String indexName, List<String> ids) {  
    List<BulkOperation> bulkOperations = new ArrayList<>();  
    ids.forEach(a -> bulkOperations.add(BulkOperation.of(b -> b.delete(c -> c.id(a)))));  
    try {  
        BulkResponse bulkResponse = esClient.bulk(a -> a.index(indexName).operations(bulkOperations));  
        bulkResponse.items().forEach(a -> log.info("batch del result: {}", a.result()));  
        log.error("batch del bulk resp errors: {}", bulkResponse.errors());  
    } catch (IOException e) {  
        log.error("batch del doc failed, error: ", e);  
    }  
}
```

## 搜索

### 单个搜索

```java
public static void searchOne(ElasticsearchClient esClient, String indexName, String searchText) throws IOException {  
    SearchResponse<Product> searchResponse = esClient.search(s -> s  
            .index(indexName)  
            // 搜索请求的查询部分（搜索请求也可以有其他组件，如聚合）  
            .query(q -> q  
                    // 在众多可用的查询变体中选择一个。我们在这里选择匹配查询（全文搜索）  
                    .match(t -> t  
                            .field("name")  
                            .query(searchText))), Product.class);  
    TotalHits total = searchResponse.hits().total();  
    boolean isExactResult = total != null && total.relation() == TotalHitsRelation.Eq;  
    if (isExactResult) {  
        log.info("search has: {} results", total.value());  
    } else {  
        log.info("search more than : {} results", total.value());  
    }  
    List<Hit<Product>> hits = searchResponse.hits().hits();  
    for (Hit<Product> hit : hits) {  
        Product source = hit.source();  
        log.info("Found result: {}", source);  
    }  
}
```

### 分页搜索

```java
public static void searchPage(ElasticsearchClient esClient, String indexName, String searchText) throws IOException {  
    Query query = RangeQuery.of(r -> r  
                    .field("age")  
                    .gte(JsonData.of(8)))  
            ._toQuery();  
  
    SearchResponse<Product> searchResponse = esClient.search(s -> s  
                    .index(indexName)  
                    .query(q -> q  
                            .bool(b -> b.must(query)))  
                    // 分页查询，从第0页开始查询四个doc  
                    .from(0)  
                    .size(4)  
                    // 按id降序排列  
                    .sort(f -> f  
                            .field(o -> o  
                                    .field("age").order(SortOrder.Desc))),  
            Product.class);  
    List<Hit<Product>> hits = searchResponse.hits().hits();  
    for (Hit<Product> hit : hits) {  
        Product product = hit.source();  
        log.info("search page result: {}", product);  
    }  
}
```

## 参考

1. [ElasticSearch官方文档](https://www.elastic.co/guide/index.html)
2. [Elasticsearch Java API Client](https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/7.17/index.html)
3. [俊逸的博客(ElasticSearchx系列)](https://lijunyi.xyz/docs/middleware/elasticSearch/abstract.html)
4. [Elasticsearch Java API 客户端(中文文档)](https://www.yuque.com/yxdm/dirv6a)
