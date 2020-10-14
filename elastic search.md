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

### 三.ES的基本操作

##### 3.1 ES的结构

######   3.1.1 索引 

>   ES的服务中可以创建多个索引
>
> 每一个索引默认被分成5片存储
>
> 每一个分片都会存在至少要一个备份分片
>
> 备份分片默认不会帮助检索数据，当ES压力特别大的时候，备份分片才会帮助检索数据。
>
> 备份的分片必须放在不同的服务器中

![1602555240247](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602555240247.png)

######   3.1.2 Type

> 根据版本不同，类型的创建也不同

![1602555380099](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602555380099.png)

######   3.1.3 文档DOC

>   一个type下，有多个文档，一个文档类似于mysql表中的一行数据。

![1602555512315](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602555512315.png)

######   3.1.4 Field 属性

> 一个文档可以包含多个属性，一个属性类似于mysql表中的一个列。

![1602555551319](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602555551319.png)

##### 3.2操作ES的RESTFul语法

> GET请求：
>
>   http://ip:port/index  查询索引信息
>
>   http://ip:port/index/type/doc_id 查询指定的文档信息，传doc的id
>
> POST请求：
>
>   http://ip:port/index/type/_search  查询，可以再请求体中添加json字符串来代表查询条件
>
>   http://ip:port/index/type/doc_id/_update 修改文档，在请求体中指定json字符串代表修改的具体信息
>
> PUT请求：
>
>   http://ip:port/index 创建一个索引，需要在请求体重指定索引的信息
>
>   http://ip:port/index/type/_mappings 代表创建索引时，指定索引文档存储的属性的信息
>
> DELETE请求：
>
>    http://ip:port/index ： 全部删除
>
>    http://ip:port/index/type/doc_id 删除指定的文档

##### 3.3索引的操作

######   3.3.1创建一个索引

  ```json
#创建一个索引
PUT /person
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}
  ```

######  3.3.2 查看索引信息

![1602556129232](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1602556129232.png)



```json
GET /person
```

###### 3.3.3删除索引

```json
#删除索引
DELETE /person
```

##### 3.4 ES中Field的类型

> **string**
>
>  text:  被用于全文检索，将当前Field进行分词
>
>  keyword： 当前Field不会被分词
>
> **Numeric datatypes**
>
> long
>
> integer
>
> short
>
> byte
>
> double
>
> float
>
> half_float:  精度比float小一半
>
> scaled_float： 根据一个long和scaled来表达一个浮点型， long:345, scaled 100 -> 3.45
>
> **Date datatype**
>
> date：通过format属性指定具体日期格式
>
> **Boolean datatype**
>
> boolean
>
> **Binary datatype**
>
> binary 暂时支持base64编码的字符串
>
> **Range datatypes**：无需指定具体内容，只需要存储一个范围
>
> integer_range
>
> float_range
>
> long_range
>
> double_range
>
> date_range
>
> **Geo datatypes**
>
> Geo-point datatype ：经纬度

##### 3.5 创建索引并指定数据结构

```json
#创建索引，并指定数据结构
PUT /book #索引名称
{
  "settings": {
    #备份数
    "number_of_replicas": 1,
     #分片数
    "number_of_shards": 5
  },
  #指定数据结构
  "mappings": {
    #指定type
    "novel": {
      #指定Field
      "properties": {
    	#Field属性名
        "name": {
    	  #指定数据类型
          "type": "text",
          #指定分词器
          "analyzer": "ik_max_word",
          #指定当前的Field可以作为查询的条件
          "index": true,
          #是否需要额外存储
          "store": false
        },
        "author": {
          "type": "keyword"
        },
        "count": {
          "type": "long"
        },
        "on-sale": {
          "type": "date",
          #指定日期格式
          "format": "yyyy-MM-dd"
        },
        "description": {
          "type": "text",
          "analyzer": "ik_max_word"
        }
      }
    }
  }
}
```

##### 3.6文档的操作

> 文档在ES服务中的唯一标识, _index, _type, _id三个内容为符合，锁定一个文档。

###### 3.6.1添加文档

```json
#自动生成id
POST /book/novel
{
  "name": "万族之劫",
  "author": "老鹰捉小鸡",
  "count": "5000000",
  "on-sale": "2020-01-01",
  "description": "热血，从头干到尾"
}
#指定id
POST /book/novel/1
{
  "name": "玩家凶猛",
  "author": "黑灯夏火",
  "count": "2000000",
  "on-sale": "2020-01-01",
  "description": "搞笑，异能，乱斗"
}

```



###### 3.6.2修改文档

```json
#指定id，如果id已经存在会直接覆盖原有数据
POST /book/novel/1
{
  "name": "玩家凶猛",
  "author": "黑灯夏火",
  "count": "2000000",
  "on-sale": "2020-01-01",
  "description": "搞笑，异能，乱斗"
}
```

```json
#推荐使用此方法进行修改，写了哪些字段就会更新哪些字段，不写不更新
POST /book/novel/1/_update
{
  "doc": {
    "count": "3000000",
    "on-sale": "2020-02-01"
  }
}
```

###### 3.6.3删除文档

```json
#根据id删除doc
DELETE /book/novel/yBHpIHUBGbVumt5jno75
```

### 四.JAVA操作ES

##### 4.1Java连接es

> 创建maven工程

> 导入依赖

```xml
<!-- 1. elasticsearch -->
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>6.5.4</version>
</dependency>
<!-- 2. elasticsearch的高级api-->
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>6.5.4</version>
</dependency>
<!-- 3. 测试用junit -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<!-- 4. lombok插件 -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.22</version>
</dependency>
```

> java连接es

```java
  public static RestHighLevelClient getClient() {
    //创建HttpHosts
    HttpHost httpHost = new HttpHost("10.77.100.38", 9200);

    //创建 RestClientBuilder
    RestClientBuilder restClientBuilder = RestClient.builder(httpHost);

    //创建RestHighLevelClient
    RestHighLevelClient restHighLevelClient = new RestHighLevelClient(restClientBuilder);

    //return RestHighLevelClient
    return restHighLevelClient;
  }
```

##### 4.2 Java操作索引

######  4.2.1 新建索引和数据结构

```java
package com.zfc.test1;

import com.zfc.utils.ESClient;
import java.io.IOException;
import org.elasticsearch.action.admin.indices.create.CreateIndexRequest;
import org.elasticsearch.action.admin.indices.create.CreateIndexResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.settings.Settings.Builder;
import org.elasticsearch.common.xcontent.XContentBuilder;
import org.elasticsearch.common.xcontent.json.JsonXContent;
import org.junit.Test;

/**
 * @author ：zhangfc
 * @date ：Created in 2020/10/13 17:26
 * @description：
 */
public class Demo2 {


  String index = "person";

  String type = "man";
  
  RestHighLevelClient client = ESClient.getClient();

  @Test
  public void createIndex() throws IOException {
    //1.准备关于索引的settings
    Builder settings = Settings.builder()
        .put("number_of_shards", 5)
        .put("number_of_replicas", 1);
    //2准备关于索引的结构 mappings
    XContentBuilder mappings = JsonXContent.contentBuilder()
        .startObject()
          .startObject("properties")
            .startObject("name")
              .field("type", "text")
            .endObject()
            .startObject("age")
             .field("type", "integer")
            .endObject()
            .startObject("birth")
             .field("type", "date")
             .field("format", "yyyy-MM-dd")
             .endObject()
          .endObject()
        .endObject();
    //3.封装settings和mappings到一个request对象中
    CreateIndexRequest request = new CreateIndexRequest(index).settings(settings)
        .mapping(type, mappings);
    //4.通过client对象去连接ES并执行创建索引
    CreateIndexResponse createIndexResponse = client.indices()
        .create(request, RequestOptions.DEFAULT);
    System.out.println("resp" + createIndexResponse.toString());
  }
}
```

###### 4.2.2 检查index是否存在

```java
@Test
public void exists() throws IOException {
    //准备request对象
    GetIndexRequest request = new GetIndexRequest();
    request.indices(index);

    //通过client操作
    boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);

    //输出结果
    System.out.println("是否存在===" + exists);
}
```

###### 4.2.3删除索引

```java
@Test
public void deleteIndex() throws IOException {
    //创建request
    DeleteIndexRequest deleteRequest = new DeleteIndexRequest();
    deleteRequest.indices(index);
    //通过client操作
    AcknowledgedResponse delete = client.indices().delete(deleteRequest, 		                             RequestOptions.DEFAULT);

    // 输出结果
    System.out.println("结果===" + delete.isAcknowledged());
}
```

##### 4.3 Java操作文档

###### 4.3.1 添加文档

