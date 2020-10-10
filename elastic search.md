### 一. ElasticSearch 介绍

##### 1.1介绍

> ES是一个使用java语言并基于lucence编写的搜索引擎框架
>
> 提供了分布式的全文搜索功能。
>
> 提供了一个统一的基于Restful风格的接口，官方客户端也对多重语言都提供了相应的api。
>
> Lucene: 本身就是一个搜索引擎的底层。
>
> 分布式： ES主要是为了突出横向扩展能力。
>
> 全文检索：将一段词语进行分词，并且将分出的单个词语统一的放到一个分词库中，在搜索时，根据关键字，在分词库中检索，找到匹配的内容。（倒排索引）
>
> Restful风格的接口：操作ES很简单，只需要发送一个HTTP请求，并且根据请求方式不同，携带参数不同，自行相应的功能。
>
> 应用广泛：github，wiki， gold man。

##### 1.2 ES和Slor的区别

> 1.Slor 在查询没有改动的数据时，速度相对ES更快一些，但是如果数据会改动，速度就会低很多，ES的查询效率基本不变
>
> 2.Slor 大家基于需要依赖Zookeeper来帮助管理，ES本身就支持集群的搭建，不用第三方的介入
>
> 3.Slor国内文档不多。ES出现后，文档非常健全。
>
> 4.ES对现在的云计算和大数据支持的特别好。

##### 1.3 倒排索引

 1.将存放的数据以一定的方式进行分词，并且将分词的内容存放到一个单独的分词库中

 2.当用户去查询数据时，会将用户的查询关键字进行分词。

 3.query 根据输入的关键字，去分词库中检索内容，得到数据的id标识

 4.fetch：根据去分词库中检索到的id，直接去存放数据的位置拉取指定的数据。

![1602309802479](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602309802479.png)

### 二.安装ElasticSearch

##### 2.1安装ES&Kibana

```yml
version: "3.1"
services:
  elasticsearch:
    image: daocloud.io/library/elasticsearch:6.5.4
    restart: always
    container_name: elasticsearch
    ports:
      - 9200:9200
  kibana:
    image: daocloud.io/library/kibana:6.5.4
    restart: always
    container_name: kibana
    ports:
      - 5601:5601
    #连接elasticsearch
    environment:
      - elasticsearch_url=http://10.77.100.38:9200
    #监听容器的名称
    depends_on:
      - elasticsearch
```

**elasticsearch启动不起来，错误码78**

> 在/etc/sysctl.conf最后一行加上 vm.max_map_count=262144
>
> 或者：
>
> sysctl -w vm.max_map_count=262144

##### 2.2安装ik分词器

> ik分词器：
>
> 版本要对应es的版本
>
> 下载地址：https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.5.4/elasticsearch-analysis-ik-6.5.4.zip

进入elasticsearch容器内部下载分词器

> docker exec -it 容器id bash
>
> cd  bin/
>
> ./elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.5.4/elasticsearch-analysis-ik-6.5.4.zip
>
> 重启elasticsearch容器，以让分词器生效

##### 2.3分词器使用

```json
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "张富成"
}
#结果
{
  "tokens" : [
    {
      "token" : "张",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "富",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "成",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "CN_CHAR",
      "position" : 2
    }
  ]
}
```

