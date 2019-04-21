---
layout: post
title: 'Elasticsearch使用笔记'
date: 2019-04-19
author: yamadadada
color: green
cover: 'https://images.contentstack.io/v3/assets/bltefdd0b53724fa2ce/blt280217a63b82a734/5bbdaacf63ed239936a7dd56/elastic-logo.svg'
tags: elasticsearch spring
typora-root-url: ..
---

# 1 参考资料

- [spring-data-elasticsearch的github主页](https://github.com/spring-projects/spring-data-elasticsearch)
- [官方文档](https://docs.spring.io/spring-data/elasticsearch/docs/3.0.7.RELEASE/reference/html/)
- [中文翻译](https://www.jianshu.com/p/27e1d583aafb)
- [入门参考1](https://www.cnblogs.com/guozp/archive/2018/04/02/8686904.html)
- [入门参考2](https://segmentfault.com/a/1190000015568618)

# 2 配置

- spring boot 2.1.4
- sping-data-elasticsearch 3.1.6
- elasticsearch 6.2.2

## Maven依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

maven父依赖关系：

```
org.springframework.boot » spring-boot-starter: 2.1.4
org.springframework.data » spring-data-elasticsearch: 3.1.6
```

版本关系：

| spring data elasticsearch | elasticsearch |
| :-----------------------: | :-----------: |
|           3.2.x           |     6.5.0     |
|           3.1.x           |     6.2.2     |
|           3.0.x           |     5.5.0     |
|           2.1.x           |     2.4.0     |
|           2.0.x           |     2.2.0     |
|           1.3.x           |     1.5.2     |

### application.yml

```yaml
spring:
  data:
    elasticsearch:
      cluster-name: five-cluster
      properties:
        transport:
          tcp:
            connect_timeout: 120s
      repositories:
        enabled: true
      cluster-nodes: 127.0.0.1:9300
```

# 3 Elasticsearch安装与配置

### elasticsearch

配置：在config/elasticsearch.yml下添加下面配置

```yaml
http.cors.enabled: true
http.cors.allow-origin: "*"
```

启动：在bin目录下输入命令：elasticsearch

### elasticsearch-head插件

安装：[github地址](https://github.com/mobz/elasticsearch-head)

首次启动时：

- 预先安装node环境，检查版本
- npm install
- npm run start

### 集群

配置：针对不同的结点，在config/elasticsearch.yml下添加下面配置

```yml
cluster.name: five-cluster
node.name: master
node.master: true（默认为false）
network.host: 127.0.0.1
http.port: 9200
discovery.zen.ping.unicast.hosts: ["127.0.0.1"]（非master结点时用于寻找集群）
```

不同的结点配置不同的端口

### 添加index

PUT /five，body加上下面的配置：

```json
{
    "settings":{
           "number_of_shards":1,
           "number_of_replicas":0,
           "index":{
                  "analysis":{
                         "analyzer":{
                                "default":{
                                       "tokenizer":"ik_max_word"
                                }
                         }
                  }
           }
    }
}
```

### 添加mapping映射

PUT /five/_mapping/item?pretty，body加上（item为类型名）

```json
{
    "properties": {
    	"itemId": {
    		"type": "long"
    	},
    	"orderId": {
    		"type": "long"
    	},
    	"itemName": {
    		"type": "text",
            "analyzer": "ik_max_word"
    	},
    	"number": {
    		"type": "integer"
    	},
    	"price": {
    		"type": "double"
    	}
    }
}
```

### 清空数据

POST /five/_delete_by_query

```json
{
  "query": {
    "match_all": {}
  }
}
```

# 4 同步mysql数据到elasticsearch

参考资料： [Mysql数据同步Elasticsearch方案总结](https://my.oschina.net/u/4000872/blog/2252620)

## 同步的三种方案

### 1.logstash-input-jdbc

logstash官方插件,集成在logstash中,下载logstash即可,通过配置文件实现mysql与elasticsearch数据同步

优点

- 能实现mysql数据全量和增量的数据同步,且能实现定时同步.
- 版本更新迭代快,相对稳定.
- 作为ES固有插件logstash一部分,易用

缺点

- 不能实现同步删除操作,MySQL数据删除后Elasticsearch中数据仍存在.
- 同步最短时间差为一分钟,一分钟数据同步一次,无法做到实时同步.

### 2.go-mysql-elasticsearch

go-mysql-elasticsearch 是国内作者开发的一款插件

优点

- 能实现mysql数据增加,删除,修改操作的实时数据同步

缺点

- 无法实现数据全量同步Elasticsearch
- 仍处理开发、相对不稳定阶段

### 3. elasticsearch-jdbc

目前最新的版本是2.3.4，支持的ElasticSearch的版本为2.3.4, 未实践

优点

- 能实现mysql数据全量和增量的数据同步.

缺点

- 目前最新的版本是2.3.4，支持的ElasticSearch的版本为2.3.4
- 不能实现同步删除操作,MySQL数据删除后Elasticsearch中数据仍存在.

## 这里使用logstash-input-jdbc

- 首先下载logstash，这里使用logstash6.4.2（使用6.2.2会报错，原因未知）
- 进入bin目录安装logstash-input-jdbc插件

```
cd /logstash-6.4.2/bin
./logstash-plugin install logstash-input-jdbc
```

### 配置文件

在config文件夹中新建jdbc.sql，配置模版如下

```
# 输入部分
input {
  stdin {}
  jdbc {
    # mysql数据库驱动
    jdbc_driver_library => "/usr/local/logstash-6.4.2/config/mysql-connector-java-5.1.30.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    # mysql数据库链接，数据库名
    jdbc_connection_string => "jdbc:mysql://localhost:3306/octopus"
    # mysql数据库用户名，密码
    jdbc_user => "root"
    jdbc_password => "12345678"
    # 设置监听间隔  各字段含义（分、时、天、月、年），全部为*默认含义为每分钟更新一次
    schedule => "* * * * *"
    # 分页
    jdbc_paging_enabled => "true"
    # 分页大小
    jdbc_page_size => "50000"
    # sql语句执行文件，也可直接使用 statement => 'select * from t_employee'
    statement_filepath => "/usr/local/logstash-6.4.2/config/jdbc.sql"
    # elasticsearch索引类型名
    type => "t_employee"
  }
}

# 过滤部分(不是必须项）
filter {
    json {
        source => "message"
        remove_field => ["message"]
    }
}

# 输出部分
output {
    elasticsearch {
        # elasticsearch索引名
        index => "octopus"
        # 使用input中的type作为elasticsearch索引下的类型名
        document_type => "%{type}"   # <- use the type from each input
        # elasticsearch的ip和端口号
        hosts => "localhost:9200"
        # 同步mysql中数据id作为elasticsearch中文档id
        document_id => "%{id}"
    }
    stdout {
        codec => json_lines
    }
}

# 注: 使用时请去掉此文件中的注释，不然会报错
```

配置文件实际例子：

```
input {
  stdin {}
  jdbc {
    jdbc_driver_library => "config\mysql-connector-java-5.1.43.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/five"
    jdbc_user => "root"
    jdbc_password => "jerry75911"
    schedule => "* * * * *"
    jdbc_paging_enabled => "true"
    jdbc_page_size => "50000"
	statement => "select item_id as itemId, order_id as orderId, item_name as itemName, number, price from item"
    type => "item"
    lowercase_column_names => "false"
  }
}
output {
    elasticsearch {
        index => "five"
        document_type => "%{type}"
        hosts => "localhost:9200"
        document_id => "%{item_id}"
    }
    stdout {
        codec => json_lines
    }
}
```

logstash默认会把别名的大写转为大写，所以把lowercase_column_names设置为false

### 运行

```
cd logstash-6.4.2
# 检查配置文件语法是否正确
bin/logstash -f config/jdbc.conf --config.test_and_exit
# 启动
bin/logstash -f config/jdbc.conf --config.reload.automatic
```

# 5 es中文分词工具elasticsearch-analysis-ik

[github主页](https://github.com/medcl/elasticsearch-analysis-ik)

### 安装

这里使用6.2.2版本（与es对应）

在es的bin目录下执行以下命令

```
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.2.2/elasticsearch-analysis-ik-6.2.2.zip
```

# 6 Java代码例子

### 实体类

```java
@Data
@Document(indexName = "five", type = "item")
public class EsItem {

    @Id
    private Long itemId;

    private Long orderId;

    @Field(analyzer = "ik_max_word", type = FieldType.Text)
    private String itemName;

    private Integer number;

    private BigDecimal price;
}
```

### Repository

```java
public interface EsItemRepository extends ElasticsearchRepository<EsItem, Long> {

    List<EsItem> findByItemName(String name);

    Page<EsItem> findByItemName(String name, Pageable pageable);
}
```

### Service

```java
@Service
public class SearchServiceImpl implements SearchService {

    @Autowired
    private EsItemRepository itemRepository;

    @Override
    public List<EsItem> findItemByItemName(String name) {
        return itemRepository.findByItemName(name);
    }

    @Override
    public Page<EsItem> findItemByItemName(String name, Pageable pageable) {
        return itemRepository.findByItemName(name, pageable);
    }
}
```

### Controller

```java
@Controller
@RequestMapping("/search")
public class SearchController {

    @Autowired
    private SearchService searchService;

    @GetMapping("")
    @ResponseBody
    public Object search(@RequestParam(value = "name") String name) {
        List<EsItem> itemList = searchService.findItemByItemName(name);
        return ResultVOUtil.success(itemList);
    }
}
```

