---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: juejin
highlight: juejin
---



#### 前言
最近笔者接到了公司内部的ES团队的通知，需要将自己应用的数据从ES5搬迁到ES7。我在维护一个基于Golang语言开发的项目，遇到了一些关于ES 的Golang Client [https://github.com/olivere/elastic](https://github.com/olivere/elastic) 的问题，特此写这篇博客记录一下。
#### 具体任务

在从ES5迁移到ES7的过程中，由于ES的换代的一些新特性和改变，需要首先进行具体任务的分解，主要包括
* **数据格式升级**：从ES6开始，ES便不推荐在同一个Index下存在多个type的数据。因此需要将此前在同一个Index下数据拆散到不同的Index之中就可以解决问题（官方原因是之前index下不同实体太多，切分布稀疏不均匀，严重干扰 Lucene 压缩文档的能力，具体可参考 [Removal of mapping types](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/removal-of-types.html)）
* **ES client（olivere/elastic）的升级**：由于底层已经切换到了最新的ES7，并且在我司内部的ES7 将强制启动token认证，同时由于采用了 ES的client [olivere/elastic](https://github.com/olivere/elastic) 所以需要选择最合适的方案进行代码的迁移。

#### "No Elasticsearch Node Available"
 首先为了改最少的代码，笔者的第一个想法便是用<u>**olivere**</u>的ES5的client [elastic.v5](https://gopkg.in/olivere/elastic.v5) 直接访问最新的ES Endpoint,结果第一个遇到的问题便是"No Elasticsearch Node Available"。在经过Google一番之后, 从这篇博客中 [程序印象](https://www.do1618.com/archives/1355/no-elasticsearch-node-available/) ，查出初步的主要原因是[Sniffing](https://github.com/olivere/elastic/wiki/Sniffing)的配置造成的。
 
##### Elasticsearch：Sniffing
[Sniffing](https://github.com/olivere/elastic/wiki/Sniffing) 的主要功能是将Elasticsearch的静态节点列表传递给客户端，然后将客户端请求平均分布在Elasticsearch 集群节点之间.如果启用 sniffing，则客户端将开始调用 `_nodes /_all /http` 端点(类似 `http://127.0.0.1:9200/__nodes /_all /http`)，并且响应将是群集中存在的所有节点及其 IP 地址的列表。然后，客户端将更新其连接池以使用所有新节点，并使群集状态与客户端的连接池保持同步。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9deded0a28945e0beb5ecdba0309e88~tplv-k3u1fbpfcp-watermark.image)

如果想要检查自己的ES集群的Sniffing的配置，可以通过访问 

* `curl -XGET 'http://127.0.0.1:9200/_nodes/http?pretty=true`
```
{
  "_nodes" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "cluster_name" : "elasticsearch",
  "nodes" : {
    "v9vBH0xXQ1-GZw-bOfxlFQ" : {
      "name" : "v9vBH0x",
      "transport_address" : "127.0.0.1:9300",
      "host" : "127.0.0.1",
      "ip" : "127.0.0.1",
      "version" : "7.2.1",
      "build_hash" : "fe6cb20",
      "roles" : [
        "master",
        "data",
        "ingest"
      ],
      "attributes" : {
        "ml.max_open_jobs" : "10",
        "ml.enabled" : "true"
      },
      "http" : {
        "bound_address" : [
          "0.0.0.0:9200"
        ],
        "publish_address" : "127.0.0.1:9200", # sniffing mode setting
        "max_content_length_in_bytes" : 104857600
      }
    }
  }
}
```




如果想要在[elastic.v5](https://gopkg.in/olivere/elastic.v5)中禁止sniffing，在进行Client初始化的时候可以通过`SetSniff(false)`来实现。


`client, err := elastic.NewClient(elastic.SetURL(host),
elastic.SetSniff(false))
`

但是在进行了上述的尝试之后，还是遇到了“No Elasticsearch Node Available”这个问题，证明Sniffing并不是链接不上ES7集群这个问题的主要原因。又经过了一番搜索之后， 查到了原因是 elastic.v5驱动的工作方式与 ES7配合的过程中存在某些问题，这个GitHub issue中则提供了进一步的分析。[v5 connect panic: no Elasticsearch node available ](https://github.com/olivere/elastic/issues/387)。那么只能放弃使用elastic.v5的想法，将ES的Client升级到[elastic.v7](https://gopkg.in/olivere/elastic.v7)去。

#### 总结
经过这次踩的坑，个人觉得对ES进行CRUD最好的方式还是用原生的HTTP请求便可，这样子下一次升级的时候只需要将ES的endpoint 切换一下便可，省去了才不同版本之间的Library切换所需要的时间和精力。


#### 参考链接


*  [Sniffing](https://github.com/olivere/elastic/wiki/Sniffing)
*  [“No Elasticsearch Node Available”](https://www.do1618.com/archives/1355/no-elasticsearch-node-available/)
* [https://github.com/olivere/elastic/issues/312](https://github.com/olivere/elastic/issues/312)
* [Elasticsearch：sniffing 的最佳实践：What, when, why, how](https://blog.csdn.net/UbuntuTouch/article/details/107199603)
* [https://github.com/elastic/go-elasticsearch/issues/86](https://github.com/elastic/go-elasticsearch/issues/86)
* [removal-of-types.html](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/removal-of-types.html)