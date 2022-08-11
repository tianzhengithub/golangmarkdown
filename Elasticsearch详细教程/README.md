### Elasticsearch详细笔记

### 一、Elasticsearch概述

#### 1.1 开篇

结构化数据

![img](images/a421a414ea8fb8e25c466add5b00d31f.png)

非结构化数据

![img](images/109d3c62c1f36b4f352170488b516709.png)



半结构化数据

![img](images/48742ba6bbdc47e0dae2754027f30a31.png)

#### 1.2 技术选型

**Elasticsearch 是什么**

The Elastic Stack, 包括 Elasticsearch、 Kibana、 Beats 和 Logstash（也称为 ELK Stack）。能够安全可靠地获取任何来源、任何格式的数据，然后实时地对数据进行搜索、分析和可视化。

Elaticsearch，简称为 ES， ES 是一个开源的高扩展的分布式全文搜索引擎， 是整个 ElasticStack 技术栈的核心。

它可以近乎实时的存储、检索数据；本身扩展性很好，可以扩展到上百台服务器，处理 PB 级别的数据。



**全文搜索引擎**

Google，百度类的网站搜索，它们都是根据网页中的关键字生成索引，我们在搜索的时候输入关键字，它们会将该关键字即索引匹配到的所有网页返回；还有常见的项目中应用日志的搜索等等。对于这些非结构化的数据文本，关系型数据库搜索不是能很好的支持。

一般传统数据库，全文检索都实现的很鸡肋，因为一般也没人用数据库存文本字段。进行全文检索需要扫描整个表，如果数据量大的话即使对 SQL 的语法优化，也收效甚微。建立了索引，但是维护起来也很麻烦，对于 insert 和 update 操作都会重新构建索引。

基于以上原因可以分析得出，在一些生产环境中，使用常规的搜索方式，性能是非常差的：

- 搜索的数据对象是大量的非结构化的文本数据。

- 文件记录量达到数十万或数百万个甚至更多。

- 支持大量基于交互式文本的查询。

- 需求非常灵活的全文搜索查询。

- 对高度相关的搜索结果的有特殊需求，但是没有可用的关系数据库可以满足。

- 对不同记录类型、非文本数据操作或安全事务处理的需求相对较少的情况。为了解决结构化数据搜索和非结构化数据搜索性能问题，我们就需要专业，健壮，强大的全文搜索引擎 。

这里说到的全文搜索引擎指的是目前广泛应用的主流搜索引擎。它的工作原理是计算机索引程序通过扫描文章中的每一个词，对每一个词建立一个索引，指明该词在文章中出现的次数和位置，当用户查询时，检索程序就根据事先建立的索引进行查找，并将查找的结果反馈给用户的检索方式。这个过程类似于通过字典中的检索字表查字的过程。

#### 1.3 Elasticsearch 应用案例

- GitHub: 2013 年初，抛弃了 Solr，采取 Elasticsearch 来做 PB 级的搜索。 “GitHub 使用Elasticsearch 搜索 20TB 的数据，包括 13 亿文件和 1300 亿行代码”。

- 维基百科：启动以 Elasticsearch 为基础的核心搜索架构

- 百度：目前广泛使用 Elasticsearch 作为文本数据分析，采集百度所有服务器上的各类指标数据及用户自定义数据，通过对各种数据进行多维分析展示，辅助定位分析实例异常或业务层面异常。目前覆盖百度内部 20 多个业务线（包括云分析、网盟、预测、文库、直达号、钱包、 风控等），单集群最大 100 台机器， 200 个 ES 节点，每天导入 30TB+数据。

- 新浪：使用 Elasticsearch 分析处理 32 亿条实时日志。

- 阿里：使用 Elasticsearch 构建日志采集和分析体系。

- Stack Overflow：解决 Bug 问题的网站，全英文，编程人员交流的网站。
  

#### 1.4 -教学大纲

- 第1章 Elasticsearch概述

- 第2章 Elasticsearch入门

- 第3章 Elasticsearch环境

- 第4章 Elasticsearch进阶

- 第5章 Elasticsearch集成

- 第6章 Elasticsearch优化

- 第7章 Elasticsearch面试题
  

### 二、Elasticsearch入门

#### 2.1 [官方网址](https://www.elastic.co/cn/)

[官方文档](https://www.elastic.co/guide/index.html)

[Elasticsearch 7.8.0下载页面](https://www.elastic.co/cn/downloads/past-releases/elasticsearch-7-8-0)

Windows 版的 Elasticsearch 压缩包，解压即安装完毕，解压后的 Elasticsearch 的目录结构如下 ：

![1](./images/1.png)

解压后，进入 bin 文件目录，点击 elasticsearch.bat 文件启动 ES 服务 。

注意： 9300 端口为 Elasticsearch 集群间组件的通信端口， 9200 端口为浏览器访问的 http协议 RESTful 端口。

打开浏览器，输入地址： http://localhost:9200，测试返回结果，返回结果如下：


```json
{
  "name" : "192.local",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "hEnwz9zORIifwL6HovF2fA",
  "version" : {
    "number" : "7.17.5",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "8d61b4f7ddf931f219e3745f295ed2bbc50c8e84",
    "build_date" : "2022-06-23T21:57:28.736740635Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

#### 2.2 RESTful & JSON

REST 指的是一组架构约束条件和原则。满足这些约束条件和原则的应用程序或设计就是 RESTful。 Web 应用程序最重要的 REST 原则是，客户端和服务器之间的交互在请求之间是无状态的。从客户端到服务器的每个请求都必须包含理解请求所必需的信息。如果服务器在请求之间的任何时间点重启，客户端不会得到通知。此外，无状态请求可以由任何可用服务器回答，这十分适合云计算之类的环境。客户端可以缓存数据以改进性能。

在服务器端，应用程序状态和功能可以分为各种资源。资源是一个有趣的概念实体，它向客户端公开。资源的例子有：应用程序对象、数据库记录、算法等等。每个资源都使用 URI(Universal Resource Identifier) 得到一个唯一的地址。所有资源都共享统一的接口，以便在客户端和服务器之间传输状态。使用的是标准的 HTTP 方法，比如 GET、 PUT、 POST 和DELETE。

在 REST 样式的 Web 服务中，每个资源都有一个地址。资源本身都是方法调用的目
标，方法列表对所有资源都是一样的。这些方法都是标准方法，包括 HTTP GET、 POST、PUT、 DELETE，还可能包括 HEAD 和 OPTIONS。简单的理解就是，如果想要访问互联网上的资源，就必须向资源所在的服务器发出请求，请求体中必须包含资源的网络路径， 以及对资源进行的操作(增删改查)。

REST 样式的 Web 服务若有返回结果，大多数以JSON字符串形式返回。

#### 2.3 Postman客户端工具

如果直接通过浏览器向 Elasticsearch 服务器发请求，那么需要在发送的请求中包含
HTTP 标准的方法，而 HTTP 的大部分特性且仅支持 GET 和 POST 方法。所以为了能方便地进行客户端的访问，可以使用 Postman 软件Postman 是一款强大的网页调试工具，提供功能强大的 Web API 和 HTTP 请求调试。

软件功能强大，界面简洁明晰、操作方便快捷，设计得很人性化。 Postman 中文版能够发送任何类型的 HTTP 请求 (GET, HEAD, POST, PUT…)，不仅能够表单提交，且可以附带任意类型请求体。


#### 2.4 倒排索引

![2](./images/2.png)

Elasticsearch 是**面向文档型数据库**，一条数据在这里就是一个文档。 为了方便大家理解，我们将 Elasticsearch 里存储文档数据和关系型数据库 MySQL 存储数据的概念进行一个类比

![3](./images/3.png)

ES 里的 Index 可以看做一个库，而 Types 相当于表， Documents 则相当于表的行。这里 Types 的概念已经被逐渐弱化， Elasticsearch 6.X 中，一个 index 下已经只能包含一个type， Elasticsearch 7.X 中, Type 的概念已经被删除了。

#### 2.5 HTTP-索引-创建

对比关系型数据库，创建索引就等同于创建数据库。

在 Postman 中，向 ES 服务器发 PUT 请求 ： http://127.0.0.1:9200/shopping

请求后，服务器返回响应：

```json
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "shopping"
}
```

后台日志：

```bash
[2022-08-10T15:47:07,160][INFO ][o.e.c.m.MetadataCreateIndexService] [192.local] [shopping] creating index, cause [api], templates [], shards [1]/[1]
```

如果重复发 PUT 请求 ： http://127.0.0.1:9200/shopping 添加索引，会返回错误信息 :

```json
{
    "error": {
        "root_cause": [
            {
                "type": "resource_already_exists_exception",
                "reason": "index [shopping/J0WlEhh4R7aDrfIc3AkwWQ] already exists",
                "index_uuid": "J0WlEhh4R7aDrfIc3AkwWQ",
                "index": "shopping"
            }
        ],
        "type": "resource_already_exists_exception",
        "reason": "index [shopping/J0WlEhh4R7aDrfIc3AkwWQ] already exists",
        "index_uuid": "J0WlEhh4R7aDrfIc3AkwWQ",
        "index": "shopping"
    },
    "status": 400
}

```

#### 2.6 HTTP-索引-查询&删除

查看所有索引
在 Postman 中，向 ES 服务器发 GET 请求 ： http://127.0.0.1:9200/_cat/indices?v

这里请求路径中的_cat 表示查看的意思， indices 表示索引，所以整体含义就是查看当前 ES服务器中的所有索引，就好像 MySQL 中的 show tables 的感觉，服务器响应结果如下 :

```json
health status index            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .geoip_databases MByvMPJcRC6q1xFiYisslQ   1   0         40            0     37.9mb         37.9mb
yellow open   shopping         dxKLUaNBSXeKPRROyeFlsg   1   1          0            0       226b           226b
```

![4](./images/4.png)

#### 2.7 查看单个索引

在postman中，向es服务器发送get请求：http://127.0.0.1:9200/shopping

返回结果如下：

```json
{
    "shopping": {
        "aliases": {},
        "mappings": {},
        "settings": {
            "index": {
                "routing": {
                    "allocation": {
                        "include": {
                            "_tier_preference": "data_content"
                        }
                    }
                },
                "number_of_shards": "1",
                "provided_name": "shopping",
                "creation_date": "1660117627142",
                "number_of_replicas": "1",
                "uuid": "dxKLUaNBSXeKPRROyeFlsg",
                "version": {
                    "created": "7170599"
                }
            }
        }
    }
}
```

#### 2.8 删除索引

在postman中，向 ES 服务器发 DELETE 请求： http://127.0.0.1:9200/shopping

返回结果如下：

```json
{
    "acknowledged": true
}
```

再次查看所有索引，GET http://127.0.0.1:9200/_cat/indices?v，返回结果如下：

```
health status index uuid pri rep docs.count docs.deleted store.size pri.store.size
```

成功删除。

#### 2.9 HTTP-文档-创建（PUT&POST)

假设索引已经创建好了，接下来我们来创建文档，并添加数据。这里的文档可以类比为关系型数据库中的表数据，添加的数据格式为 JSON 格式

在 Postman 中，向 ES 服务器发 POST 请求 ： http://127.0.0.1:9200/shopping/_doc，请求体JSON内容为：

```json
{
    "title":"小米手机",
    "category":"小米",
    "images":"http://www.gulixueyuan.com/xm.jpg",
    "price":3999.00
}
```



![5](./images/5.png)

注意，此处发送请求的方式必须为 POST，不能是 PUT，否则会发生错误 。

返回结果：

```json
{
    "_index": "shopping",
    "_type": "_doc",
    "_id": "_XPLhoIBg3C-pJdYUv6X",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```

上面的数据创建后，由于没有指定数据唯一性标识（ID），默认情况下， ES 服务器会随机生成一个。

如果想要自定义唯一性标识，需要在创建时指定： http://127.0.0.1:9200/shopping/_doc/1，请求体JSON内容为：

```json
{
    "title":"小米手机",
    "category":"小米",
    "images":"http://www.gulixueyuan.com/xm.jpg",
    "price":3999.00
}
```

返回结果如下：

```json
{
    "_index": "shopping",
    "_type": "_doc",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 1,
    "_primary_term": 1
}
```

**此处需要注意：如果增加数据时明确数据主键，那么请求方式也可以为 PUT。**

#### 2.10 HTTP-查询-主键 查询 & 全查询

查看文档时，需要指明文档的唯一性标识，类似于 MySQL 中数据的主键查询

在 Postman 中，向 ES 服务器发 GET 请求 ： http://127.0.0.1:9200/shopping/_doc/1 。

返回结果如下：

```json
{
    "_index": "shopping",
    "_type": "_doc",
    "_id": "1",
    "_version": 1,
    "_seq_no": 1,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "title": "小米手机",
        "category": "小米",
        "images": "http://www.gulixueyuan.com/xm.jpg",
        "price": 3999
    }
}

```

查找不存在的内容，向 ES 服务器发 GET 请求 ： http://127.0.0.1:9200/shopping/_doc/1001。

返回结果如下：

```json
{
    "_index": "shopping",
    "_type": "_doc",
    "_id": "1001",
    "found": false
}

```

查看索引下所有数据，向 ES 服务器发 GET 请求 ： http://127.0.0.1:9200/shopping/_search。

返回结果如下：

```json
{
    "took": 726,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 1.0,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "1",
                "_score": 1.0,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

#### 2.11 HTTP-全量修改 & 局部修改 & 删除

##### 2.11.1 全量修改

和新增文档一样，输入相同的 URL 地址请求，如果请求体变化，会将原有的数据内容覆盖

在 Postman 中，向 ES 服务器发 POST 请求 ： http://127.0.0.1:9200/shopping/_doc/1

请求体JSON内容为:

```json
{
    "title":"华为手机",
    "category":"华为",
    "images":"http://www.gulixueyuan.com/hw.jpg",
    "price":1999.00
}
```

修改成功后，服务器相应结果：

```json
{
    "_index": "shopping",
    "_type": "_doc",
    "_id": "1",
    "_version": 3,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 3,
    "_primary_term": 1
}
```

##### 2.11.2 局部修改

修改数据时，也可以只修改某一给条数据的局部信息

在 Postman 中，向 ES 服务器发 POST 请求 ： http://127.0.0.1:9200/shopping/_update/1。

请求体JSON内容为:

```json
{
	"doc": {
		"title":"小米手机",
		"category":"小米"
	}
}
```

返回结果如下：

```json
{
    "_index": "shopping",
    "_type": "_doc",
    "_id": "1",
    "_version": 4,
    "_seq_no": 4,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "title": "xaiomi手机",
        "category": "小米",
        "images": "http://www.gulixueyuan.com/hw.jpg",
        "price": 1999.00
    }
}
```

##### 2.11.3 删除

删除一个文档不会立即从磁盘上移除，它只是被标记成已删除（逻辑删除）。

在 Postman 中，向 ES 服务器发 DELETE 请求 ： http://127.0.0.1:9200/shopping/_doc/1

返回结果：

```json
{
    "_index": "shopping",
    "_type": "_doc",
    "_id": "1",
    "_version": 5,
    "result": "deleted",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 5,
    "_primary_term": 1
}
```

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_doc/1，查看是否删除成功：

```json
{
    "_index": "shopping",
    "_type": "_doc",
    "_id": "1",
    "found": false
}

```

#### 2.12 HTTP-条件查询 & 分页查询 & 查询排序

##### 2.12.1 条件查询

假设有以下文档内容，（在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search）：

```json
{
    "took": 5,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 6,
            "relation": "eq"
        },
        "max_score": 1,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "ANQqsHgBaKNfVnMbhZYU",
                "_score": 1,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "A9R5sHgBaKNfVnMb25Ya",
                "_score": 1,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 1999
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "BNR5sHgBaKNfVnMb7pal",
                "_score": 1,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 1999
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "BtR6sHgBaKNfVnMbX5Y5",
                "_score": 1,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 1999
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "B9R6sHgBaKNfVnMbZpZ6",
                "_score": 1,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 1999
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "CdR7sHgBaKNfVnMbsJb9",
                "_score": 1,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 1999
                }
            }
        ]
    }
}

```

##### 2.12.2 URL带参查询

**查找category为小米的文档**，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search?q=category:小米，返回结果如下：

```json
{
    "took": 33,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.5753642,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 0.5753642,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

上述为URL带参数形式查询，这很容易让不善者心怀恶意，或者参数值出现中文会出现乱码情况。为了避免这些情况，我们可用使用带JSON请求体请求进行查询。

##### 2.12.3 请求体带参查询

接下带JSON请求体，还是**查找category为小米的文档**，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下

```json
{
	"query":{
		"match":{
			"category":"小米"
		}
	}
}
```

返回结果如下：

```json
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 0.9400072,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 0.9400072,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "3",
                "_score": 0.9400072,
                "_source": {
                    "title": "xiaomi手机",
                    "category": "小米",
                    "images": "http://www.xiaomi.com/xm.jpg",
                    "price": 6999.00
                }
            }
        ]
    }
}
```

##### 2.12.4 带请求体方式的查找所有内容

**查找所有文档内容**，也可以这样，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"match_all":{}
	}
}
```

则返回所有文档内容：

```json
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 1.0,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.0,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "3",
                "_score": 1.0,
                "_source": {
                    "title": "xiaomi手机",
                    "category": "小米",
                    "images": "http://www.xiaomi.com/xm.jpg",
                    "price": 6999.00
                }
            }
        ]
    }
}
```

##### 2.12.5 查询指定字段

**如果你想查询指定字段**，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"match_all":{}
	},
	"_source":["title"]
}
```

返回结果如下：

```json
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 1.0,
                "_source": {
                    "title": "小米手机"
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.0,
                "_source": {
                    "title": "华为手机"
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "3",
                "_score": 1.0,
                "_source": {
                    "title": "xiaomi手机"
                }
            }
        ]
    }
}
```

##### 2.12.6 分页查询

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"match_all":{}
	},
	"from":0,
	"size":2
}
```

返回结果如下：

```json
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 1.0,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.0,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                }
            }
        ]
    }
}
```

##### 2.12.6 查询排序

如果你想通过排序查出价格最高的手机，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"match_all":{}
	},
	"sort":{
		"price":{
			"order":"desc"
		}
	}
}
```

返回结果如下：

```json
{
    "took": 30,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": null,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "3",
                "_score": null,
                "_source": {
                    "title": "xiaomi手机",
                    "category": "小米",
                    "images": "http://www.xiaomi.com/xm.jpg",
                    "price": 6999.00
                },
                "sort": [
                    6999.0
                ]
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": null,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                },
                "sort": [
                    4999.0
                ]
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": null,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                },
                "sort": [
                    3999.0
                ]
            }
        ]
    }
}
```

#### 2.13 HTTP-多条件查询 & 范围查询

##### 2.13.1 多条件查询

假设想找出小米牌子，价格为3999元的。（must相当于数据库的&&）

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"bool":{
			"must":[{
				"match":{
					"category":"小米"
				}
			},{
				"match":{
					"price":3999.00
				}
			}]
		}
	}
}
```

返回结果如下：

```json
{
    "took": 19,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.9400072,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 1.9400072,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

假设想找出小米和华为的牌子。（should相当于数据库的||）

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query": {
		"bool": {
			"should": [{
				"match": {
					"category": "小米"
				}
			}, {
				"match": {
					"category": "华为"
				}
			}],
			"filter": {
				"range": {
					"price": {
						"gt": 4000
					}
				}
			}
		}

	}

}
```

返回结果：

```json
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.9616582,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.9616582,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "3",
                "_score": 0.9400072,
                "_source": {
                    "title": "xiaomi手机",
                    "category": "小米",
                    "images": "http://www.xiaomi.com/xm.jpg",
                    "price": 6999.00
                }
            }
        ]
    }
}
```

#### 2.13.2 范围查询

假设想找出小米和华为的牌子，价格大于2000元的手机。

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"bool":{
			"should":[{
				"match":{
					"category":"小米"
				}
			},{
				"match":{
					"category":"华为"
				}
			}],
            "filter":{
            	"range":{
                	"price":{
                    	"gt":2000
                	}
	            }
    	    }
		}
	}
}

```

返回结果如下：

```json
{
    "took": 8,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.9616582,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.9616582,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 0.9400072,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "3",
                "_score": 0.9400072,
                "_source": {
                    "title": "xiaomi手机",
                    "category": "小米",
                    "images": "http://www.xiaomi.com/xm.jpg",
                    "price": 6999.00
                }
            }
        ]
    }
}
```

#### 2.14 HTTP-全文检索 & 完全匹配 & 高亮查询

##### 2.14.1 全文检索

这功能像搜索引擎那样，如品牌输入“小华”，返回结果带回品牌有“小米”和华为的。

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"match":{
			"category" : "小华"
		}
	}
}
```

返回结果如下：

```json
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 0.9808291,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 0.9808291,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 0.4700036,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "3",
                "_score": 0.4700036,
                "_source": {
                    "title": "xiaomi手机",
                    "category": "小米",
                    "images": "http://www.xiaomi.com/xm.jpg",
                    "price": 6999.00
                }
            }
        ]
    }
}
```

##### 2.14.2 完全匹配

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"match_phrase":{
			"category" : "为"
		}
	}
}
```

返回结果如下：

```json
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.9808291,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 0.9808291,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                }
            }
        ]
    }
}
```

##### 2.13.2 高亮查询

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"match_phrase":{
			"category" : "为"
		}
	},
    "highlight":{
        "fields":{
            "category":{}//<----高亮这字段
        }
    }
}

```

返回结果如下：

```json
{
    "took": 36,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.9808291,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 0.9808291,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                },
                "highlight": {
                    "category": [
                        "华<em>为</em>"
                    ]
                }
            }
        ]
    }
}
```

#### 2.15 HTTP-聚合查询

聚合允许使用者对 es 文档进行统计分析，类似与关系型数据库中的 group by，当然还有很多其他的聚合，例如取最大值max、平均值avg等等。

接下来按price字段进行分组：

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"aggs":{//聚合操作
		"price_group":{//名称，随意起名
			"terms":{//分组
				"field":"price"//分组字段
			}
		}
	}
}
```

返回结果如下：

```json
{
    "took": 31,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "_XPLhoIBg3C-pJdYUv6X",
                "_score": 1.0,
                "_source": {
                    "title": "小米手机",
                    "category": "小米",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 3999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.0,
                "_source": {
                    "title": "华为手机",
                    "category": "华为",
                    "images": "http://www.gulixueyuan.com/xm.jpg",
                    "price": 4999.00
                }
            },
            {
                "_index": "shopping",
                "_type": "_doc",
                "_id": "3",
                "_score": 1.0,
                "_source": {
                    "title": "xiaomi手机",
                    "category": "小米",
                    "images": "http://www.xiaomi.com/xm.jpg",
                    "price": 6999.00
                }
            }
        ]
    },
    "aggregations": {
        "price_group": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
                {
                    "key": 3999.0,
                    "doc_count": 1
                },
                {
                    "key": 4999.0,
                    "doc_count": 1
                },
                {
                    "key": 6999.0,
                    "doc_count": 1
                }
            ]
        }
    }
}
```

上面返回结果会附带原始数据的。若不想要不附带原始数据的结果，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"aggs":{
		"price_group":{
			"terms":{
				"field":"price"
			}
		}
	},
    "size":0
}

```

返回结果如下：

```json
{
    "took": 9,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    },
    "aggregations": {
        "price_group": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
                {
                    "key": 3999.0,
                    "doc_count": 1
                },
                {
                    "key": 4999.0,
                    "doc_count": 1
                },
                {
                    "key": 6999.0,
                    "doc_count": 1
                }
            ]
        }
    }
}
```

若想对所有手机价格求**平均值**。

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"aggs":{
		"price_avg":{//名称，随意起名
			"avg":{//求平均
				"field":"price"
			}
		}
	},
    "size":0
}

```

返回值结果：

```json
{
    "took": 6,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    },
    "aggregations": {
        "price_avg": {
            "value": 5332.333333333333
        }
    }
}
```

#### 2.16 HTTP-映射关系

有了索引库，等于有了数据库中的 database。

接下来就需要建索引库(index)中的映射了，类似于数据库(database)中的表结构(table)。

创建数据库表需要设置字段名称，类型，长度，约束等；索引库也一样，需要知道这个类型下有哪些字段，每个字段有哪些约束信息，这就叫做映射(mapping)。

先创建一个索引：

```json
# PUT http://127.0.0.1:9200/user
```

返回结果：

```json
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "user"
}
```

**创建映射**

```json
# PUT http://127.0.0.1:9200/user/_mapping

{
    "properties": {
        "name":{
        	"type": "text",
        	"index": true
        },
        "sex":{
        	"type": "keyword",
        	"index": true
        },
        "tel":{
        	"type": "keyword",
        	"index": false
        }
    }
}
```

返回结果如下:

```json
{
    "acknowledged": true
}
```

查询映射：

```json
#GET http://127.0.0.1:9200/user/_mapping
```

返回结果如下：

```json
{
    "user": {
        "mappings": {
            "properties": {
                "name": {
                    "type": "text"
                },
                "sex": {
                    "type": "keyword"
                },
                "tel": {
                    "type": "keyword",
                    "index": false
                }
            }
        }
    }
}
```

增加数据

```json
#PUT http://127.0.0.1:9200/user/_create/1001
{
	"name":"小米",
	"sex":"男的",
	"tel":"1111"
}

```

返回结果如下：

```json
{
    "_index": "user",
    "_type": "_doc",
    "_id": "1001",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```

查找 name 含有 "小" 数据：

```json
#GET http://127.0.0.1:9200/user/_search
{
	"query":{
		"match":{
			"name":"小"
		}
	}
}
```

返回结果如下：

```json
{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.2876821,
        "hits": [
            {
                "_index": "user",
                "_type": "_doc",
                "_id": "1001",
                "_score": 0.2876821,
                "_source": {
                    "name": "小米",
                    "sex": "男的",
                    "tel": "1111"
                }
            }
        ]
    }
}
```

查找sex含有”男“数据：

```json
#GET http://127.0.0.1:9200/user/_search
{
	"query":{
		"match":{
			"sex":"男"
		}
	}
}
```

返回结果如下：

```json
{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 0,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    }
}
```

找不想要的结果，只因创建映射时"sex"的类型为"keyword"。

"sex"只能完全为”男的“，才能得出原数据。

```json
#GET http://127.0.0.1:9200/user/_search
{
	"query":{
		"match":{
			"sex":"男的"
		}
	}
}
```

返回结果如下：

```json
{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.2876821,
        "hits": [
            {
                "_index": "user",
                "_type": "_doc",
                "_id": "1001",
                "_score": 0.2876821,
                "_source": {
                    "name": "小米",
                    "sex": "男的",
                    "tel": "1111"
                }
            }
        ]
    }
}
```

查询电话

```json
# GET http://127.0.0.1:9200/user/_search
{
	"query":{
		"match":{
			"tel":"11"
		}
	}
}
```

返回结果如下：

```json
{
    "error": {
        "root_cause": [
            {
                "type": "query_shard_exception",
                "reason": "failed to create query: Cannot search on field [tel] since it is not indexed.",
                "index_uuid": "yuEPksOOTFmrTFV5ujv2DQ",
                "index": "user"
            }
        ],
        "type": "search_phase_execution_exception",
        "reason": "all shards failed",
        "phase": "query",
        "grouped": true,
        "failed_shards": [
            {
                "shard": 0,
                "index": "user",
                "node": "c5CrEH_uSNy3E9JW7SMBEw",
                "reason": {
                    "type": "query_shard_exception",
                    "reason": "failed to create query: Cannot search on field [tel] since it is not indexed.",
                    "index_uuid": "yuEPksOOTFmrTFV5ujv2DQ",
                    "index": "user",
                    "caused_by": {
                        "type": "illegal_argument_exception",
                        "reason": "Cannot search on field [tel] since it is not indexed."
                    }
                }
            }
        ]
    },
    "status": 400
}
```

报错只因创建映射时"tel"的"index"为false。

#### 三、JavaAPI-环境准备

新建Maven工程

添加依赖 创建POM文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>yooomecloud</artifactId>
        <groupId>com.yooome.springcloud</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>cloud-config-elasticsearch-9200</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <!-- SpringBoot整合Web组件 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--elasticsearch需要依赖-->
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>7.7.0</version>
            <exclusions>
                <exclusion>
                    <groupId>org.elasticsearch</groupId>
                    <artifactId>elasticsearch</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.elasticsearch.client</groupId>
                    <artifactId>elasticsearch-rest-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>7.7.0</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>7.7.0</version>
        </dependency>
    </dependencies>
</project>
```

#### 3.1 JavaAPI-索引-创建

```java
@GetMapping("/elsa/search/{id}")
    public String GetElasticsearch(@PathVariable("id") Integer id) throws IOException {

        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost", 9200, "http")));
        System.out.println(client);
        // 创建索引 - 请求对象
        CreateIndexRequest createIndexRequest = new CreateIndexRequest("user3");
        // 发送请求，获取响应
        CreateIndexResponse response = client.indices().create(createIndexRequest, RequestOptions.DEFAULT);
        boolean acknowledged = response.isAcknowledged();
        System.out.println("操作状态：" + acknowledged);
        // 关闭客户端连接
        client.close();

        return "get elastic search dsgs " + "server port " + serverPort + " ack " + id + " acknowledged " + acknowledged;
    }
```

#### 3.2 JavaAPI-索引-查询 & 删除

##### 3.2.1 查询

```java
 @GetMapping("elas/search/user")
    public String GetIndexResponse() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));

        GetIndexRequest request = new GetIndexRequest("user");
        GetIndexResponse response = client.indices().get(request, RequestOptions.DEFAULT);

        return "aliases: " + response.getAliases() + " mappings: " + response.getMappings() + " settings: " + response.getSettings();
    }
```

##### 3.2.2 删除

```java
@GetMapping("/elas/delete")
    public String DeleteIndexResponse() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        DeleteIndexRequest request = new DeleteIndexRequest("user3");
        AcknowledgedResponse response = client.indices().delete(request, RequestOptions.DEFAULT);

        return "response: " + response.isAcknowledged();
    }
```

#### 3.3 JavaAPI-文档-新增 & 修改

##### 3.3.1 重构

上下文由于频繁使用以下连接 Elasticsearch 和 关闭它的代码，于是个人对它进行重构。

##### 3.3.2 新增

```java
@GetMapping("/elas/create/doc")
    public String CreateElasticsearchIndex() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        IndexRequest request = new IndexRequest();
        request.index("user4").id("1000");
        //创建数据对象
        User user = new User();
        user.setAge(18);
        user.setName("zhangsan");
        user.setSex("男");

        ObjectMapper objectMapper = new ObjectMapper();
        String s = objectMapper.writeValueAsString(user);
        // 添加文档数据，数据格式为 JSON 格式
        request.source(s, XContentType.JSON);
        IndexResponse response = client.index(request, RequestOptions.DEFAULT);
        System.out.println("_index:" + response.getIndex());
        System.out.println("_id:" + response.getId());
        System.out.println("_result:" + response.getResult());

        return "_index:" + response.getIndex() + "_id:" + response.getId() + "_result:" + response.getResult();
    }
```

##### 3.3.3 修改

```java
@GetMapping("/elas/update/doc")
    public String UpdateElasDoc() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 修改文档 - 请求对象
        UpdateRequest request = new UpdateRequest();
        // 修改配置参数
        request.index("user4").id("1000");
        // 设置请求体，对数据修改
        request.doc(XContentType.JSON, "sex", "女");
        UpdateResponse response = client.update(request, RequestOptions.DEFAULT);

        return "index " + response.getIndex() + " _id " + response.getId() + " _result " + response.getResult();
    }
```

#### 3.4 JavaAPI-文档-查询 & 删除

##### 3.4.1 查询

```java
 @GetMapping("/elas/search/doc")
    public String SearchElasDoc() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建请求对象
        GetRequest request = new GetRequest();
        // 客户端发送请求，获取响应对象
        request.index("user4").id("1000");
        GetResponse response = client.get(request, RequestOptions.DEFAULT);

        client.close();
        return "_index = " + response.getIndex() + " _type = " + response.getType() + " _id = " + response.getId() + " source = " + response.getSource();
    }
```

##### 3.4.2 删除

```java
@GetMapping("/elas/delete/doc")
    public String DeleteElasDoc() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建请求对象
        DeleteRequest request = new DeleteRequest();

        // 客户端发送请求，获取响应对象
        request.index("user4").id("1000");
        DeleteResponse delete = client.delete(request, RequestOptions.DEFAULT);

        client.close();
        return delete.toString();
    }
```

#### 3.5 JavaAPI-文档-批量新增 & 批量删除

##### 3.5.1 批量新增

```java
  @GetMapping("/elas/batch/add")
    public String BatchAddElasDoc() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建添加对象
        BulkRequest request = new BulkRequest();
        request.add(new IndexRequest().index("user5").id("1001").source(XContentType.JSON,"name","zhangsan"));
        request.add(new IndexRequest().index("user5").id("1002").source(XContentType.JSON,"name","lisi"));
        request.add(new IndexRequest().index("user5").id("1003").source(XContentType.JSON,"name","wangwu"));
        //客户端发送请求，获取响应对象
        BulkResponse responses = client.bulk(request, RequestOptions.DEFAULT);
        return "took = " + responses.getTook() +" item = " + responses.getItems();
    }
```

##### 3.5.2 批量删除

```java
 @GetMapping("/elas/batch/delete")
    public String BatchDeleteElasDoc() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建添加对象
        BulkRequest request = new BulkRequest();
        request.add(new DeleteRequest().index("user5").id("1001"));
        request.add(new DeleteRequest().index("user5").id("1002"));
        request.add(new DeleteRequest().index("user5").id("1003"));
        //客户端发送请求，获取响应对象
        BulkResponse responses = client.bulk(request, RequestOptions.DEFAULT);
        return "took = " + responses.getTook() +" item = " + responses.getItems();
    }
```

#### 3.6 JavaAPI-文档-高级查询-全量查询

先批量增加数据

```java
@GetMapping("/elas/batch/add")
    public String BatchAddElasDoc() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建添加对象
        BulkRequest request = new BulkRequest();
        request.add(new IndexRequest().index("user5").id("1001").source(XContentType.JSON,"name","zhangsan"));
        request.add(new IndexRequest().index("user5").id("1002").source(XContentType.JSON,"name","lisi"));
        request.add(new IndexRequest().index("user5").id("1003").source(XContentType.JSON,"name","wangwu"));
        request.add(new IndexRequest().index("user5").id("1004").source(XContentType.JSON,"name","zhaoliu"));
        request.add(new IndexRequest().index("user5").id("1005").source(XContentType.JSON,"name","hanqi"));
        //客户端发送请求，获取响应对象
        BulkResponse responses = client.bulk(request, RequestOptions.DEFAULT);
        return "took = " + responses.getTook() +" item = " + responses.getItems();
    }
```

**查询所有索引数据**

```java
 @GetMapping("/elas/batch/search/index")
    public String BatchSearchElasIndex() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建添加对象
        // 创建搜索请求对象
        SearchRequest request = new SearchRequest();
        request.indices("user5");
        // 构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        // 查询所有数据
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took:" + response.getTook());
        System.out.println("timeout:" + response.isTimedOut());
        System.out.println("total:" + hits.getTotalHits());
        System.out.println("MaxScore:" + hits.getMaxScore());
        System.out.println("hits========>>");
        for (SearchHit hit : hits) {
            //输出每条查询的结果信息
            System.out.println(hit.getSourceAsString());
        }
        System.out.println("<<========");
        return "took = " + response.getTook() + " time out = "+ response.isTimedOut() + " total hits = " + hits.getTotalHits() + " max score = " + hits.getMaxScore();
    }
```

#### 3.7 JavaAPI-文档-高级查询-分页查询 & 条件查询 & 查询排序

##### 3.7.1 条件查询

```java
@GetMapping("/elas/best/search/index")
    public String BestSearchElasIndex() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建搜索请求对象
        SearchRequest request = new SearchRequest();
        request.indices("user5");
        // 构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.termQuery("name", "zhangsan"));
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took:" + response.getTook());
        System.out.println("timeout:" + response.isTimedOut());
        System.out.println("total:" + hits.getTotalHits());
        System.out.println("MaxScore:" + hits.getMaxScore());
        System.out.println("hits========>>");
        for (SearchHit hit : hits) {
            //输出每条查询的结果信息
            System.out.println(hit.getSourceAsString());
        }
        System.out.println("<<========");

        return "took = " + response.getTook() + " time out = "+ response.isTimedOut() + " total hits = " + hits.getTotalHits() + " max score = " + hits.getMaxScore();
    }
```

##### 3.7.2 分页查询

```java
@GetMapping("/elas/page/search/index")
    public String PageSearchElasIndex() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建搜索请求对象
        SearchRequest request = new SearchRequest();
        request.indices("user5");
        // 构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        // 分页查询
        // 当前页其实索引(第一条数据的顺序号)， from
        sourceBuilder.from(0);

        // 每页显示多少条 size
        sourceBuilder.size(2);

        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took:" + response.getTook());
        System.out.println("timeout:" + response.isTimedOut());
        System.out.println("total:" + hits.getTotalHits());
        System.out.println("MaxScore:" + hits.getMaxScore());
        System.out.println("hits========>>");
        for (SearchHit hit : hits) {
            //输出每条查询的结果信息
            System.out.println(hit.getSourceAsString());
        }
        System.out.println("<<========");

        return "took = " + response.getTook() + " time out = "+ response.isTimedOut() + " total hits = " + hits.getTotalHits() + " max score = " + hits.getMaxScore();
    }
```

##### 3.7.3 查询排序

```java
@GetMapping("/elas/sort/order")
    public String SortOrderElas() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建搜索请求对象
        SearchRequest request = new SearchRequest();
        request.indices("user");

        // 构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        // 排序
        sourceBuilder.sort("age", SortOrder.ASC);
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took:" + response.getTook());
        System.out.println("timeout:" + response.isTimedOut());
        System.out.println("total:" + hits.getTotalHits());
        System.out.println("MaxScore:" + hits.getMaxScore());
        System.out.println("hits========>>");
        for (SearchHit hit : hits) {
            //输出每条查询的结果信息
            System.out.println(hit.getSourceAsString());
        }
        System.out.println("<<========");

        return "";
    }
```

#### 3.8 JavaAPI-文档-高级查询-组合查询 & 范围查询

##### 3.8.1 组合查询

```java
@GetMapping("/elas/match/query")
    public void matchQuery() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建搜索请求对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        // 构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        // 必须包含
        boolQueryBuilder.must(QueryBuilders.matchQuery("age", "20"));
        // 一定不含
        boolQueryBuilder.mustNot(QueryBuilders.matchQuery("name", "zhangsan"));
        // 可能包含
        boolQueryBuilder.should(QueryBuilders.matchQuery("sex", "男"));
        sourceBuilder.query(boolQueryBuilder);
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took:" + response.getTook());
        System.out.println("timeout:" + response.isTimedOut());
        System.out.println("total:" + hits.getTotalHits());
        System.out.println("MaxScore:" + hits.getMaxScore());
        System.out.println("hits========>>");
        for (SearchHit hit : hits) {
            //输出每条查询的结果信息
            System.out.println(hit.getSourceAsString());
        }
        System.out.println("<<========");
    }
```

##### 3.8.2 范围查询

```java
@GetMapping("/elas/range/query")
    public void RangeQuery() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));

        // 创建搜索请求对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        // 构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("age");
        // 大于等于
        //rangeQuery.gte("30");
        // 小于等于
        rangeQuery.lte("40");
        sourceBuilder.query(rangeQuery);
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took:" + response.getTook());
        System.out.println("timeout:" + response.isTimedOut());
        System.out.println("total:" + hits.getTotalHits());
        System.out.println("MaxScore:" + hits.getMaxScore());
        System.out.println("hits========>>");
        for (SearchHit hit : hits) {
            //输出每条查询的结果信息
            System.out.println(hit.getSourceAsString());
        }
        System.out.println("<<========");
    }
```

#### 3.9 JavaAPI-文档-高级查询-模糊查询 & 高亮查询

##### 3.9.1 模糊查询

```java
 @GetMapping("/elas/fuzzy/query")
    public void fuzzyQuery() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 创建搜索请求对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        // 构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.fuzzyQuery("name", "wangw").fuzziness(Fuzziness.ONE));
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took:" + response.getTook());
        System.out.println("timeout:" + response.isTimedOut());
        System.out.println("total:" + hits.getTotalHits());
        System.out.println("MaxScore:" + hits.getMaxScore());
        System.out.println("hits========>>");
        for (SearchHit hit : hits) {
            //输出每条查询的结果信息
            System.out.println(hit.getSourceAsString());
        }
        System.out.println("<<========");
    }
```

##### 3.9.2 高亮查询

```java
@GetMapping("/elas/high/query")
    public void HighLightSearch() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 高亮查询
        SearchRequest request = new SearchRequest().indices("user");
        //2.创建查询请求体构建器
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //构建查询方式：高亮查询
        TermsQueryBuilder termsQueryBuilder =
                QueryBuilders.termsQuery("name","lisi");
        //设置查询方式
        sourceBuilder.query(termsQueryBuilder);
        //构建高亮字段
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.preTags("<font color='red'>");//设置标签前缀
        highlightBuilder.postTags("</font>");//设置标签后缀
        highlightBuilder.field("name");//设置高亮字段
        //设置高亮构建对象
        sourceBuilder.highlighter(highlightBuilder);
        //设置请求体
        request.source(sourceBuilder);
        //3.客户端发送请求，获取响应对象
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //4.打印响应结果
        SearchHits hits = response.getHits();
        System.out.println("took::"+response.getTook());
        System.out.println("time_out::"+response.isTimedOut());
        System.out.println("total::"+hits.getTotalHits());
        System.out.println("max_score::"+hits.getMaxScore());
        System.out.println("hits::::>>");
        for (SearchHit hit : hits) {
            String sourceAsString = hit.getSourceAsString();
            System.out.println(sourceAsString);
            //打印高亮结果
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            System.out.println(highlightFields);
        }
        System.out.println("<<::::");
    }
```

#### 3.10 JavaAPI-文档-高级查询-最大值查询 & 分组查询

##### 3.10.1 最大值查询

```java
 @GetMapping("/elas/maxvalue/query")
    public void MaxValueSearch() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 高亮查询
        SearchRequest request = new SearchRequest().indices("user");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.aggregation(AggregationBuilders.max("maxAge").field("age"));
        //设置请求体
        request.source(sourceBuilder);
        //3.客户端发送请求，获取响应对象
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //4.打印响应结果
        SearchHits hits = response.getHits();
        System.out.println(response);
        for (SearchHit hit : hits) {
            System.out.println(hit);
        }
    }
```

##### 3.10.2 分组查询

```java
@GetMapping("/elas/group/query")
    public void GroupSearch() throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200, "http")));
        // 高亮查询
        SearchRequest request = new SearchRequest().indices("user");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.aggregation(AggregationBuilders.terms("age_groupby").field("age"));
        //设置请求体
        request.source(sourceBuilder);
        //3.客户端发送请求，获取响应对象
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //4.打印响应结果
        SearchHits hits = response.getHits();
        System.out.println(response);
        for (SearchHit hit : hits) {
            System.out.println(hit);
        }
    }
```

### 四、Elasticsearch环境

#### 4.1 -环境-简介

**单机 & 集群**

单台 Elasticsearch 服务器提供服务，往往都有最大的负载能力，超过这个阈值，服务器性能就会大大降低甚至不可用，所以生产环境中，一般都是运行在指定服务器集群中。除了负载能力，单点服务器也存在其他问题：

单台机器存储容量有限

单服务器容易出现单点故障，无法实现高可用

单服务的并发处理能力有限

配置服务器集群时，集群中节点数量没有限制，大于等于 2 个节点就可以看做是集群了。一般出于高性能及高可用方面来考虑集群中节点数量都是 3 个以上

总之，集群能提高性能，增加容错。

**集群 Cluster**

**一个集群就是由一个或多个服务器节点组织在一起，共同持有整个的数据，并一起提供索引和搜索功能。**一个 Elasticsearch 集群有一个唯一的名字标识，这个名字默认就是”elasticsearch”。这个名字是重要的，因为一个节点只能通过指定某个集群的名字，来加入这个集群。

**节点 Node**

集群中包含很多服务器， 一个节点就是其中的一个服务器。 作为集群的一部分，它存储数据，参与集群的索引和搜索功能。

一个节点也是由一个名字来标识的，默认情况下，这个名字是一个随机的漫威漫画角色的名字，这个名字会在启动的时候赋予节点。这个名字对于管理工作来说挺重要的，因为在这个管理过程中，你会去确定网络中的哪些服务器对应于 Elasticsearch 集群中的哪些节点。

一个节点可以通过配置集群名称的方式来加入一个指定的集群。默认情况下，每个节点都会被安排加入到一个叫做“elasticsearch”的集群中，这意味着，如果你在你的网络中启动了若干个节点，并假定它们能够相互发现彼此，它们将会自动地形成并加入到一个叫做“elasticsearch”的集群中。

在一个集群里，只要你想，可以拥有任意多个节点。而且，如果当前你的网络中没有运行任何 Elasticsearch 节点，这时启动一个节点，会默认创建并加入一个叫做“elasticsearch”的集群。

#### 4.2 -环境-Windows集群部署

##### 4.2.1 部署集群

1、创建 elasticsearch-cluster 文件夹

创建 elasticsearch-7.8.0-cluster 文件夹，在内部复制三个 elasticsearch 服务。

![img](./images/6d6022b6d30a9e2c1a4e18092d5f130f.png)

2、修改集群文件目录中每个节点的 config/elasticsearch.yml 配置文件

**node-1001 节点**

```yml
#节点 1 的配置信息：
#集群名称，节点之间要保持一致
cluster.name: my-elasticsearch
#节点名称，集群内要唯一
node.name: node-1001
node.master: true
node.data: true
#ip 地址
network.host: localhost
#http 端口
http.port: 1001
#tcp 监听端口
transport.tcp.port: 9301
#discovery.seed_hosts: ["localhost:9301", "localhost:9302","localhost:9303"]
#discovery.zen.fd.ping_timeout: 1m
#discovery.zen.fd.ping_retries: 5
#集群内的可以被选为主节点的节点列表
#cluster.initial_master_nodes: ["node-1", "node-2","node-3"]
#跨域配置
#action.destructive_requires_name: true
http.cors.enabled: true
http.cors.allow-origin: "*"
```

**node-1002 节点**

```yml
#节点 2 的配置信息：
#集群名称，节点之间要保持一致
cluster.name: my-elasticsearch
#节点名称，集群内要唯一
node.name: node-1002
node.master: true
node.data: true
#ip 地址
network.host: localhost
#http 端口
http.port: 1002
#tcp 监听端口
transport.tcp.port: 9302
discovery.seed_hosts: ["localhost:9301"]
discovery.zen.fd.ping_timeout: 1m
discovery.zen.fd.ping_retries: 5
#集群内的可以被选为主节点的节点列表
#cluster.initial_master_nodes: ["node-1", "node-2","node-3"]
#跨域配置
#action.destructive_requires_name: true
http.cors.enabled: true
http.cors.allow-origin: "*"

```

**node-1003 节点**

```yml
#节点 3 的配置信息：
#集群名称，节点之间要保持一致
cluster.name: my-elasticsearch
#节点名称，集群内要唯一
node.name: node-1003
node.master: true
node.data: true
#ip 地址
network.host: localhost
#http 端口
http.port: 1003
#tcp 监听端口
transport.tcp.port: 9303
#候选主节点的地址，在开启服务后可以被选为主节点
discovery.seed_hosts: ["localhost:9301", "localhost:9302"]
discovery.zen.fd.ping_timeout: 1m
discovery.zen.fd.ping_retries: 5
#集群内的可以被选为主节点的节点列表
#cluster.initial_master_nodes: ["node-1", "node-2","node-3"]
#跨域配置
#action.destructive_requires_name: true
http.cors.enabled: true
http.cors.allow-origin: "*"

```

3、如果有必要，删除每个节点中的 data 目录中所有内容 。

#### 4.3 启动集群

分别依次双击执行节点的bin/elasticsearch.bat, 启动节点服务器（可以编写一个脚本启动），启动后，会自动加入指定名称的集群。

#### 4.4 测试集群

1、用Postman，查看集群状态

GET http://127.0.0.1:1001/_cluster/health
GET http://127.0.0.1:1002/_cluster/health
GET http://127.0.0.1:1003/_cluster/health
返回结果皆为如下：

```json
{
    "cluster_name": "my-application",
    "status": "green",
    "timed_out": false,
    "number_of_nodes": 3,
    "number_of_data_nodes": 3,
    "active_primary_shards": 0,
    "active_shards": 0,
    "relocating_shards": 0,
    "initializing_shards": 0,
    "unassigned_shards": 0,
    "delayed_unassigned_shards": 0,
    "number_of_pending_tasks": 0,
    "number_of_in_flight_fetch": 0,
    "task_max_waiting_in_queue_millis": 0,
    "active_shards_percent_as_number": 100.0
}

```

**status字段**指示着当前集群在总体上是否工作正常。它的三种颜色含义如下：

1. green：所有的主分片和副本分片都正常运行。
2. yellow：所有的主分片都正常运行，但不是所有的副本分片都正常运行。
3. red：有主分片没能正常运行。

2、用Postman，在一节点增加索引，另一节点获取索引

向集群中的node-1001节点增加索引：

```java
#PUT http://127.0.0.1:1001/user
```

返回结果：

```json
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "user"
}
```

向集群中的node-1003节点获取索引：

```java
#GET http://127.0.0.1:1003/user
```

返回结果：

```json
{
    "user": {
        "aliases": {},
        "mappings": {},
        "settings": {
            "index": {
                "creation_date": "1617993035885",
                "number_of_shards": "1",
                "number_of_replicas": "1",
                "uuid": "XJKERwQlSJ6aUxZEN2EV0w",
                "version": {
                    "created": "7080099"
                },
                "provided_name": "user"
            }
        }
    }
}
```

如果在1003创建索引，同样在1001也能获取索引信息，这就是集群能力。

### 五、环境-Linux单节点部署

#### 5.1 软件安装

1、下载软件

[下载Linux版的Elasticsearch](https://www.elastic.co/cn/downloads/past-releases/elasticsearch-7-8-0)

2、解压软件

```bash
# 解压缩
tar -zxvf elasticsearch-7.8.0-linux-x86_64.tar.gz -C /opt/module
# 改名
mv elasticsearch-7.8.0 es
```

3、创建用户

因为安全问题， Elasticsearch 不允许 root 用户直接运行，所以要创建新用户，在 root 用户中创建新用户。

```bash
useradd es #新增 es 用户
passwd es #为 es 用户设置密码
userdel -r es #如果错了，可以删除再加
chown -R es:es /opt/module/es #文件夹所有者
```

4、修改配置文件

修改/opt/module/es/config/elasticsearch.yml文件。

```bash
# 加入如下配置
cluster.name: elasticsearch
node.name: node-1
network.host: 0.0.0.0
http.port: 9200
cluster.initial_master_nodes: ["node-1"]
```

修改/etc/security/limits.conf

```bash
# 在文件末尾中增加下面内容
# 每个进程可以打开的文件数的限制
es soft nofile 65536
es hard nofile 65536
```

修改/etc/security/limits.d/20-nproc.conf

```bash
# 在文件末尾中增加下面内容
# 每个进程可以打开的文件数的限制
es soft nofile 65536
es hard nofile 65536
# 操作系统级别对每个用户创建的进程数的限制
* hard nproc 4096
# 注： * 带表 Linux 所有用户名称
```

修改/etc/sysctl.conf

```bash
# 在文件中增加下面内容
# 一个进程可以拥有的 VMA(虚拟内存区域)的数量,默认值为 65536
vm.max_map_count=655360
```

重新加载

```bash
sysctl -p
```

#### 5.2 启动软件

使用 ES 用户启动

```bash
cd /opt/module/es/
#启动
bin/elasticsearch
#后台启动
bin/elasticsearch -d  
```

启动时，会动态生成文件，如果文件所属用户不匹配，会发生错误，需要重新进行修改用户和用户组

![img](./images/75e24cd1fbc430577681bb4f8a0d3e2b.png)

关闭防火墙

```bash
#暂时关闭防火墙
systemctl stop firewalld
#永久关闭防火墙
systemctl enable firewalld.service #打开防火墙永久性生效，重启后不会复原
systemctl disable firewalld.service #关闭防火墙，永久性生效，重启后不会复原
```

#### 5.3 测试软件

浏览器中输入地址： http://linux1:9200/

![img](./images/477065a2cb499a32871549c21e2fc487.png)

### 六、环境-Linux集群部署

#### 6.1 软件安装

一、下载软件

[下载Linux版的Elasticsearch](https://www.elastic.co/cn/downloads/past-releases/elasticsearch-7-8-0)

二、解压软件

```bash
# 解压缩
tar -zxvf elasticsearch-7.8.0-linux-x86_64.tar.gz -C /opt/module
# 改名
mv elasticsearch-7.8.0 es-cluster
```

将软件分发到其他节点： linux2, linux3

三、创建用户

因为安全问题， Elasticsearch 不允许 root 用户直接运行，所以要创建新用户，在 root 用户中创建新用户。

```bash
useradd es #新增 es 用户
passwd es #为 es 用户设置密码
userdel -r es #如果错了，可以删除再加
chown -R es:es /opt/module/es #文件夹所有者
```

四、修改配置文件

修改/opt/module/es/config/elasticsearch.yml 文件，分发文件。

```yml
# 加入如下配置
#集群名称
cluster.name: cluster-es
#节点名称， 每个节点的名称不能重复
node.name: node-1
#ip 地址， 每个节点的地址不能重复
network.host: linux1
#是不是有资格主节点
node.master: true
node.data: true
http.port: 9200
# head 插件需要这打开这两个配置
http.cors.allow-origin: "*"
http.cors.enabled: true
http.max_content_length: 200mb
#es7.x 之后新增的配置，初始化一个新的集群时需要此配置来选举 master
cluster.initial_master_nodes: ["node-1"]
#es7.x 之后新增的配置，节点发现
discovery.seed_hosts: ["linux1:9300","linux2:9300","linux3:9300"]
gateway.recover_after_nodes: 2
network.tcp.keep_alive: true
network.tcp.no_delay: true
transport.tcp.compress: true
#集群内同时启动的数据任务个数，默认是 2 个
cluster.routing.allocation.cluster_concurrent_rebalance: 16
#添加或删除节点及负载均衡时并发恢复的线程个数，默认 4 个
cluster.routing.allocation.node_concurrent_recoveries: 16
#初始化数据恢复时，并发恢复线程的个数，默认 4 个
cluster.routing.allocation.node_initial_primaries_recoveries: 16
```

修改/etc/security/limits.conf ，分发文件

```yml
# 在文件末尾中增加下面内容
es soft nofile 65536
es hard nofile 65536
```

修改/etc/security/limits.d/20-nproc.conf，分发文件

```yml
# 在文件末尾中增加下面内容
es soft nofile 65536
es hard nofile 65536
\* hard nproc 4096
\# 注： * 带表 Linux 所有用户名称
```

修改/etc/sysctl.conf

```yml
# 在文件中增加下面内容
vm.max_map_count=655360
```

重新加载

```yml
sysctl -p
```

#### 6.2 启动软件

分别在不同节点上启动 ES 软件

```yml
cd /opt/module/es-cluster
#启动
bin/elasticsearch
#后台启动
bin/elasticsearch -d
```

#### 6.3 测试集群

![img](./images/0412e37cb5249d1ff0e813ee87f49a50.png)

### 七、Elasticsearch进阶

#### 7.1 进阶-核心概念

![img](./images/146a779da01f53e7f7a8d53132d3c7cf.png)

#### 7.2 索引Index

一个索引就是一个拥有几分相似特征的文档的集合。比如说，你可以有一个客户数据的索引，另一个产品目录的索引，还有一个订单数据的索引。一个索引由一个名字来标识（必须全部是小写字母），并且当我们要对这个索引中的文档进行索引、搜索、更新和删除（CRUD）的时候，都要使用到这个名字。在一个集群中，可以定义任意多的索引。

能搜索的数据必须索引，这样的好处是可以提高查询速度，比如：新华字典前面的目录就是索引的意思，目录可以提高查询速度。

Elasticsearch 索引的精髓：一切设计都是为了提高搜索的性能。

#### 7.3 类型Type

在一个索引中，你可以定义一种或多种类型。

一个类型是你的索引的一个逻辑上的分类/分区，其语义完全由你来定。通常，会为具
有一组共同字段的文档定义一个类型。不同的版本，类型发生了不同的变化。

![6](./images/6.png)

#### 7.4 文档Document

一个文档是一个可被索引的基础信息单元，也就是一条数据。

比如：你可以拥有某一个客户的文档，某一个产品的一个文档，当然，也可以拥有某个订单的一个文档。文档以 JSON（Javascript Object Notation）格式来表示，而 JSON 是一个到处存在的互联网数据交互格式。

在一个 index/type 里面，你可以存储任意多的文档。

#### 7.5 字段Field

相当于是数据表的字段，对文档数据根据不同属性进行的分类标识。

映射Mapping
mapping 是处理数据的方式和规则方面做一些限制，如：某个字段的数据类型、默认值、分析器、是否被索引等等。这些都是映射里面可以设置的，其它就是处理 ES 里面数据的一些使用规则设置也叫做映射，按着最优规则处理数据对性能提高很大，因此才需要建立映射，并且需要思考如何建立映射才能对性能更好。

#### 7.6 分片Shards

一个索引可以存储超出单个节点硬件限制的大量数据。比如，一个具有 10 亿文档数据
的索引占据 1TB 的磁盘空间，而任一节点都可能没有这样大的磁盘空间。 或者单个节点处理搜索请求，响应太慢。为了解决这个问题，**Elasticsearch 提供了将索引划分成多份的能力，每一份就称之为分片。**当你创建一个索引的时候，你可以指定你想要的分片的数量。每个分片本身也是一个功能完善并且独立的“索引”，这个“索引”可以被放置到集群中的任何节点上。

1. 分片很重要，主要有两方面的原因：

2. 允许你水平分割 / 扩展你的内容容量。

允许你在分片之上进行分布式的、并行的操作，进而提高性能/吞吐量。
至于一个分片怎样分布，它的文档怎样聚合和搜索请求，是完全由 Elasticsearch 管理的，对于作为用户的你来说，这些都是透明的，无需过分关心。

被混淆的概念是，一个 Lucene 索引 我们在 Elasticsearch 称作 分片 。 一个Elasticsearch 索引 是分片的集合。 当 Elasticsearch 在索引中搜索的时候， 他发送查询到每一个属于索引的分片（Lucene 索引），然后合并每个分片的结果到一个全局的结果集。

Lucene 是 Apache 软件基金会 Jakarta 项目组的一个子项目，提供了一个简单却强大的应用程式接口，能够做全文索引和搜寻。在 Java 开发环境里 Lucene 是一个成熟的免费开源工具。就其本身而言， Lucene 是当前以及最近几年最受欢迎的免费 Java 信息检索程序库。但 Lucene 只是一个提供全文搜索功能类库的核心工具包，而真正使用它还需要一个完善的服务框架搭建起来进行应用。

目前市面上流行的搜索引擎软件，主流的就两款： Elasticsearch 和 Solr,这两款都是基于 Lucene 搭建的，可以独立部署启动的搜索引擎服务软件。由于内核相同，所以两者除了服务器安装、部署、管理、集群以外，对于数据的操作 修改、添加、保存、查询等等都十分类似。

#### 7.7 副本Replicas

在一个网络 / 云的环境里，失败随时都可能发生，在某个分片/节点不知怎么的就处于
离线状态，或者由于任何原因消失了，这种情况下，有一个故障转移机制是非常有用并且是强烈推荐的。为此目的， Elasticsearch 允许你创建分片的一份或多份拷贝，这些拷贝叫做复制分片(副本)。

复制分片之所以重要，有两个主要原因：

在分片/节点失败的情况下，提供了高可用性。因为这个原因，注意到复制分片从不与原/主要（original/primary）分片置于同一节点上是非常重要的。
扩展你的搜索量/吞吐量，因为搜索可以在所有的副本上并行运行。
总之，每个索引可以被分成多个分片。一个索引也可以被复制 0 次（意思是没有复制）或多次。一旦复制了，每个索引就有了主分片（作为复制源的原来的分片）和复制分片（主分片的拷贝）之别。

分片和复制的数量可以在索引创建的时候指定。在索引创建之后，你可以在任何时候动态地改变复制的数量，但是你事后不能改变分片的数量。

默认情况下，Elasticsearch 中的每个索引被分片 1 个主分片和 1 个复制，这意味着，如果你的集群中至少有两个节点，你的索引将会有 1 个主分片和另外 1 个复制分片（1 个完全拷贝），这样的话每个索引总共就有 2 个分片， 我们需要根据索引需要确定分片个数。分配

#### 7.8 Allocation

将分片分配给某个节点的过程，包括分配主分片或者副本。如果是副本，还包含从主分片复制数据的过程。这个过程是由 master 节点完成的。

### 八、进阶-系统架构-简介

![img](./images/e4d13427545dc174eb9ccface85c1f0c.png)



一个运行中的 Elasticsearch 实例称为一个节点，而集群是由一个或者多个拥有相同
cluster.name 配置的节点组成， 它们共同承担数据和负载的压力。当有节点加入集群中或者从集群中移除节点时，集群将会重新平均分布所有的数据。

当一个节点被选举成为主节点时， 它将负责管理集群范围内的所有变更，例如增加、
删除索引，或者增加、删除节点等。 而主节点并不需要涉及到文档级别的变更和搜索等操作，所以当集群只拥有一个主节点的情况下，即使流量的增加它也不会成为瓶颈。 任何节点都可以成为主节点。我们的示例集群就只有一个节点，所以它同时也成为了主节点。

作为用户，我们可以将请求发送到集群中的任何节点 ，包括主节点。 每个节点都知道
任意文档所处的位置，并且能够将我们的请求直接转发到存储我们所需文档的节点。 无论我们将请求发送到哪个节点，它都能负责从各个包含我们所需文档的节点收集回数据，并将最终结果返回給客户端。 Elasticsearch 对这一切的管理都是透明的。

#### 8.1 进阶-单节点集群

我们在包含一个空节点的集群内创建名为users 的索引，为了演示目的，我们将分配 3 个朱分片和一份副本（每个主分片拥有一个副本分片）。

```json
#PUT http://127.0.0.1:1001/users
{
    "settings" : {
        "number_of_shards" : 3,
        "number_of_replicas" : 1
    }
}
```

![img](./images/54ee6753be248cc7d345b38a0eae7d96.png)

通过 elasticsearch-head 插件（一个Chrome插件）查看集群情况 。

![img](./images/e8b15d0b243d486e91f478a220da63bf.png)

- 集群健康值:yellow( 3 of 6 )：表示当前集群的全部主分片都正常运行，但是副本分片没有全部处在正常状态。

![img](./images/489b6de480112879a00067b793bde685.png)

- 3 个主分片正常。

![img](./images/3ce9a78d26ee762f0a7a8abf7817a58e.png)

- 3 个副本分片都是 Unassigned，它们都没有被分配到任何节点。 在同 一个节点上既保存原始数据又保存副本是没有意义的，因为一旦失去了那个节点，我们也将丢失该节点 上的所有副本数据。

当前集群是正常运行的，但存在丢失数据的风险。

#### 8.2 elasticsearch-head chrome插件安装

[插件获取网址](https://github.com/mobz/elasticsearch-head)，下载压缩包，解压后将内容放入自定义命名为elasticsearch-head文件夹。

接着点击Chrome右上角选项->工具->管理扩展（或则地址栏输入chrome://extensions/），选择打开“开发者模式”，让后点击“加载已解压得扩展程序”，选择elasticsearch-head/_site，即可完成chrome插件安装。

![img](./images/bf76cb1bfbdf07555918d9055817ab44.png)

#### 8.3 进阶-故障转移

当集群中只有一个节点在运行时，意味着会有一个单点故障问题——没有冗余。 幸运的是，我们只需再启动一个节点即可防止数据丢失。当你在同一台机器上启动了第二个节点时，只要它和第一个节点有同样的 cluster.name 配置，它就会自动发现集群并加入到其中。但是在不同机器上启动节点的时候，为了加入到同一集群，你需要配置一个可连接到的单播主机列表。之所以配置为使用单播发现，以防止节点无意中加入集群。只有在同一台机器上
运行的节点才会自动组成集群。

如果启动了第二个节点，集群将会拥有两个节点 : 所有主分片和副本分片都已被分配 。

![img](./images/bf76cb1bfbdf07555918d9055817ab44.png)

通过 elasticsearch-head 插件查看集群情况

![img](./images/18db400822b83e727d6206f486b7b2ea.png)

- 集群健康值:green( 3 of 6 )：表示所有 6 个分片（包括 3 个主分片和 3 个副本分片）都在正常运行。

![img](./images/e485d8263a4aa3a94af0be951bd5a241.png)

3 个主分片正常。

![img](./images/e485d8263a4aa3a94af0be951bd5a241.png)

- 第二个节点加入到集群后， 3 个副本分片将会分配到这个节点上——每 个主分片对应一个副本分片。这意味着当集群内任何一个节点出现问题时，我们的数据都完好无损。所 有新近被索引的文档都将会保存在主分片上，然后被并行的复制到对应的副本分片上。这就保证了我们 既可以从主分片又可以从副本分片上获得文档。















