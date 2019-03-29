
# Dgraph 学习笔记

> 
- Dgraph集群由不同的节点（Zero，服务器和Ratel）组成，每个节点完成不同功能。
- Dgraph Zero 用于控制群集，将服务器分配到一个组，并在服务器组之间重新平衡数据。
- Dgraph Alpha 用于托管谓词和索引。
- Dgraph Ratel 提供用户界面来运行查询，突变和转换模式。
- 您至少需要一个Dgraph Zero和一个 Dgraph Alpha服务器同时启动。

## 1、安装 Dgraph

- linux下安装

From Install Scripts (Linux/Mac) Install the binaries with
```curl https://get.dgraph.io -sSf | bash```

- Windows 之上安装
从 Github 仓库获取最新版本, 解压获得可执行的二进制文件。

## 2、从可执行程序启动

- 启动 Dgraph zero
该程序控制Dgraph集群，维护成员信息、分配分片和移动分片等。默认使用5080端口与外部通讯
```dgraph zero```

- 启动 Dgraph data server 数据服务，默认使用9080端口与外部通讯
```dgraph alpha --lru_mb 2048 --zero localhost:5080 ```

> 提示：您可以通过修改 lru_mb 参数设置 Dgraph alpha 服务的预设内存。这只是对 Dgraph alpha 服务的一个提示，实际使用率会高于此值。建议将lru_mb设置为可用内存 的三分之一。

- 启动 Ratel Dgraph 的 UI ，通过UI进行修改和查询。 默认使用8000端口对外提供访问
```dgraph-ratel```

## 3、Run Queries
可以通过浏览器打开 http://localhost:8000.连接Ratel进行相关可视化操作。也可以通过命令行执行：curl localhost:8080/query -XPOST -d $'...' 或者是把引号之间的内容通过浏览器执行。

# 教程

## 1、简介

- 添加或删除数据在Dgraph中叫做 mutation（突变）

mutations 有两种标准格式: RDF (Resource Description Framework) N-Quad and JSON (JavaScript Object Notation). RDF 是一个gua is a widely used standard in Graph or Ontology systems.

- In Dgraph the mutation operations consist of two patterns: blank UID reference or explicit UID reference.
  - Blank UID
  在 mutation 中，任何与UID显式引用格式不匹配的定义都可以视为空白UID引用，也称为空白节点。空节点的结构包括:下划线+冒号+唯一名称(标识符)
  ```
  正确：_:michael and _:amit. 
  也是但不推荐：<_:uid43> or <_:uid4e030f> <其他错误写法>
  ```
  - Explicit UID
  To do a mutation for an existing node, you can refer to the node’s UID directly in the mutation. The syntax is simple:
  ```<THE-UID> # or 
     <0x4e030f> <somePredicate> "some new data" .
  ```

> Note 你应该只引用存在的 UIDs，如果你对不存在的UID执行mutation操作, Dgraph 将返回错误。 使用 blank nodes创建新节点，Dgraph为新的节点分配一个新的 UID。

- **在图中，对象(或实体)称为节点，关系称为边或谓词。**

- 图表不仅仅适用于社交网络。其他的合适场景包括

相互连接的数据，比如需要连接的SQL表
高级搜索
推荐引擎
模式检测
网络，如计算机、道路和电信
流程，比如业务流程和生物流程
事件之间的因果关系或其他联系
公司或市场的结构

- 图形数据库是为存储和查询图形而优化的数据库。

在关系处理方面，图形数据库比SQL数据库快得多。SQL数据库被类似图形的数据拖住了，因为遵循边意味着连接表;有时大表;边越多，连接越多，需要加载和处理的数据也越多。
在图形数据库中，边是一个基本结构，所以跟随边是一个简单的查找，这使得这个操作非常快。Dgraph是一个图形数据库。它优化了图形数据的存储和查询速度。

## 查询语言GraphQL+-的基本特性

- 每个查询 都有一个名字，返回结果也以该名字标记

- 通过检索条件 func: ... 匹配节点. 函数 eq 找到相同的节点. 结果是匹配的节点及有节点引出向外的边。示例：
```
    {
        find_michael(func: eq(name@., "Michael")) {
            uid
            name@.
            age
        }
    }
```

- 在Dgraph和GraphQL+-中，查询返回的是图形，而不是表或数据列表。

- 函数及筛选 Functions and filtering

Functions can only be applied to predicates that have been indexed - that’s part of the lesson about schema.
The logical connectives AND, OR and NOT combine multiple functions in a filter.

- 排序 Sorting (orderasc or orderdesc) 

```friend (orderasc: age) {...}```

- 分页 Pagination (first, offset and after)

```friend (orderasc: name@., offset: 1, first: 2) {...}```

- 计数 Count

```
{
    name
    age
    count(friend)
}
```

> The root func: 只接受一个函数 and doesn’t accept AND, OR and NOT connectives as in filters. So the syntax (func: ...) @filter(... AND ...) is required when filtering on multiple properties at the root.

```
{ 
    lots_of_friends(func: ge(count(friend), 2)) @filter(ge(age, 20) AND lt(age, 30)) { 
        name@. 
        age 
        friend { name@. } 
    } 
}
```

- 函数 Has

The function has(edge_name) returns nodes that have an outgoing edge of the given name.

- 别名 Alias

The output graph can set names for edges in the output with aliasing. 如下例，number_of_friends 是别名。

```
number_of_friends : count(friend)
```

- Cascade

The @cascade 表示除去不完全包含匹配边的节点。

- 规范化 Normalize

The @normalize 表示仅返回设置了别名的边对应的节点，同时去除嵌套。

- 注释 Comments

Queries 可以支持包含注释，以 # 开头的行是注释。

- Facets : Edge attributes 边的属性

Dgraph支持facet(边上的键值对)作为RDF三元组的扩展。也就是说，facet向边添加属性，而不是向节点添加属性。例如，两个节点之间的朋友边缘可能具有表示亲密的友谊的布尔属性。Facets 也可以用作边的权重。

## Schema 模式，用于描述你的数据

- 添加修改模式 Adding schema - mutating schema

Dgraph存储了一个描述 predicates（谓词、边）类型的 schema （模式）。

当我们想要向现有模式添加新数据时，我们可以直接添加它。但是如果我们想在新模式中添加新数据，我们有两个选择：1、添加数据并让Dgraph计算出模式，2、或者指定模式，然后添加数据。但是函数和过滤只能应用于索引谓词，为此我们需要指定模式。

- 添加修改数据 mutating data

- 外部标识符

Dgraph 不支持为节点设置外部 id。如果应用程序需要 Dgraph 分配的 uid 以外的节点的惟一标识符，那么这些标识符必须作为边提供。由用户应用程序来确保此类 id/key 的唯一性。

- 多语言支持

```
_:myID <an_edge> "something"@en .
_:myID <an_edge> "某物"@zh-Hans .

See the JSON:
{
    "set": [
        {
            "uid": "_:myID",
            "an_edge@en": "something",
            "an_edge@zh-Hans": "某物"
        }
    ]
}
```

- 反向边 Reverse edges 

边是定向的，查询时不能反向遍历边。在数据建模方面，一些反向边缘总是有意义的，比如friend。其他的，例如boss_of，有时是双向的，但并不总是双向的。  

在两个方向查询有两个方式：1、将反向边添加到模式中，并添加所有反向边数据。2、告诉 Dgraph 始终使用模式中的 @reverse 关键字存储反向边缘。

运行模式突变，Dgraph将计算所有反向边缘。an_edge的反边缘是~an_edge。  

- 集成已有数据 Integrating existing data

尝试使用以前突变中的空白节点是行不通的。空白节点不会持久存储在存储中，因此当引用在以前的突变中创建的节点时:

```
不可以通过如下方式:
_:sarah <works_for> _:company1 .
而是
<uid-for-sarah> <works_for> <uid-for-company1> .
```

使用Dgraph实例的uid编写一个连接公司和友谊数据的突变。整个过程通常是通过编程完成的——查询数据以获得uid，制定突变，然后批处理更新。

- 删除数据

三种方式

1. <uid> <edge> <uid>/"value" . 删除单个组
2. <uid> <edge> * . 删除边所有的相关组
3. <uid> * * . 删除其下所有组

删除关系及子节点
```
 {
    "delete": [
        {
            "uid": "0x2", # Answer UID.
            "comment": {
                "uid": "0x3" # Delete relation (edge) with the Comment
        }
        },
        {
            "uid": "0x3" # Delete the actual comment
        }
    ]
}
```

- 谓词查询 Predicate Query

对于图_predicate_中的任何节点，查询所有输出边的名称。
> 注意，这与设计图表时的模式或意图不同，因为任何给定节点可能具有或不具有其他节点具有的所有谓词。通常，大型图通常表示建模数据的部分视图。我们分析数据的工作是理解数据中的关系，同时处理缺陷或不完整的知识。

- 展开谓词 Expand Predicate

expand(...predicates...) 

In Ggraph github repository you’ll find a dataset about movies, directors and actors.This dataset is a one million triple subset of a larger dataset of movies that contains around 230000 actors and around 86000 movies.

- Query Blocks, Query Variables and Aggregation

dgraph 可以通过as关键词将一个查询块的任意部分设为一个变量, 以供后面的子查询或者其它查询块使用. 这个变量本质上是一个uid列表, 因此要利用uid函数进行引用。

```
{
  PJ as var(func:allofterms(name@en, "Peter Jackson")) @normalize @cascade {
    F as director.film
  }

  peterJ(func: uid(PJ)) @normalize @cascade {
    name : name@en
    actor.film {
      performance.film @filter(uid(F)) {
        film_name: name@en
      }
      performance.character {
        character: name@en
      }
    }
  }
}

{
  var(func: allofterms(name@en, "Taraji Henson")) {
    actor.film {
      F as performance.film {
        G as genre
      }
    }
  }

  Taraji_films_by_genre(func: uid(G)) {
    genre_name : name@en
    films : ~genre @filter(uid(F)) {
      film_name : name@en
    }
  }
}
联合
{
  var(func: allofterms(name@en, "Peter Jackson")) {
    F_PJ as director.film {
      starring{
        A_PJ as performance.actor
      }
    }
  }

   var(func: allofterms(name@en, "Martin Scorsese")) {
    F_MS as director.film {
      starring{
        A_MS as performance.actor
      }
    }
  }

  actors(func: uid(A_PJ)) @filter(uid(A_MS)) @cascade {
    actor: name@en
    actor.film {
      performance.film @filter (uid(F_PJ, F_MS)) {
      	name@en
      }
    }
  }
}

Value variables - min and max
{
  q(func: allofterms(name@en, "Ang Lee")) {
    director.film {
      uid
      name@en

      # Count the number of starring edges for each film
      num_actors as count(starring)

      # In this block, num_actors is the value calculated for this film.
      # The film with uid and name
    }

    # Here num_actors is a map of film uid to value for all
    # of Ang Lee's films
    #
    # It can't be used directly, but aggregations like min and max
    # work over all the values in the map

    most_actors : max(val(num_actors))
  }

  # to use num_actors in another query, make sure it's done in a context
  # where the film uid to value map makes sense.
}

Value variables - sum and avg
{
  ID as var(func: allofterms(name@en, "Steven Spielberg")) {
    director.film {
      num_actors as count(starring)
    }
    average as avg(val(num_actors))
  }

  avs(func: uid(ID)) @normalize {
    name : name@en
    average_actors : val(average)
    num_films : count(director.film)
  }
}

Value variables: filtering and ordering
{
  ID as var(func: allofterms(name@en, "Steven")) {
    director.film {
      num_actors as count(starring)
    }
    average as avg(val(num_actors))
  }

  avs(func: uid(ID), orderdesc: val(average)) @filter(ge(val(average), 40)) @normalize {
    name : name@en
    average_actors : val(average)
    num_films : count(director.film)
  }
}

Value variables: math functions
{
	var(func:allofterms(name@en, "Jean-Pierre Jeunet")) {
		name@en
		films as director.film {
			stars as count(starring)
			directors as count(~director.film)
			ratio as math(stars / directors)
		}
	}

	best_ratio(func: uid(films), orderdesc: val(ratio)){
		name@en
		stars_per_director : val(ratio)
		num_stars : val(stars)
	}
}

{ # Get all directors
  var(func: has(director.film)) @cascade {
    director.film {
      date as initial_release_date
    }
    # Store maxDate as a variable
    maxDate as max(val(date))
    daysSince as math(since(maxDate)/(24*60*60))
  }

  # Order by maxDate
  me(func: uid(maxDate), orderdesc: val(maxDate), first: 10) {
    name@en
    days : val(daysSince)

    # For each director, sort by release date and get latest movie.
    director.film(orderdesc: initial_release_date, first: 1) {
      name@en
      initial_release_date
    }
}

GroupBy
{
  var(func:allofterms(name@en, "Steven Spielberg")) {
    director.film @groupby(genre) {
      a as count(uid)
    }
  }

  byGenre(func: uid(a), orderdesc: val(a)) {
    name@en
    num_movies : val(a)
  }
}

expand and _predicate_
{
  var(func:allofterms(name@en, "Cherie Nowlan")) {
    pred as _predicate_
  }

  q(func:allofterms(name@en, "Cherie")) {
    expand(val(pred)) { expand(_all_) }
  }
}
```

## Search - more ways to find nodes

- 索引

当Dgraph基于过滤器搜索字符串、日期或其他值时，它需要一个索引来提高搜索效率。

int、float、geo和date都有默认索引，但是string有选择索引类型的选项。可以为同一个字符串值谓词构建多个索引。  

对于字符串，可以使用以下索引：  
- term(默认值)，用于allofterms, anyofterms
- exactly，用于不等式——匹配整个字符串
- 哈希，就像匹配精确但哈希的字符串一样，适用于长字符串
- fulltext，用于alloftext的全文搜索，用于anyoftext三元组，
- trigram 用于正则表达式

Regular Expressions
```
{
  aliens(func: regexp(name@en, /.*alien.*/i)) @cascade {
    name@en
    ~genre {
      name@en
      starring {
        performance.actor @filter(regexp(name@en,
          /(.*ali.*e.*n.*)|(.*a.*lie.*n.*)|(.*a.*l.*ien.*)/i)) {
          name@en
        }
      }
    }
  }
}
```

- 全文搜索

全文搜索是谷歌为web页面所做的。它与术语匹配不同，因为它试图尊重语言、语法和时态。例如，将搜索词run与包含run、running和run的文档匹配。
它不完全匹配术语，而是使用:  
词干检查:查找一个常见的基本单词，以便在时态、复数/单数或其他屈折变化方面的差异仍然匹配  
停止单词:删除and、or、it和可能经常出现而无法搜索的单词。  

## 集群部署
假设集群机器数为3台，下载和安装：此步骤与单机部署的下载安装相同, 在3台机器上均执行上述指令.

启动服务

```
//选择某一台机器上启动zero服务, 用my参数指定自身的host, 以便和集群的alpha服务通讯, 用replicas指定副本数
dgraph zero  --my dev210.sugo.net:5080 --replicas 3  
//在三台机器上启动alpha服务,  用my参数指定自身的host 
dgraph alpha --lru_mb 2048 --my dev210.sugo.net:7080 --zero dev210.sugo.net:5080   //210机器执行
dgraph alpha --lru_mb 2048 --my dev211.sugo.net:7080 --zero dev210.sugo.net:5080   //211机器执行
dgraph alpha --lru_mb 2048 --my dev212.sugo.net:7080 --zero dev210.sugo.net:5080   //212机器执行
//选择某一台机器启动ratel服务
dgraph-ratel   
```  

> 注意:集群模式下, 若zero设置了replicas参数, 则集群必须有半数以上的alpha节点存活才可对外提供服务. 在本例下, 即需要2个及以上数量的alpha节点存活才能提供服务.
