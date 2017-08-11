# 通用查询参数

下表汇总了 Solr 通用的查询参数，它们由 [standard](./standard.md)、
[DisMax](./dismax.md) 和 [eDisMax](./extended_dismax.md) 请求处理器所支持。

|参数      |描述                       |
|---------|--------------------------|
|[defType](#defType) |要使用的查询解析器|
|[sort](#sort) |根据响应的分数或其他指定的特征，按升序或降序对查询结果进行排序。|
|[start](#start) |在Solr开始显示内容的响应中指定偏移量（默认为0）。即分页的偏移量一个意思|
|[rows](#rows) |返回结果的条数（默认值：10）|
|[fq](#fq) |对查询结果进行过滤|
|[fl](#fl) |指定返回结果的字段|
|[debug](#debug) |在响应中请求其他调试信息。指定`debug = timing`参数只返回时序信息; 指定`debug = results`参数为返回的每个文档返回“explain”信息; 指定`debug = query`参数返回所有调试信息。|
|[explainOther](#explainOther) |允许客户端指定Lucene查询标识文档集合。若非空，将返回查询结果的每个文档的解释信息，和主查询（使用q参数查询）相关的debug信息|
|[timeAllowed](#timeAllowed) |定义允许处理查询的时间。 如果未在该段时间之间完成查询响应，则可能只返回部分信息|
|[segmentTerminateEarly](#segmentTerminateEarly)|Indicates that, if possible, Solr should stop collecting documents from each individual (sorted) segment once it can determine that any subsequent documents in that segment will not be candidates for the rows being returned. The default is false
|
                                                 
                                                 
|[omitHeader](#omitHeader) |如果设置为true，则从返回的结果中去除标题。标题包含有关请求的信息，例如请求完成的时间。 默认值为false|
|[wt](#wt) |指定用于格式化查询响应的响应处理器|
|[logParamsList](#logParamsList) |默认情况下，Solr记录所有参数。设置此参数以限制记录哪些参数。有效条目是要记录的参数，用逗号分隔（即`logParamsList = param1，param2`）。一个空列表将不记录任何参数，所以如果需要记录所有参数，那么根本不要定义这个附加参数|
|[choParams](#choParams) |响应头中包含的请求参数，此参数控制响应头的该部分中包含的内容。值为none，all和explicit。 默认值explicit。|

以下是这些参数的详细信息

### <a name="defType"></a>defType 参数

defType 参数用来选择 Solr 应用于处理请求中的主查询参数 (`q`) 的查询解析器。如：

```
defType=dismax
``` 

若没有指定 defType，则默认使用 [标准查询解析器](./standard.md)。(即 `defType=lucene`)

### <a name="sort"></a>sort 参数

`sort` 参数会以升序(`asc`)或降序(`desc`) 来排列搜索结果。
该参数用于数值或字母序内容上。不区分大小写(即 `asc` 或 `ASC`)。

Solr 可以根据文档的分数或某个字段的单个的
indexed 或使用了 [DocValues](../../schema/docvalues.md) 的字段的值
(即任何 `schema.xml` 中属性包括 `multiValued="false"` 并且 `docValues="true"` 或 `indexed="true"` 之一为真
 - 若字段没有启用 DocValues，将使用被索引的项在运行时来构建它们)
来排序。

前提是：
* 字段是非分词的(即字段没有分析器且其内容已被解析为 tokens ，这样会使得排序不一致)，或者
* 字段使用了一个的分析器(如 KeywordTokenizer)

如果你想能够在字段上排序又希望将其分词，请在 schema.xml 中使用 `<copyField>` 指令来克隆字段。
然后在该字段上搜索而在其克隆字段上排序。

下表解释了 Solr 如何在多种设置下如何响应 `sort` 参数

|示例        |结果                        |
|-----------|---------------------------|
|           |If the sort parameter is omitted, sorting is performed as though the parameter were set to score desc .|
|score desc |Sorts in descending order from the highest score to the lowest score.|
|price asc  |Sorts in ascending order of the price field|
|inStock desc, price asc|Sorts by the contents of the inStock field in descending order, then within those results sorts in ascending order by the contents of the price field.|

根据 sort 参数的参数：

* 一个 sort 排序必须包含一个字段名称(或 `score` 作为伪字段)，
后跟一个空格(URL 字符串中转码为 `+` 或 `%20`)，
再后面跟一个排序方向(`asc` 或 `desc`)。
* 多个 sort 排序间可用逗号分割，使用语法 `sort=<field name>+<direction>,<field name>+<direction>],...`
    * 当提供了超过一个排序条件时，第二个项仅会在第一个项相同时起作用。第三项会在前两项都相同时起作用。等等。

### <a name="start"></a>start 参数

当指定时， `start` 参数指定了查询结果集的偏移量并指示 Solr 从该偏移量开始显示结果。

默认值为 '0'。或者说默认 Solr 返回的结果没有偏移量，就从结果集本身的起始处开始显示。

设置 `start` 参数为其它值，如 3,会导致 Solr 跳过前面的记录并从该偏移量标识的文档处开始。

你可以将 `start` 参数用作分页。例如， 当 `row` 参数设置为 10 时，你可以显示三个连续的页面结果
通过将 `start` 设置为 0 发送一个查询, 然后设置为 10 再发送相同的查询，在设置为 20 再发送查询。

### <a name="rows"></a>rows 参数

你可以使用 `row` 参数对查询结果分页。
该参数指定了 Solr 应该在一次请求中返回给客户端的完整数据集中最大的文档数量。

其默认值为 10。即默认 Solr 每次查询的响应中返回 10 个文档。

### <a name="fq"></a>fq(Filter Query) 参数

`fq` 参数定义了可用于限制可被返回的文档的超集的查询，而不会影响分数。
它对于加速复杂的查询时很有用，_因为以 `fq` 指定的查询会在主查询缓存之外被独立地进行缓存_。
当后续的查询使用相同的过滤器时，缓存命中，这样过滤的结果可以从缓存中很快返回。

当使用 `fq` 参数时，记住以下几点：

* `fq` 参数可以在一个查询中多次指定。只有每个过滤参数实例的文档集的交集中的文档才会被返回。
下例中，仅有流行度大于10 且 section 为 0 的才会匹配。

```
fq=popularity:[10 TO *]&fq=section:0
```

* 过滤器查询可以涉及复杂的布尔查询。上例也可以写做单个的具有两个强制子句的查询，如：

```
fq=+popularity:[10 TO *] +section:0
```

* 每个过滤器查询的文档集都被独立地缓存。
因此考虑上面的例子：如果两个条件常常一起出现时使用单个具有两个强制子句的 `fq` 查询，
而如果两个条件相对独立时使用两个独立的 `fq` 参数。
(学习调整缓存大小并确保过滤器缓存确实存在，可查看 [配置Solr实例](../../config/readme.md) )
* 对所有的参数，URL 中的特殊字符需要进行合适的转码并编码为Hex 值。
有很多在线工具可帮助你完成 URL 编码。如 http://meyerweb.com/eric/tools/dencoder/ 。

### <a name="fl"></a>fl(Field List) 参数

`fl` 参数指定查询结果的返回字段。字段的属性必须有`stored="true"`或`docValues="true"`之一。

字段列表可以以空格分隔或逗号分隔的字段名称的列表。
字符串 `score` 可用查询返回中包含分数字段。
通配符 `*` 选择了文档中所有 stored 的字段。
你也可以给字段列表请求添加伪字段、函数和转换器。

下表展示了一些使用 `fl` 的基本示例：

|字段列表        |结果                    |
|--------------|------------------------|
|id name price |Return only the id, name, and price fields. |
|id,name,price |Return only the id, name, and price fields. |
|id name,price |Return only the id, name, and price fields. |
|id score      |Return the id field and the score.          |
|*             |Return all the fields in each document. This is the default value of the fl parameter. |
|* score       |Return all the fields in each document, along with each field's score. |

#### 函数值

[函数](./function.md) 可针对结果中的每个文档进行计算，并作为伪字段返回：

```
fl=id,title,product(price,popularity)
```

#### 文档转换器

[文档转换器](searching/transform.md) 可用于修改查询结果中每个文档返回的信息：

```
fl=id,title,[explain]
```

#### 字段别名

你可以通过一个前缀 `<displayName>:` 来修改用在相应中某字段、函数或转换器的键的名称。如：

```
fl=id,sales_price:price,secret_sauce:prod(price,popularity),why_score:[explainstyle=nl]
```

响应：

```json
{
  "response": {
    "numFound": 2,
    "start": 0,
    "docs": [
      {
        "id": "6H500F0",
        "secret_sauce": 2100.0,
        "sales_price": 350.0,
        "why_score": {
          "match": true,
          "value": 1.052226,
          "description": "weight(features:cache in 2) [DefaultSimilarity], result of:",
          "details": [
            {
              "...":"gg"
            }
          ]
        }
      }
    ]
  }
}
```

### <a name="debug"></a>debug 参数

`debug` 参数可被指定多次且支持以下参数：

* `debug=query`： 仅返回关于查询的调试信息
* `debug=timing`： 返回关于查询耗时的调试信息
* `debug=reaults`： 返回分数结果(也称 "explain") 的调试信息
* `debug=all`：返回关于请求的所有可用的调试信息。(或者使用: `debug=true`)

为后向兼容， `debugQuery=true` 可作为 `debug=all` 的一种替代方式来指定

默认行为是不包含调试信息。

### <a name="explainOther"></a>explainOther 参数

`explainOther` 参数指定了一个 Lucene 查询来标识一个文档集。
若该参数被包含并设置为非空白值，该查询将返回调试信息，以及每个文档匹配该 Lucene 查询的 “解释信息”，
相对于主查询(它由 `q` 参数指定)。如：

```
q=supervillians&debugQuery=on&explainOther=id:juggernaut
```

上面的查询让你可以检查顶部的匹配文档的分数解释，
将其和对匹配 `id:juggernaut` 的文档做比较，并决定为何该排序不是你想要的。

该参数的默认值为空，这样不会返回任何额外的 “解释信息”。

### <a name="timeAllowed"></a>timeAllowed 参数

定义允许处理查询的时间。 如果未在该段时间之间完成查询响应，则可能只返回部分信息。单位是毫秒。

### <a name="omitHeader"></a>omitHeader 参数

该参数应被设为 true 或 false。

若为 true，该参数从返回结果中排除头部。
头部包含了关于请求的信息，如花费了多长时间完成。
该参数的默认值为 false。

### <a name="wt"></a>wt 参数

`wt` 参数选择了 Solr 应该用作查询响应的 Response Writer。
更多信息，查看 [ResponseWriters](../response_writers.md)

### `cache=false` 参数

Solr 默认会缓存所有查询以及过滤器查询的结果。
要禁用缓存，可设置 `cache=false` 参数。

你也可以使用 `cost` 选项来控制非缓存的过滤器查询被求值的顺序。
这让你可以将便宜的非缓存过滤器放在昂贵的非缓存过滤器之前。

对非常高成本的过滤器，若 `cache=false` 和 `cost>=100` 且该查询实现了 `PostFilter` 接口，
一个 Collector 将会被查询所请求并用于在它们匹配主查询及其它过滤器查询之后进行文档的过滤。
存在多种后处理过滤器；且它们也可以根据成本排序。

例如：

```
// normal function range query used as a filter, all matching documents
// generated up front and cached
fq={!frange l=10 u=100}mul(popularity,price)

// function range query run in parallel with the main query like a traditional
// lucene filter
fq={!frange l=10 u=100 cache=false}mul(popularity,price)

// function range query checked after each document that already matches the query
// and all other filters. Good for really expensive function queries.
fq={!frange l=10 u=100 cache=false cost=100}mul(popularity,price)
```

### <a name="logParamsList"></a>logParamsList 参数

默认 Solr 给所有请求的参数记录日志。从 4.7 开始，设置这个属性可用于限制请求的哪些参数被记入日志。
这可以帮助控制仅被认为对你的组织有意义的参数。

如，你可以定义：

```
logParamsList=q,fq
```

这样就只记录 `q` 和 `fq` 参数。

若不想记录任何字段，你可以设置 `logParamsList` 为空 (即， `logParamsList=`)。

> 该参数不仅适用于查询请求，也适用于任何对 Solr 的请求。

### <a name="echoParams"></a>echoParams 参数

该参数控制着返回头中包含那些请求参数。

下面的表解释每个选项的意思

|Value|Meaning
|------|---------|
|explicit|这是默认值。只有包含在实际请求中的参数加上_参数（这是64位数字时间戳）才会被添加到响应头的params部分。
|all|所有参数|
|none|没有任何参数|


