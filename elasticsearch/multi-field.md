### 使用方式
multi-field可以将同一个字段应用于不同的分词解析，以此来实现不同的查询用途。比如，ES默认对string字段（不属于date和numeric类型的）设置成text 和一个keyword子字段。
### 多字段的使用场景
multi-field主要通过mapping中的fields参数实现。  

#### keyword与text
#### 多分词器
可以N个字段
