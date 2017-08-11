# DisMax 查询解析器

DisMax 查询解析器设计用于处理由用户输入的简单短语(无复杂的语法)，并根据每个字段的重要性，在多个字段间使用不同的权重搜索词语。
额外的选项可让用户能够基特定场景(和用户输入独立)规则来影响分数。

通常 DisMax 查询解析器的接口更像 Google 而不像 “标准” Solr 请求处理器的接口。
这种相似性让 DisMax 适用于很多消费型应用。
它接收简单的语法，并且很少会产生错误消息。

DisMax 查询解析器支持 Lucene 查询解析器语法的一个非常简单的子集。
像在 Lucene 中一样，引号可用于分组子句，而 `+/-` 可用于表示强制和可选子句。
所有的其他 Lucene 查询解析器特殊字符(出 AND 和 OR 之外) 都会被转码以简化用户体验。
DisMax 查询解析器负责使用包含跨字段和由用户指定的加权因子的 DisMax 查询，从用户构建良好的查询。
它也让 Solr 管理员提供了额外的加权查询、加权函数和过滤查询以人为地影响所有查询的输出。
这些选项都可以在 `solrconfig.xml` 文件中作为该处理器的默认参数指定后在 Solr 查询 URL 中覆盖。

你是否对 DisMax 名称背后的技术概念感兴趣？
DisMax 表示 Maximum Disjunction。下面是 Maximum Disjunction 或 DisMax 查询的一个定义：

> A query that generates the union of documents produced by its subqueries, and that scores each document
  with the maximum score for that document as produced by any subquery, plus a tie breaking increment for
  any additional matching subqueries.

不管你是否能记住这个解释，你只需要记住 DisMax 请求处理器主要设计为易于使用，并接受几乎任何输入，而不返回错误。

## DisMax 参数

除了通用的请求参数、高亮参数和简单的 facet 参数外， DisMax 查询解析器还支持以下参数。
像标准查询解析器一样， DisMax 查询解析器允许在 `solrconfig.xml` 中指定默认参数，
或者在请求中由查询时的值进行覆盖。

|参数    |描述                        |
|-------|---------------------------|
|[q](#q)|定义该查询的原始输入字符串。     |
|[q.alt](#q-alt)|当 `q` 参数没有被使用时，调用标准查询解析器并定义查询字符串。|
|[qf](#qf)|查询字段：指定执行查询时需查询索引中的哪些字段。若不存在，默认为 `df`。 |
|[mm](#mm)|最小“应”匹配：指定查询中必须匹配的最小子句数。若在查询语句中或 `solrconfig.xml` 中没有指定 `mm` 参数，有效的 `q.op` 的值(不管是查询中、还是 `solrconfig.xml` 中的默认值，还是来自 `schema.xml` 中的 `defaultOperator` 选项)。若 `q.op` 是高效的 `AND`，则 `mm=100%`；若 `q.op` 是 `OR`,则 `mm=1`。希望强制其遗留行为的用户需要在 `solrconfig.xml` 中给 `mm` 设置一个默认值。用户应该将它添加为其请求处理器的配置的默认值。这个参数允许在表达式中混合空格(如`" 3 < -25% 10 < -3\n", " \n-25%\n", " \n3\n "`)。 |
|[pf](#pf)|短语字段：对在 `q` 参数中所有词项相邻地出现的文档加权其分数。 |
|[ps](#ps)|短语 slop： 指定两个术语可以分开以匹配指定短语的位置数|
|[qs](#qs)|查询短语Slop：指定两个术语可以分开以匹配指定短语的位置数。 与qf参数特别配合使用。|
|[tie](#tie)|Tie Breaker：在DisMax查询中指定一个float值（应该远远小于1）作为tiebreaker使用。 默认值：0.0 |
|[bq](#bq)|加权查询：指定一个因子，当考虑一个匹配时哪个词项或短语应被 "加权" 其重要性。 |
|[bf](#bf)|加权函数： 指定被应用于加权的函数(查看函数查询相关细节) |

### <a name="q"><a>参数 `q`

`q`参数定义了构成搜索的主要“查询”。参数支持原始输入字符串，没有特殊的转义。‘+’和‘-’被视为“强制”和“禁止”修饰符。字符串使用双引号包裹视为一个短语(for example, "San Jose")。 奇数个的引号视为没有引号。

> q参数不支持通配符，例如‘*’。

### <a name="q-alt"><a>参数 `q.alt`
6
当主查询参数q没有指定或为空时,使用`q.alt`参数定义一个查询语句 (默认情况下使用标准解析器)。
当你需要一个query去匹配所有文档 (don't forget `&rows=0` for that one!) 为了得到faceting统计，此时`q.alt` 参数将派上用场。


### <a name="qf"><a>参数 `qf`

权重的意思，看下面的例子就理解了:

```
qf="fieldOne^2.3 fieldTwo fieldThree^0.4"
```

assigns fieldOne a boost of 2.3, leaves fieldTwo with the default boost (because no boost factor is
specified), and fieldThree a boost of 0.4. These boost factors make matches in fieldOne much more
significant than matches in fieldTwo , which in turn are much more significant than matches in fieldThree .

### <a name="mm"><a>参数 `mm`

When processing queries, Lucene/Solr recognizes three types of clauses: mandatory, prohibited, and "optional"
(also known as "should" clauses). By default, all words or phrases specified in the q parameter are treated as
"optional" clauses unless they are preceded by a "+" or a "-". When dealing with these "optional" clauses, the mm
parameter makes it possible to say that a certain minimum number of those clauses must match. The DisMax
query parser offers great flexibility in how the minimum number can be specified.

The table below explains the various ways that mm values can be specified.

|语法      |示例    |描述                      |
|---------|-------|--------------------------|
|正整数    |3      |Defines the minimum number of clauses that must match, regardless of how many clauses there are in total.|
|负整数    |-2     |Sets the minimum number of matching clauses to the total number of optional clauses, minus this value.|
|百分比    |75%    |Sets the minimum number of matching clauses to this percentage of the total number of optional clauses. The number computed from the percentage is rounded down and used as the minimum.|
|负百分比  |-25%   |Indicates that this percent of the total number of optional clauses can be missing. The number computed from the percentage is rounded down, before being subtracted from the total to determine the minimum number.|
|已正整数开头后跟一个 `>` 或 `<`，后跟另外一个值的表达式 |3<90% |Defines a conditional expression indicating that if the number of optional clauses is equal to (or less than) the integer, they are all required, but if it's greater than the integer, the specification applies. In this example: if there are 1 to 3 clauses they are all required, but for 4 or more clauses only 90% are required.|
|多个涉及 `>` 或 `<` 的条件表达式 |2<-25% 9<-3|Defines multiple conditions, each one being valid only for numbers greater than the one before it. In the example at left, if there are 1 or 2 clauses, then both are required. If there are 3-9 clauses all but 25% are required. If there are more then 9 clauses, all but three are required.|

当指定 `mm` 的值时，需铭记以下几点：

* When dealing with percentages, negative values can be used to get different behavior in edge cases. 75%
and -25% mean the same thing when dealing with 4 clauses, but when dealing with 5 clauses 75% means
3 are required, but -25% means 4 are required.
* If the calculations based on the parameter arguments determine that no optional clauses are needed, the
usual rules about Boolean queries still apply at search time. (That is, a Boolean query containing no
required clauses must still match at least one optional clause).
* No matter what number the calculation arrives at, Solr will never use a value greater than the number of
optional clauses, or a value less than 1. In other words, no matter how low or how high the calculated
result, the minimum number of required matches will never be less than 1 or greater than the number of
clauses.
* When searching across multiple fields that are configured with different query analyzers, the number of
optional clauses may differ between the fields. In such a case, the value specified by mm applies to the
maximum number of optional clauses. For example, if a query clause is treated as stopword for one of the
fields, the number of optional clauses for that field will be smaller than for the other fields. A query with
such a stopword clause would not return a match in that field if mm is set to 100% because the removed
clause does not count as matched.

The default value of mm is 100% (meaning that all clauses must match).

### <a name="pf"><a>参数 `pf`

Once the list of matching documents has been identified using the fq and qf parameters, the pf parameter can
be used to "boost" the score of documents in cases where all of the terms in the q parameter appear in close
proximity.

The format is the same as that used by the qf parameter: a list of fields and "boosts" to associate with each of
them when making phrase queries out of the entire q parameter.

### <a name="ps"><a>参数 `ps`

The ps parameter specifies the amount of "phrase slop" to apply to queries specified with the pf parameter.
Phrase slop is the number of positions one token needs to be moved in relation to another token in order to
match a phrase specified in a query.

### <a name="qs"><a>参数 `qs`

The qs parameter specifies the amount of slop permitted on phrase queries explicitly included in the user's query
string with the qf parameter. As explained above, slop refers to the number of positions one token needs to be
moved in relation to another token in order to match a phrase specified in a query.

### <a name="tie"><a>参数 `tie`

The tie parameter specifies a float value (which should be something much less than 1) to use as tiebreaker in
DisMax queries.

When a term from the user's input is tested against multiple fields, more than one field may match. If so, each
field will generate a different score based on how common that word is in that field (for each document relative to
all other documents). The tie parameter lets you control how much the final score of the query will be
influenced by the scores of the lower scoring fields compared to the highest scoring field.

A value of "0.0" makes the query a pure "disjunction max query": that is, only the maximum scoring subquery
contributes to the final score. A value of "1.0" makes the query a pure "disjunction sum query" where it doesn't
matter what the maximum scoring sub query is, because the final score will be the sum of the subquery scores.
Typically a low value, such as 0.1, is useful.

### <a name="bq"><a>参数 `bq`

The bq parameter specifies an additional, optional, query clause that will be added to the user's main query to
influence the score. For example, if you wanted to add a relevancy boost for recent documents:

```
q=cheese
bq=date:[NOW/DAY-1YEAR TO NOW/DAY]
```

You can specify multiple bq parameters. If you want your query to be parsed as separate clauses with separate
boosts, use multiple bq parameters.

### <a name="bf"><a>参数 `bf`

The bf parameter specifies functions (with optional boosts) that will be used to construct FunctionQueries which
will be added to the user's main query as optional clauses that will influence the score. Any function supported
natively by Solr can be used, along with a boost value. For example:

```
recip(rord(myfield),1,2,3)^1.5
```

Specifying functions with the bf parameter is essentially just shorthand for using the bq param combined with the
`{!func}` parser.
For example, if you want to show the most recent documents first, you could use either of the following:

```
bf=recip(rord(creationDate),1,1000,1000)
  ...or...
bq={!func}recip(rord(creationDate),1,1000,1000)

```

## 提交给 DisMax 查询解析器的查询示例

所有本节的示例 URL 假设你使用的是 Solr 的 "techproducts" 示例：

```bash
bin/solr -e techproducts
```

Normal results for the word "video" using the StandardRequestHandler with the default search field:

```
http://localhost:8983/solr/techproducts/select?q=video&fl=name+score
```

"dismax"处理器被配置为跨字段的，为了确保能够获得更好的匹配结果。
 
```
http://localhost:8983/solr/techproducts/select?defType=dismax&q=video
```

请注意，此实例还配置了默认字段列表，可以在URL中覆盖该列表。

```
http://localhost:8983/solr/techproducts/select?defType=dismax&q=video&fl=*,score
```

你可以自定义搜索哪些字段并且设置权重

```
http://localhost:8983/solr/techproducts/select?defType=dismax&q=video&qf=features^20.0+text^0.3
```

提可以提供特定字段的权重

```
http://localhost:8983/solr/techproducts/select?defType=dismax&q=video&bq=cat:electronics^5.0
```

Another instance of the handler is registered using the qt "instock" and has slightly different configuration
options, notably: a filter for (you guessed it) `inStock:true`) .

```
http://localhost:8983/solr/techproducts/select?defType=dismax&q=video&fl=name,score,inStock

http://localhost:8983/solr/techproducts/select?defType=dismax&q=video&qt=instock&fl=name,score,inStock
```

One of the other really cool features in this handler is robust support for specifying the
"BooleanQuery.minimumNumberShouldMatch" you want to be used based on how many terms are in your user's
query. These allows flexibility for typos and partial matches. For the dismax handler, one and two word queries
require that all of the optional clauses match, but for three to five word queries one missing word is allowed.

```
http://localhost:8983/solr/techproducts/select?defType=dismax&q=belkin+ipod

http://localhost:8983/solr/techproducts/select?defType=dismax&q=belkin+ipod+gibberish

http://localhost:8983/solr/techproducts/select?defType=dismax&q=belkin+ipod+apple
```

就像StandardRequestHandler一样，它支持debugQuery选项来查看解析的查询以及每个文档的分数说明。

```
http://localhost:8983/solr/techproducts/select?defType=dismax&q=belkin+ipod+gibberish&debugQuery=true
 
http://localhost:8983/solr/techproducts/select?defType=dismax&q=video+card&debugQuery=true
```
