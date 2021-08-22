### 使用方式
multi-field可以将同一个字段应用于不同的分词解析，以此来实现不同的查询用途。比如，ES默认对string字段（不属于date和numeric类型的）设置成text 和一个keyword子字段。
### 多字段的使用场景
multi-field主要通过mapping中的fields参数实现。  
主要实现方法如下：
```
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "city": {
        "type": "text",
        "fields": {
          "raw": { 
            "type":  "keyword"
          }
        }
      }
    }
  }
}

PUT my-index-000001/_doc/1
{
  "city": "New York"
}

PUT my-index-000001/_doc/2
{
  "city": "York"
}

GET my-index-000001/_search
{
  "query": {
    "match": {
      "city": "york" 
    }
  },
  "sort": {
    "city.raw": "asc" 
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" 
      }
    }
  }
}
```
返回结果如下：  
```
```
#### keyword与text
在实际ES生产使用中，同一个字段同时具有text和keyword字段类型是比较有效的，text满足全文搜索的需求，而keyword用于聚合和排序。

#### 多分词器
当然在特殊的生产场景下，可能会有同一个字段需要分词器甚至多个字段类型的使用需求，这个fields也可以可以满足的，只是要评估好使用成本，multi-field会造成存储成本相应的增加。
