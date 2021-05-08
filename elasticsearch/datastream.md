## 1.	Data stream 的概念
### 1.1 时序性数据
时间序列数据（time series data）是在不同时间上收集到的数据，用于所描述现象随时间变化的情况。这类数据反映了某一事物、现象等随时间的变化状态或程度。  
总的来说，这类数据主要基于时间特性明显，随着时间的流逝，往往过去时间的数据没有现在时间的重要或者敏感。  
对于 ES 处理时序性数据，有人总结了主要有以下特点：  
* 由时间戳 + 数据组成。基于时间的事件，可以是服务器日志或者社交媒体流。
* 通常搜索最近事件，旧文件变得不太重要
* 索引的使用主要基于时间，但是数据并不一定随着时间均衡分布。
* 时序性数据一旦存入后很少修改。
* 时序性数据随着时间的增加，数据量会很大。  

Elastisearch 在时序性数据的使用中，往往会有以下的缺点：  
* 索引随着时间增加而数目较多。
* 索引大小无法均衡。
* 管理索引成本较高，需要维护 merge 合并删除等一系列任务。
* 节点资源与冷热数据分布不匹配。  

在这样的一个场景下，Data stream -数据流的概念应运而生。  
Data stream 是 Elastic Stack 7.9 的一个新的功能。Data stream 可以跨多个索引存储只追加时序性数据，同时为查询写入等请求提供唯一的一个命名资源。 Data stream 非常适合日志，事件，指标以及其他持续生成的数据。  
简单来说，Data stream 根据模板生成存储数据的后备索引，然后自动将搜索或者索引请求路由到存储流数据的后备索引。 而这些后备索引则根据索引生命周期管理（ ILM ）来自动管理。 例如，你可以使用 ILM 自动将较旧的后备索引移动到较便宜的硬件上（冷热数据处理），根据索引大小自动 rollover 出新的后备索引，或者删除到时间限制的索引。  
在一定程度上，Data stream 的管理优势是利用了 ILM 的特性。但是 ILM 在普通场景下需要根据索引的别名（ alias ）逐个设置，而 Data stream 则是抛弃了 alias 的限制，可以直接批量化设置相似名称的索引，大大增加了 ILM 的使用范围。  
### 1.2 Data stream 的组成
数据流在 Elasticsearch 集群中由一个或多个隐藏的、自动生成的后备索引组成。  
![Data stream 01](https://github.com/kin122/duoankin.github.io/blob/main/elasticsearch/images/%E6%95%B0%E6%8D%AE%E6%B5%81-01.png)  
在实际的 ES 操作中，数据流依靠索引模板来设定数据流实体的后备索引。  
* 模板包含用于配置流的后备索引的映射和设置。
* 同一个索引模板可用于多个数据流。
* 不能删除数据流正在使用的索引模板。  

每个索引到数据流的文档必须包含一个 @timestamp 字段，映射为 date 或 date_nanos 字段类型。如果索引模板没有为 @timestamp 字段指定映射， Elasticsearch 将 @timestamp 映射为带有默认选项的日期字段。  
Data stream 的读请求主要如下图，数据流自动将请求路由到其所有后备索引。  
![Data stream 02](https://github.com/kin122/duoankin.github.io/blob/main/elasticsearch/images/%E6%95%B0%E6%8D%AE%E6%B5%81-02.png)
而对于写请求，数据流则将该请求自动转发给最新的后备索引。  
![Data stream 03](https://github.com/kin122/duoankin.github.io/blob/main/elasticsearch/images/%E6%95%B0%E6%8D%AE%E6%B5%81-03.png)
对于写请求，有两点需要注意：  
* 不能将新文档添加到其他非最新后备索引，即使直接将请求发送到这些索引也不行。
* 不能对正在写入的索引做Clone/Close/Delete/Freeze/Shrink/Split相关操作。注：7.12版本可以close。 

### 1.3 Data stream 的特性
#### 1.3.1 生成
每个data stream的后备索引都有一个generation数，一个六位数，零填充的整数，从 000001 开始，用作该流的 rollover 的计数。  
后备索引名主要依照以下格式：  
`.ds-<data-stream>-<generation>`
Generation越大，后备索引包含的数据越新。 例如，web-server-logs 数据流最新的 generation 为 34。该流的最新后备索引名为 .ds-web-server-logs-000034。  
注意：某些操作（例如 shrink 或 restore）可以更改后备索引的名称。 这些名称更改不会从其数据流中删除后备索引。  

