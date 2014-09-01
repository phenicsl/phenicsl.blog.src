Date: 2014-08-14 00:15:00
Title: Cayley代码分析 - 基本结构
Tags: golang, graphdb
Category: Graph Database

Cayley是由golang实现的开源基于triple store的graph database. caley可以支持多个backed, 比如LevelDB, Bolt, MongoDB和In-Memory Store. 也同时支持Gremlin和MQL的查询方式.

这篇文章主要介绍 In-Memory Store的方式下, Cayley采用JavaScript/Gremlin进行查询的过程, 关注点是分析In-Memory方式下的数据结构和查询流程, 描述Cayley相对于一般的NoSQL数据库或者关系型数据库Mysql的查询优化.


<!-- PELICAN_END_SUMMARY -->

## TripleStore

graph database的存储形式有很多种, 而TripleStore是其中常见的一种, 先看TripleStore 的例子:

    :::
	1. Alice follows Chester .
	2. Alice follows Mike .
	3. Mike follows Chester .

TripleStore数据存储单元被成为Triple, 即三元式, 在Cayley中, 每一个triple还可以有一个label, 因此实际上是Quad(四元式). Quad由四个部分组成:
* Subject: 表示发起动作的一方, 比如Quad 1中, Alice是subject
* Object: 表示接受动作的一方, 比如Quad 1中, Chester是object
* Predicate: 表示Subject和Object的关系, 在Quad 1中, predicate是follows
* Label: 表示整个Quad的标签, 在上面的例子中, lable都是".", 表示为空.

Quad用来表示subject和object之间的关系, 关系用predicate来表示, 比如例子中的三个Quad, 都是用来表示 "follows" 的关系.

## JavaScript/Gremlin

cayley 支持使用JavaScript对数据进行查询, 并且事先定义了类似于 Gremlin的graph对象. cayley内部使用了一个称为[otto][1]的JavaScript解释器, 当用户在REPL直接输入Javascript语句时, 或者通过HTTP发送JavaScript请求, cayley都通过JavaScript解释器来解析并执行JavaScript代码, 对graph database进行查询.

[otto][1] 是一个generic的JavaScript解释器, 本身并不包含graph database相关的内容, cayley在此基础上定义了类似Gremlin的JavasScript API来提供 graph database查询功能.

Cayley在JavaScript环境中预定义了 ``graph`` 对象来引入graph database的功能. `graph`是一个特殊的对象, 所有的graph database查询都从graph开始. graph对象定义了 ``graph.Vertex([nodeId], [nodeId]...)`` 方法返回指定了特定vertex的Path Object, ``Vertex`` 可以接受一个或者多个 vertex的name作为参数. 而该方法实际返回的是一个Path Object, Path Object定义了 ``travels methods`` 来进行 graph database的遍历. 所以可以把 ``Vertex`` 函数看作用来指定图遍历的起点. 我们可以把多个调用比如下面的例子 ::

   :::JavaScript
   graph.Vertex("Alice").Out("follows").All()

上面的例子会返回 Alice "follows" 的所有vertex. 这里, `graph.Vertex("Alice")` 首先会返回指定了vertex为 "Alice" 的 Path Object, 随后的调用 `Out` 函数是一个 ``traverls method``, `Out("follows")` 表示从之前返回的 Path Object 开始遍历所有向外的 "follows" 的路径, 这同样会得到一个 Path Object, 表示从 "Alice" 开始, 遍历 "follows" 路径(edge), 得到的所有 vertex. 最后调用的 ``All`` 是一个 ``finals method``, 当调用到 ``finals method`` 时, 便开始输出结果.

从上面的例子可以看出 JavaScript/Gremlin的基本语法. 下面我们来做一下归纳:
1. `graph` JavaScript 对象引入 graph database功能, `graph.Vertex()` 方法返回表示特定vertex的Path Object.
2. Path Object 定义了 ``traverls methods`` 和 ``finals methods``
3. ``traverls methods`` 定义了 graph database 的遍历操作, 在 Path Object上调用 ``traverls methods`` 同样返回Path Object. 比如上面的 ``Out()`` 方法.
4. ``finals methods`` 最终返回整个遍历的结果. 比如上面的 ``All()`` 方法.

## Memory TripleStore

cayley支持多种数据存储的后端, 包括:
* LevelDB
* Bolt
* MongoDB
* In-Memory, ephemeral

这里, 我们只介绍In-Memory的存储方式.

在cayley中, In-Memory的TripleStore定义为下面的样子:

    :::go
    type TripleStore struct {
        idCounter       int64
        tripleIdCounter int64
        idMap           map[string]int64
        revIdMap        map[int64]string
        triples         []quad.Quad
        size            int64
        index           TripleDirectionIndex
    }

* idCounter : 当前的node id count, 当quad被添加到TripeStore中之后, 其中的每一个 node 都会被赋予一个Id. idCounter记录了当前Id的最大值. 当一个新的node加入TripeStore之后, 会执行的下面的操作:

    :::go
	node.Id = idCounter
	idCount += 1

* tripleIdCounter : 当前的 quad id count, 当quad被添加到TripleStore中之后, 

在cayley中, Quad的定义如下:

    :::go
	type Quad struct {
	    Subject   string `json:"subject"`
        Predicate string `json:"predicate"`
        Object    string `json:"object"`
	    Label     string `json:"label,omitempty"`
    }

	

## 基本结构



[1]: https://github.com/robertkrimen/otto
