# Elasticsearch 使用规范

> 适用于所有使用 Elasticsearch 的后端模块，与 backend-monolith.md / backend-microservice.md 配合使用


## 一、索引设计规范

### 1. 索引命名

- 格式：`{业务}_{数据类型}`，小写 + 下划线，如 `product_info`、`order_log`、`article_content`
- 禁止大写、中文、特殊字符
- 同类数据需要重建索引时用版本号后缀：`product_info_v2`，通过别名切换

### 2. Mapping 定义原则

**创建索引时必须显式定义 mapping**（禁止依赖 ES 自动推断，自动推断的类型往往不对）

**字段类型选择决策表**：

| 数据特征 | ES 类型 | 示例 |
|---------|---------|------|
| 精确匹配（ID、状态码、枚举、手机号） | **keyword** | orderId、status、category |
| 全文检索（标题、描述、内容） | **text**（+ 分词器） | title、content、description |
| 数值范围查询/排序/聚合 | **integer / long / double** | price、quantity、score |
| 日期范围查询 | **date** | createTime、updateTime |
| 嵌套对象（需独立查询） | **nested** | tags、specs（数组内对象需独立条件查询时） |
| 嵌套对象（不需独立查询） | **object** | address（整体查询即可） |
| 地理位置 | **geo_point** | 经纬度 |

### 3. 分词器选择

| 分词器 | 适用场景 | 分词粒度 |
|--------|---------|---------|
| **ik_max_word** | **索引时**（写入时分词） | 最细粒度，提高召回率 |
| **ik_smart** | **搜索时**（查询时分词） | 最粗粒度，提高精确率 |

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "ik_index_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word"
        },
        "ik_search_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_index_analyzer",
        "search_analyzer": "ik_search_analyzer",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 128 }
        }
      },
      "status": { "type": "integer" },
      "price": { "type": "double" },
      "createTime": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss" }
    }
  }
}
```

### 4. 文档 ID 设计

- 有业务唯一 ID：用业务 ID 作为 `_id`（避免重复写入，便于更新）
- 无业务唯一 ID：让 ES 自动生成

### 5. 字段设计原则

- **text 字段必须加 keyword 子字段**（排序、聚合时用 keyword，text 不支持）
- 字段名用驼峰，与 Java 实体保持一致
- 不需要检索/排序/聚合的字段：`"index": false`（节省空间）
- 不需要返回的大字段：`"store": false`（默认不存储原文，_source 中有）


## 二、查询规范

### 1. 查询类型选择

| 场景 | 查询方式 | 说明 |
|------|---------|------|
| 全文搜索 | **match / match_phrase** | 对 text 字段分词匹配 |
| 精确匹配 | **term** | 对 keyword 字段精确匹配 |
| 组合条件 | **bool**（must/should/filter/must_not） | 多条件组合 |
| 范围查询 | **range** | 价格区间、时间范围 |
| 前缀搜索 | **match_phrase_prefix** | 搜索联想/自动补全 |
| 聚合统计 | **aggregation** | 分组统计、Top N |

### 2. bool 查询模板

```java
NativeSearchQuery query = new NativeSearchQueryBuilder()
    .withQuery(QueryBuilders.boolQuery()
        // 影响评分的条件（需要相关性排序时用）
        .must(QueryBuilders.matchQuery("title", keyword))
        // 不影响评分的过滤条件（推荐用 filter，可缓存，性能更好）
        .filter(QueryBuilders.termQuery("status", 1))
        .filter(QueryBuilders.rangeQuery("price").gte(10).lte(100))
        .filter(QueryBuilders.rangeQuery("createTime").gte(startDate))
    )
    .withPageable(PageRequest.of(pageNum - 1, pageSize))
    .withSort(SortBuilders.fieldSort("createTime").order(SortOrder.DESC))
    .build();

SearchHits<Product> hits = elasticsearchRestTemplate.search(query, Product.class);
```

### 3. filter vs must 区别

| 子句 | 评分 | 缓存 | 性能 | 适用 |
|------|------|------|------|------|
| **filter** | 不计算 | 可缓存 | **高** | 过滤条件（状态、时间范围、精确匹配）→ **默认用 filter** |
| **must** | 计算 | 不缓存 | 中 | 需要影响相关性排序的条件（关键词搜索） |

### 4. 查询防坑

```java
// 错误：term 查 text 字段（text 字段被分词了，term 匹配不到）
QueryBuilders.termQuery("title", "苹果手机")

// 正确：match 查 text 字段
QueryBuilders.matchQuery("title", "苹果手机")

// 正确：term 查 keyword 字段
QueryBuilders.termQuery("status", 1)
```


## 三、分页策略

| 方案 | 适用场景 | 深度分页 | 性能 |
|------|---------|---------|------|
| **from + size** | 浅分页（前 100 页） | 受限（默认 max_result_window=10000） | 中 |
| **search_after** | 深度分页、无限滚动 | 无限制 | **高** |
| scroll | 批量导出（已不推荐实时查询） | 无限制 | 低 |

**常规分页**（pageNum * pageSize < 10000）：

```java
.withPageable(PageRequest.of(pageNum - 1, pageSize))
```

**深度分页**（超过 10000 条，用 search_after）：

```java
NativeSearchQuery query = new NativeSearchQueryBuilder()
    .withQuery(...)
    .withPageable(PageRequest.of(0, pageSize))  // search_after 时 page 从 0 开始
    .withSort(SortBuilders.fieldSort("createTime").order(SortOrder.DESC))
    .withSort(SortBuilders.fieldSort("_id"))  // tiebreaker 防止排序值相同
    .build();

// 第二页起：用上一页最后一条的 sort value
query.setSearchAfter(lastSortValues);

// 结果中获取 sort value 用于下次翻页
List<SortValues> sortValues = hits.getSearchHits()
    .get(hits.getSearchHits().size() - 1).getSortValues();
```


## 四、高亮搜索

```java
NativeSearchQuery query = new NativeSearchQueryBuilder()
    .withQuery(QueryBuilders.matchQuery("title", keyword))
    .withHighlightBuilder(new HighlightBuilder()
        .field("title")
        .preTags("<em class=\"highlight\">")
        .postTags("</em>")
        .fragmentSize(200)       // 片段长度
        .numOfFragments(3)       // 最多返回 3 个片段
    )
    .build();
```


## 五、索引别名管理

**所有查询必须通过别名访问，不直接用物理索引名**（便于零停机重建索引）

```java
// 1. 创建新索引
CreateIndexRequest request = new CreateIndexRequest("product_info_v2");
request.mapping(mappingJson);
client.indices().create(request, RequestOptions.DEFAULT);

// 2. 同步数据（reindex）
ReindexRequest reindex = new ReindexRequest()
    .setSourceIndices("product_info_v1")
    .setDestIndex("product_info_v2");
client.reindex(reindex, RequestOptions.DEFAULT);

// 3. 原子切换别名
IndicesAliasesRequest aliasRequest = new IndicesAliasesRequest();
aliasRequest.addAliasAction(
    new IndicesAliasesRequest.AliasActions(IndicesAliasesRequest.AliasActions.Type.REMOVE)
        .index("product_info_v1").alias("product_info")
);
aliasRequest.addAliasAction(
    new IndicesAliasesRequest.AliasActions(IndicesAliasesRequest.AliasActions.Type.ADD)
        .index("product_info_v2").alias("product_info")
);
client.indices().updateAliases(aliasRequest, RequestOptions.DEFAULT);

// 4. 确认后删除旧索引
client.indices().delete(new DeleteIndexRequest("product_info_v1"), RequestOptions.DEFAULT);
```


## 六、批量操作

```java
// bulk 批量写入（每次 1000~5000 条，不要一次性写太多）
BulkRequest bulkRequest = new BulkRequest();
for (Product product : productList) {
    IndexRequest request = new IndexRequest("product_info")
        .id(product.getId().toString())
        .source(JSONUtil.toJson(product), XContentType.JSON);
    bulkRequest.add(request);
}
bulkRequest.timeout(TimeValue.timeValueSeconds(30));

BulkResponse response = client.bulk(bulkRequest, RequestOptions.DEFAULT);
if (response.hasFailures()) {
    log.error("批量写入部分失败: {}", response.buildFailureMessage());
}
```


## 七、MySQL + ES 数据同步方案

| 方案 | 一致性 | 复杂度 | 改业务代码 | 适用场景 |
|------|--------|--------|-----------|---------|
| **双写**（同步写 DB + ES） | 强一致 | 低 | 需要 | 数据量小，对一致性要求高 |
| **异步消息**（DB → MQ → ES） | 最终一致 | 中 | 需要 | 数据量中等，**推荐方案** |
| **Canal 监听 binlog** | 最终一致 | 高 | 不需要 | 数据量大，不想改业务代码 |
| 定时全量同步 | 弱一致 | 低 | 不需要 | 对实时性要求低的报表数据 |


## 八、Spring Data Elasticsearch 配置

```yaml
# application.yml
spring:
  elasticsearch:
    uris: http://localhost:9200
    connection-timeout: 5s
    socket-timeout: 30s
```

```java
@Configuration
public class EsConfig {
    @Bean
    public RestHighLevelClient elasticsearchClient(
            @Value("${spring.elasticsearch.uris}") String uris) {
        HttpHost host = HttpHost.create(uris);
        RestClientBuilder builder = RestClient.builder(host)
            .setRequestConfigCallback(config -> config
                .setConnectTimeout(5000)
                .setSocketTimeout(30000));
        return new RestHighLevelClient(builder);
    }
}
```


## 九、禁止事项

- **禁止一次性查询全量数据**（必须分页，单次最多返回 1000 条）
- **禁止深度翻页用 from + size**（超过 10000 条改用 search_after）
- **禁止 mapping 随意修改**（已建好的字段类型不能改，只能新增字段或重建索引）
- **禁止在 ES 中做复杂业务逻辑**（ES 是搜索引擎，不是关系型数据库）
- **禁止不设超时的查询**（必须设置 socket timeout，防止慢查询拖垮线程）
- **禁止用 wildcard 做前缀匹配**（`*xxx` 性能极差，改用 match_phrase_prefix）
- **禁止写入后立即查询**（ES 默认 1 秒 refresh 间隔，可能查不到，用 `?refresh=true` 或等 1 秒）
- **禁止依赖 ES 自动推断 mapping**（字段类型必须显式定义）
- **禁止用 text 字段做排序/聚合/term 查询**（用 keyword 子字段）

---

## 开发规则整合

### 架构设计
- 优先采用当前主流且经过生产验证的企业级方案
- 以中型公司实际落地标准设计
- 满足业务需求即可，不允许过度设计

### 编码原则
- 使用最少代码完成需求
- 优先可读性，其次是代码量
- 避免重复代码（DRY）

### 代码要求
- 所有代码必须包含中文注释
- 必须进行必要的判空处理
- 必须进行必要的异常处理

### 性能原则
- 先保证正确性
- 再保证可维护性
- 最后再考虑性能优化
