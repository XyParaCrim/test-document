# 关于Logstash的配置

## 配置文件结构

```
config
├── pipeline-1
│   ├── elasticsearch-template.json
│   ├── jdbc-elasticsearch.conf
│   ├── jdbc-input.sql
├── pipeline-2
├── pipeline-3
├── pipeline-4
├── pipeline-5
├── pipeline-6
└── pipelines.yml
```

- **pipeline** - 定义同一进程下的不同管道
- **elasticsearch-template.json** - 定义es索引模块
- **jdbc-elasticsearch.conf** - 定义如何从数据库输入到es输出
- **jdbc-input.sql** - logstash的jdbc输入插件所需查询语句
- **pipelines.yml** - 通过这配置文件运行多个pipeline

## 配置多管道

通过配置`pipelines.yml`文件运行多管道，文件结构类似于

```yaml
- pipeline.id: question
  # 设置该管道处理线程数，默认和cpu核数一样
  pipeline.workers: 1
  # 下一个事件等待上一个未完成事件的时间
  pipeline.batch.delay: 5000
  # 事件队列存储的方式：persisted(磁盘),mmory(内存)
  queue.type: persisted
  # 管道配置文件(相对目录是相对于运行文件(~/bin/logstash))
  path.config: "../config/question/jdbc-elasticsearch.conf"
```

## logstash配置文件

logstash配置文件默认为名`jdbc-elasticsearch.conf`，表示从数据库到es的传输

logstash配置文件的大致结构

```
# 输入插件
input {
  ...
}

# 过滤插件
filter {
  ...
}

# 输出插件
output {
  ...
}
```

即`input plugin` ——> `filter plugin` ——> `output plugin`

因此，主要用到的插件有：
- **input plugins**
    - jdbc
- **filter**
    - jdbc stream
    - jdbc static
- **output**
    - elasticsearch
    
### jdbc输入插件

所有的jdbc插件配置大致相同，例如：
```logstash
jdbc {
    jdbc_connection_string => "${JDBC_CONNECT_STRING}"
    jdbc_user => "${JDBC_USER}"
    jdbc_password => "${JDBC_PASSWORD}"
    jdbc_driver_library => "${LOGSTASH_HOME}/lib/mysql-connector-java/mysql-connector-java-5.1.48-bin.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_paging_enabled => true
    jdbc_page_size => "50000"

    statement_filepath => "${LOGSTASH_CONFIG}/activity-entry/jdbc-input.sql"

    schedule => "0 */5 * * * *"

    # 同步点文件，这个文件记录了上次的同步点，重启时会读取这个文件，这个文件可以手动修改
    last_run_metadata_path => "${LOGSTASH_LAST_RUN}/activity-entry"
  }
```

以上配置使用到几个环境变量

- `JDBC_CONNECT_STRING` - 数据库连接url
- `JDBC_USER` - 数据库连接用户名
- `JDBC_PASSWORD` - 数据库连接连接密码
- `LOGSTASH_HOME` - logstash安装根目录
- `LOGSTASH_CONFIG` - logstash配置文件目录
- `LOGSTASH_LAST_RUN` - 上一次查询数据库时间戳文件存放文件位置

**在服务器里，以上环境变量是需要自行配置的。**

主要配置选项，其他配置选项参考[这里](https://www.elastic.co/guide/en/logstash/7.3/plugins-inputs-jdbc.html#plugins-inputs-jdbc-options)

| 配置名 | 描述    |
| --------------------- | ---------------------- |
| jdbc_driver_library   | jdbc-driver路径          |
| statement_filepath    | 查询语句文件          |
| schedule     | 执行计划(0 */5 * * * * - 00和30分执行)           |
| last_run_metadata_path | 上一次执行的时间戳文件位置          |
| clean_run | 是否删除之前执行时间(默认为false,若清除将执行全量同步)|

### elasticsearch输出插件

所有的elasticsearch插件配置大致相同，例如：

```logstash
elasticsearch {
      hosts => "${ELASTICSEARCH_HOST}"
      index => "mysql-%{[@metadata][type]}-%{[@metadata][index]}"
      document_id => "%{id}"
      pool_max => 100
      pool_max_per_route => 10
      template => "${LOGSTASH_CONFIG}/activity-entry/elasticsearch-template.json"
      template_name => "mysql-social-activity-entry"
      template_overwrite => true
    }
```

以上配置使用到几个环境变量

- `ELASTICSEARCH_HOST` - 输出es服务器地址
- `LOGSTASH_CONFIG` - logstash配置文件目录

主要配置选项，其他配置选项参考[这里](https://www.elastic.co/guide/en/logstash/7.3/plugins-outputs-elasticsearch.html#plugins-outputs-elasticsearch-options)

| 配置名 | 描述    |
| --------------------- | ---------------------- |
| hosts   | es服务器的uri("127.0.0.1"或者["127.0.0.1:9200","127.0.0.2:9200"])|
| index    | 索引名  |
| document_id     | 文档id |
| template | 使用的索引模块配置文件，发送到es服务器(不配置则不发送) |
| template_name | 模板名 |
| template_overwrite | 是否覆盖es服务器的模块 |

特别地
- 索引名可以根据事件里数据或者时间修改,例如(`logstash-%{+YYYY.MM.dd}`或者`logstash-%{[type]}`)
- 若索引名在es服务器未匹配到template，则会使用logstash默认的template
- @metadata这个字段的数据不会发送到es

### 比较重要的filter插件

以下两个插件主要处理这么一个问题：因为从数据库输入至logstash的数据是扁平化的，而大多数情况下es需要的数据是json这样嵌套的，所有需要
查询其他表数据进行嵌套，存到指定字段。

#### jdbc_streaming

jdbc_streaming过滤插件会执行sql查询，将结果存储到指定字段。例如：

```logstash
jdbc_streaming {
    jdbc_driver_library => "${LOGSTASH_HOME}/lib/mysql-connector-java/mysql-connector-java-5.1.48-bin.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "${JDBC_CONNECT_STRING}"
    jdbc_user => "${JDBC_USER}"
    jdbc_password => "${JDBC_PASSWORD}"
    statement => "select wm_object_category.category_id from wm_object_category where wm_object_category.object_id = :object_id and wm_object_category.object_type = :object_type and wm_object_category.domain_id = :domain_id"
    parameters => {
      "object_id" => "id"
      "object_type" => "type"
      "domain_id" => "domain_id"
    }
    target => "category"
  }
```

许多配置与jdbc输入插件一样

| 配置名 | 描述    |
| --------------------- | ---------------------- |
| parameters   | sql语句参数 |
| target    | 存放指定字段  |

上面的例子意思是用输入插件数据的id、type、domain构成的sql查询，将结构存到category字段下

主要配置选项，其他配置选项参考[这里](https://www.elastic.co/guide/en/logstash/7.3/plugins-filters-jdbc_streaming.html)

#### jdbc_static

jdbc_streaming过滤插件存在问题：假如有1000条输入，那么就会再执行1000次数据库查询的操作，即每条查询一次。
jdbc_static过滤插件会将数据库表存到本地磁盘中，直接本地查询，将结果存放到指定字段，这样就不需要频繁查询数据库。

```logstash
jdbc_static {
    loaders => [
      {
        # loader的id
        id => "wm_object_category"
        # 查询语句
        query => "select category_id, object_id, object_type, domain_id  from wm_object_category"
        # 本地表面
        local_table => "category"
      }
    ]

    # 本地表结构定义
    local_db_objects => [
      {
        # 对应local_table
        name => "category"
        # 主键字段
        index_columns => ["category_id", "object_id", "object_type"]
        # 字段类型
        columns => [
          ["category_id", "bigint"],
          ["object_id", "varchar(50)"],
          ["object_type", "varchar(30)"],
          ["domain_id", "varchar(50)"]
        ]
      }
    ]

    # 本地表查询
    local_lookups => [
      {
        # 查询id
        id => "local-category"
        # 查询语句
        query => "select category.category_id from category where category.object_id = :object_id and category.object_type = :object_type and category.domain_id = :domain_id"
        # 查询参数 -> query
        parameters => {
          object_id => "[object_id]"
          object_type => "[object_type]"
          domain_id => "[domain_id]"
        }
        # 结果存放字段
        target => "category"
      }
    ]

    # 每小时00分执行一次
    loader_schedule => "0 * * * *"
    jdbc_connection_string => "${JDBC_CONNECT_STRING}"
    jdbc_user => "${JDBC_USER}"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_driver_library => "${LOGSTASH_HOME}/lib/mysql-connector-java/mysql-connector-java-5.1.48-bin.jar"
    jdbc_password => "${JDBC_PASSWORD}"

    # 本地数据库存放位置
    staging_directory => "${LOGSTASH_HOME}/data/tmp/logstash/jdbc_static/import_data"
  }
```

特别地

- logstash采用Apache derby db
- 本地数据库表的类型一定要严格对应 - [数据类型](https://db.apache.org/derby/docs/10.14/ref/crefsqlj31068.html)
- jdbc_static会将数据库查询的数据缓存至`staging_directory`

  例如上面例子的情况，文件会长成这种样子
  
  ```
  000,001,类型1,002
  003,004,类型2,005
  005,005,类型2,005
  .................
  ```
但是如果查询出来的字段为空，则会长成这种样子

```
000,001,类型1,
,,,
,005,类型2,005
```

若为字段数据类型，那么logstash在`local_lookups`阶段读取数据文件将会报错

主要配置选项，其他配置选项参考[这里](https://www.elastic.co/guide/en/logstash/7.3/plugins-filters-jdbc_static.html#plugins-filters-jdbc_static)


#### 其他插件

其他过滤插件较为简单，可以查询[这里](https://www.elastic.co/guide/en/logstash/7.3/filter-plugins.html)



    