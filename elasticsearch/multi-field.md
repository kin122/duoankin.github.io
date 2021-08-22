### 使用方式
multi-field 可以将同一个字段应用于不同的分词解析，以此来实现不同的查询用途。比如，ES 默认对 string 字段（不属于 date 和   numeric类型的）设置成text 和一个 keyword 子字段。
### 多字段的使用场景
multi-field 主要通过 mapping 中的 fields 参数实现。  
主要实现方法如下：
```
PUT test-000001
{
  "mappings": {
    "properties": {
      "city": {
        "type": "text",
        //使用fields参数来定义一个子字段,名为keyword
        "fields": {
          "keyword": { 
            "type":  "keyword"
          }
        }
      }
    }
  }
}
```
#### keyword 与 text
在实际 ES 生产使用中，同一个字段同时具有 text 和 keyword 字段类型是比较有效的，text 满足全文搜索的需求，而 keyword 用于聚合和排序。
以上面定义的 city 字段作为案例，向 test-000001 索引添加数据
```
PUT test-000001/_doc/1
{
  "city": "New York"
}

PUT test-000001/_doc/2
{
  "city": "York"
}
```
利用 text 字段类型进行全文检索，利用 keyword 字段类型进行聚合
```
GET test-000001/_search
{
  "query": {
    "match": {
      "city": "york" 
    }
  },
  // 利用keyword字段进行排序
  "sort": {
    "city.keyword": "asc" 
  },
  // 利用keyword字段进行聚合
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.keyword" 
      }
    }
  }
}
```
返回结果如下：
```
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
    // 根据全文搜索 york 并按照 asc 排序返回两个结果
      {
        "_index" : "test-000001",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : null,
        "_source" : {
          "city" : "New York"
        },
        "sort" : [
          "New York"
        ]
      },
      {
        "_index" : "test-000001",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : null,
        "_source" : {
          "city" : "York"
        },
        "sort" : [
          "York"
        ]
      }
    ]
  },
  "aggregations" : {
  // 根据 city.keyword 聚合的结果
    "Cities" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "New York",
          "doc_count" : 1
        },
        {
          "key" : "York",
          "doc_count" : 1
        }
      ]
    }
  }
}

```
#### 多分词器
当然在特殊的生产场景下，可能会有同一个字段需要分词器甚至多个字段类型的使用需求，这个 fields 也可以可以满足的。
比如：title 字段实现 standard 和 english 两种分词器且有 keyword 字段类型。  
```
PUT test-000002
{
  "mappings": {
    "properties": {
      "title": {
        "type": "keyword",
        "fields": {
        // 实现 standard 和 english 两种不同分词的子字段
          "standard": {
            "type": "text",
            "analyzer": "standard"
          },
          "english": {
            "type": "text",
            "analyzer": "english"
          }
        }
      }
    }
  }
}
```
由于不同的分词器产生的词根并不完全一致，面对不同的搜索场景，这种方式可以很好的优化搜索体验。
比如，下面这种情况
```
PUT test-000002/_doc/1
{ "title": "quick brown fox" } 

PUT test-000002/_doc/2
{ "title": "quick brown foxes" } 

```
通过 \_analyze API 查看一下这两个文档的具体解析。  
standard 分词器。 
```
POST _analyze
{
  "analyzer": "standard",
  "text": "quick brown foxes"
}
```
结果：
```
{
  "tokens" : [
    {
      "token" : "quick",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "brown",
      "start_offset" : 6,
      "end_offset" : 11,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "foxes",
      "start_offset" : 12,
      "end_offset" : 17,
      "type" : "<ALPHANUM>",
      "position" : 2
    }
  ]
}
```
english 分词器
```
POST _analyze
{
  "analyzer": "english",
  "text": "quick brown foxes"
}
```
结果
```
{
  "tokens" : [
    {
      "token" : "quick",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "brown",
      "start_offset" : 6,
      "end_offset" : 11,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "fox",
      "start_offset" : 12,
      "end_offset" : 17,
      "type" : "<ALPHANUM>",
      "position" : 2
    }
  ]
}
```
在 english 分词中会将 foxes 解析成 fox，如果使用 quick brown foxes 查询 title.standard，只能返回一个结果，而查询 title.english 则能返回两个结果。  
所以为了面对不同分词器导致的不同的分词解析结果，可以一次查询多个分词子字段。  
```
GET test-000002/_search
{
  "query": {
    "multi_match": {
      "query": "quick brown foxes",
      "fields": [ 
        "title.standard",
        "title.english"
      ],
      "type": "most_fields" 
    }
  }
}
```

当然这种情况下需要评估好使用成本，multi-field会造成存储成本相应的增加。
