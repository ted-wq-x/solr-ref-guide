# 通用参数

下表列出了用于控制 faceting 的通用参数。

|参数      |描述                  |
|---------|---------------------|
|[facet](#facet)|若为 true，启用 faceting。|
|[facet.query](#facet-query)|此参数允许您在Lucene默认语法中指定任意查询以生成构面计数。|

这些参数由以下小节描述：

## <a name="facet"><a>参数 `facet`
 
若设置为 `true`，该参数启用查询响应中的 facet 计数。
若设置为 `false` 或空白或缺失值，该参数会禁用 faceting。
除非该参数的值为 `true`，否则以下列出的所有参数都没有效果。
其默认值为空白。

## <a name="facet-query"><a>参数 `facet.query`

该参数让你可以指定一个 Lucene 默认语法格式的任意的查询来生成一个 facet 计数。
默认， Solr 的 faceting 特性会自动决定某个字段的唯一项并对其中的每个词项返回一个数值。
使用 `facet.query`，你可以覆盖这种默认行为并准确地选择你希望被计数的哪个词项或表达式。
在一个典型的 faceting 实现中，你将指定多个 `facet.query` 参数。
这个参数对基于数值范围或基于前缀的 facet 非常有用。

你可以多次设置 `facet.query` 参数以指示有多个查询应被用于独立的 facet 约束。

要以默认语法以外的语法来使用 facet 查询，需要给该 facet 查询前缀查询记法的名称。
如使用假想的 `myfunc` 查询解析器，你可以设置 `facet.query` 参数如下：

```
facet.query={!myfunc}name~fred
```

## <a name="Field-Value Faceting Parameters"></a>Field-Value Faceting参数

有些参数能被用于触发基于字段中的索引词的facet。

使用这些参数时，请务必记住，“term”是Lucene中非常具体的概念：
在任何分析发生后被索引的文字字段/值对。对于包含词干，小写或单词分割的文本字段，生成的术语可能不是您期望的。

如果你希望solr对所有的文本字符串时都进行分析（搜索时）和faceting，请在schema中使用copyField创建两个版本的字段，Text和String。
确保两个的属性indexed="true"。

faceting相关参数

|Parameter|Description|
|---------|-----------|
|[facet.field](#facet.field)|表示一个字段作为facet|
|[facet.prefix](#facet.prefix)|词语带有这个前缀的才能使用facet|
|[facet.contains](#facet.contains)|内容带有指定字符串的才能使用facet|
|[facet.contains.ignoreCase](#facet.contains.ignoreCase)|字面意思|
|[facet.sort](#facet.sort)|facet的结果是否排序|
|[facet.limit](#facet.limit)|控制facet的返回结构数量|
|[facet.offset](#facet.offset)|facet分页的偏移量|
|[facet.mincount](#facet.mincount)|facet返回结果中facet字段的最小计数|
|[facet.missing](#facet.missing)| Solr是否统计所有匹配结果中不含有facet字段值所有匹配结果数量，除了facet字段基于术语的约束之外。
|[facet.method](#facet.method)|facet字段的算法或方法|
|[facet.exists](#facet.exists)| Caps facet counts,仅适用于facet.method = enum作为性能优化|
|[facet.excludeTerms](#facet.excludeTerms)|移除指定的词语从facet统计中。|
|[facet.enum.cache.minDf](#facet.enum.cache.minDf) |指定文档的最小频率，当确定词语的数量时filterCache应该使用|
|[facet.overrequest.count](#facet.overrequest.count)|文档的数量，超过有效的facet.limit将请求分布式搜索中的分片|
|[facet.overrequest.ratio](#facet.overrequest.ratio)|在分布式搜索中每个分片请求的有效的facet.limit|
|[facet.threads](#facet.threads)|并发进行facet|
