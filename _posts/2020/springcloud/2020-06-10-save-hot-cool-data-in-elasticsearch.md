---
layout: post
title: ElasticSearch怎样保存我们的冷热数据？
category: springcloud
tags: [elasticSearch]
keywords: elasticSearch
excerpt: 使用热数据节点、冷数据节点部署ES集群
lock: noneed
---

## 1、冷热数据分离

Elasticsearch的冷热数据分离，热数据存储到SSD硬盘，冷数据存储到普通SATA硬盘，降低成本，提升ES的热数据的读写性能。

实现方案

- ES节点被分为热节点和冷节点，通过节点属性配置实现，
- 索引通过路由分布策略设置，装热数据的索引的分片被分布到热节点上，冷数据的索引被分布到冷节点上

- 当需要分片数据从热机迁移到冷机上，估计新建一个冷数据的索引做存储冷数据，或者直接迁移到冷数据索引上。

1、配置节点属性

- 热节点: es-node-1、es-node-2 
- 冷节点: es-node-3

```shell
# 修改 es-node-1 和 es-node-2 的elasticsearch.yml，增加属性
node.attr.tag: hot   # 标识为热数据节点

# 修改 es-node-3 的elasticsearch.yml，增加属性
node.attr.tag: cool  # 标识为冷数据节点

node.attr.* # 你可以按业务场景指定需要的值，如
node.attr.hotwarm_type: hot

# 重启es节点
```

![](/assets/images/2020/icoding/elasticsearch/es-hot-cool1.jpg)

2、配置索引路由分布策略

```json
# 索引都分配到热数据节点上
PUT jd_goods/_settings
{
   "settings": {
     "index.routing.allocation.require.tag": "hot"
   }
}

PUT index_customer/_settings
{
   "settings": {
     "index.routing.allocation.require.tag": "hot"
   }
}
```

执行后发现，es会自动将两个索引的分片迁移到es-node-1和es-node-2两个热数据节点上

![](/assets/images/2020/icoding/elasticsearch/es-hot-cool2.jpg)

```shell
# 创建
PUT /index_test
{
    "settings": {
        "index": {
            "number_of_shards": "3", // 3个分片
            "number_of_replicas": "0"，// 每个分片0副本
             "routing.allocation.require.tag": "hot"
        }
    }
}
```

第二种方式，通过模板指定索引的冷热存储

```json
PUT _template/logs_2019-08-template
{
    "index_patterns": "logs_2019-08-*",
    "settings": {
        "index.number_of_replicas": "0",
        "index.routing.allocation.require.hotwarm_type": "warm"
      }
}

PUT _template/logs_2019-10-template
{
    "index_patterns": "logs_2019-10-*",
    "settings": {
    	"index.number_of_replicas": "0",
    	"index.routing.allocation.require.hotwarm_type": "hot"
     }
}
```





## 2、数据从热机迁移到冷机

```json
# 将jd_goods从热机迁移到冷机上
PUT jd_goods/_settings
{
   "settings": {
     "index.routing.allocation.include.tag": "cool"
   }
}

# 查看jd_goods的设置信息
GET jd_goods/_settings
# 返回
{
  "jd_goods" : {
    "settings" : {
      "index" : {
        "routing" : {
          "allocation" : {
            "require" : {
              "tag" : "cool"
            }
          }
        },
        "number_of_shards" : "3",
        "provided_name" : "jd_goods",
        "creation_date" : "1590719598769",
        "number_of_replicas" : "1",
        "uuid" : "UntC75oMRPOAgt6uWrNXfg",
        "version" : {
          "created" : "7060199"
        }
      }
    }
  }
}
```

执行后发现，数据自动迁移到es-node-3上，主分片数据迁移到es-node3上

![](/assets/images/2020/icoding/elasticsearch/es-hot-cool3.jpg)

我们可以基于时间创建索引，做个定时任务，将超过固定时间的索引从热机迁移到冷机，这样就可实现自动迁移冷数据到冷节点上。

## 3、读写分离

目标：使主分片分配在SSD磁盘上，副本落在SATA磁盘上，读取时优先从副本中查询数据，SSD节点只负责写入数据。

实现步骤：

1、修改集群路由分配策略配置

```sh
"allocation.awareness.attributes": "box_type",
"allocation.awareness.force.box_type.values": "hot,cool"
```

参考： https://elasticsearch.cn/article/6127#tip3



## 4、索引生命周期管理

index-lifecycle-managemen，ILM

从ES6.6开始，Elasticsearch提供索引生命周期管理功能，索引生命周期管理可以通过API或者kibana界面配置
，kibana中的索引生命周期管理位置如下图(版本6.8.2)：

![](\assets\images\2022\springcloud\es-kibana.png)

点击创建create policy，进入配置界面，可以看到索引的生命周期被分为：`Hot`, `Warm`, `Cold`, `Delete`四个阶段

- Hot phrase:

  该阶段可以根据索引的文档数，大小，时长决定是否调用rollover API来滚动索引

- Warm phrase

  当一个索引在Hot phrase被roll over后便会进入Warm phrase，进入该阶段的索引会被设置为read-only, 用户可以为这个索引设置要使用的attribute， 如对于冷热分离策略，这里可以选择temperature: warm属性。另外还可以对索引进行forceMerge, shrink等操作，这两个操作具体可以参考官方文档。

  ![](\assets\images\2022\springcloud\es-kibana-phrase.png)

- Cold phrase

  可以设置当索引rollover一段时间后进入cold阶段，这个阶段也可以设置一个属性。从冷热分离架构可以看出冷热属性是具备扩展性的，不仅可以指定hot, warm, 也可以扩展增加hot, warm, cold, freeze等多个冷热属性。如果想使用三层的冷热分离的话这里可以指定为temperature: cold, 此处还支持对索引的freeze操作，详情参考官方文档。

- Delete phrase

  可以设置索引rollover一段时间后进入delete阶段，进入该阶段的索引会自动被删除。

### RollOver的定义

当现有索引被认为太大或太旧时，滚动索引API将别名滚动到新索引。该API接受一个别名和一个条件列表。别名必须只指向一个索引。如果索引满足指定条件，则创建一个新索引，并将别名切换到指向新索引的位置。
ES6.X版本Rollover支持的三种条件是：

1）索引存储的最长时间。如： “max_age”: “7d”,
2）索引支持的最大文档数。如：“max_docs”: 1000,
3）索引最大磁盘空间大小。“max_size”: “5gb”。

下面是ES 6.x版本的Rollover API调用方式：

> 方式一，基于序号的索引管理

1、创建索引，注意索引

```json
PUT /logs-00001
{
    "aliases": 
        "logs_writes": {}
    }
}
```

2、指定RollOver规则

```json
PUT /logs_write/_rollover
{
    "conditions": {
        "max_age": "7d",
        "max_doc": 5,
        // "max_size": "5gb"
        "max_primary_shard_size": "50gb"
    }
}
```

- "max_age": "7d", 最长期限 7d，超过7天，索引会实现滚动。
- "max_docs": 5, 最大文档数 5，超过 5个文档，索引会实现滚动（测试需要，设置的很小）。
- "max_primary_shard_size": "50gb"，主分片最大存储容量 50GB，超过50GB，索引就会滚动。

注意，三个条件是或的关系，满足其中一个，索引就会滚动。

3、批量插入数据

```json
POST logs_write/logs/_bulk
{ "create": {"_id":1}}
{ "text": "111"}
{ "create": {"_id":2}}
{ "text": "222" }
{ "create": {"_id":3}}
{ "text": "333"}
{ "create": {"_id":4}}
{ "text": "4444"}

```

4、重复步骤2

返回结果

```json
{
   "old_index": "logs-000001",
   "new_index": "logs-000002",
   "rolled_over": true,
   "dry_run": false,
   "acknowledged": true,
   "shards_acknowledged": true,
   "conditions": {
     "[max_docs: 2]": true,
    "[max_age: 7d]": false,
    "[max_size: 5gb]": false
  }
}
```

这样以后，后续插入的数据索引就自动变为logs-000002，logs-000003…..logs-00000N

> 方式二，基于时间的索引管理

1、创建基于日期的索引

```json
#PUT /<logs-{now/d}-1> with URI encoding:
PUT /%3Clogs-%7Bnow%2Fd%7D-1%3E 
{
  "aliases": {
    "logs_write": {
        "is_write_index": true
    }
  }
}
```

URI 编码工具：http://t.cn/RAEZiuq

输入：`<logs-{now/d}-1>`
输出：`%3Clogs-%7Bnow%2Fd%7D-1%3E`

![](\assets\images\2022\tool\uri-code-convert.jpg)

2、插入数据

```json
PUT logs_write/_bulk
{"index":{"_id":1}}
{"title":"testing 01"}
{"index":{"_id":2}}
{"title":"testing 02"}
{"index":{"_id":3}}
{"title":"testing 03"}
{"index":{"_id":4}}
{"title":"testing 04"}
{"index":{"_id":5}}
{"title":"testing 05"}
```

3、指定别名索引的RollOver规则

```java
POST /logs_write/_rollover 
{
  "conditions": {
    "max_age": "7d",
    "max_docs": 5,
    "max_primary_shard_size": "50gb"
  }
}
```

返回结果

```java
{
   "old_index": "logs-2018.08.05-1",
   "new_index": "logs-2018.08.05-000002",
   "rolled_over": true,
   "dry_run": false,
   "acknowledged": true,
   "shards_acknowledged": true,
   "conditions": {
     "[max_docs: 1]": true
  }
}
```

如果24小时候后执行，new_index的名字就是+1天后的日期：logs-2018.08.06-000002。

4、插入数据，*在满足滚动条件的前提下滚动索引*

```json
PUT my-alias/_bulk
{"index":{"_id":6}}
{"title":"testing 06"}
```

查询数据，验证滚动是否生效

```json
GET log_write/_search
```

理论上，id=2的文档会放到第二个索引上

### 7.9版本后

7.9+版本后，ES出现了节点角色的概念，配置好节点的冷热温节点角色，索引的分片数据迁移不再需要分配策略

Elasitcsearch 7.9 之前早期版本，需要配置分片分配策略机制。举例如下：

```json
"allocate": 
 {
      "require": 
      {
      "box_type": "warm"
      }
 }
```

7.9+之后新版本相当于我们提前预设定了冷热集群架构的节点不同的角色，后台会帮我们自动迁移。

仅数据层面的节点角色做了如下细分：

- data_hot 热节点
- data_warm 暖节点
- data_cold 冷节点
- data_content 数据内容节点
- data：原有数据节点

三节点的样例冷热集群架构集群节点角色划分如下：

![](\assets\images\2022\springcloud\es-cluster.png)

es 的elasticsearch.yml 配置文件如下：

```yaml
node.roles: [ data_hot, data_content, master, ingest ]
```

我们只需要设置好不同的横向：phrase 和 每个phrase 下的不同的 action（rollover， freeze， number_of_replicas  delete forcemerge 操作）就可以了，其他的我们不需要关注了

![](\assets\images\2022\springcloud\es-policy.png)

后台会有定时任务轮询完整数据各个 phrase 阶段的更迭

### DSL 实战索引生命周期管理

```sh
# step1: 前提：演示刷新需要,默认是10分钟检查
# 检查是否满足 rollover 的周期频率值
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval": "1s"
  }
}

# step2:测试需要，值调的很小
PUT _ilm/policy/my_custom_policy_filter
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "3d",
            "max_docs": 5,
            "max_size": "50gb"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "15s",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "allocate": {
            "require": {
              "box_type": "warm"
            },
            "number_of_replicas": 0
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30s",
        "actions": {
          "allocate": {
            "require": {
              "box_type": "cold"
            }
          },
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "45s",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}

# step3:创建模板，关联配置的ilm_policy
PUT _index_template/timeseries_template
{
  "index_patterns": ["timeseries-*"],                 
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "my_custom_policy_filter",      
      "index.lifecycle.rollover_alias": "timeseries",
      "index.routing.allocation.require.box_type": "hot"
    }
  }
}

# step4:创建起始索引（便于滚动）
PUT timeseries-000001
{
  "aliases": {
    "timeseries": {
      "is_write_index": true
    }
  }
}

# step5：插入数据
PUT timeseries/_bulk
{"index":{"_id":1}}
{"title":"testing 01"}
{"index":{"_id":2}}
{"title":"testing 02"}
{"index":{"_id":3}}
{"title":"testing 03"}
{"index":{"_id":4}}
{"title":"testing 04"}

# step6：临界值（会滚动）
PUT timeseries/_bulk
{"index":{"_id":5}}
{"title":"testing 05"}

# 下一个索引数据写入
PUT timeseries/_bulk
{"index":{"_id":6}}
{"title":"testing 06"}
```

**核心步骤**总结如下：

- 第一步：创建生周期 policy。
- 第二步：创建索引模板，模板中关联 policy 和别名。
- 第三步：创建符合模板的起始索引，并插入数据。
- 第四步: 索引基于配置的 ilm 滚动

### Kibana  界面实战索引生命周期管理

1、创建policy策略

![](\assets\images\2022\springcloud\es-kibana-create-policy.png)

2、关联索引模板

模板要先创建

```sh
PUT _index_template/timebase_template 
{ "index_patterns": ["time_base-*"] }
```

创建起始索引，指定别名和写入

```sh
PUT time_base-000001 { "aliases": { "timebase_alias": { "is_write_index": true } } }
```

![](\assets\images\2022\springcloud\es-kibana-policy-template.png)

小结：

索引生命周期管理需要加强对三个概念的认知：

- 横向——Phrase 阶段：Hot、Warm、Cold、Delete 等对应索引的生、老、病、死。
- 纵向——Actions 阶段：各个阶段的动作。
- 横向纵向整合的Policy：实际是阶段和动作的综合体。

配置完毕Policy，关联好模板 template，整个核心工作就完成了80%。

剩下就是各个阶段 Actions 的调整和优化了。

实战表明：用 DSL 实现ILM 比图形化界面更可控、更便于问题排查。

参考：

https://mp.weixin.qq.com/s/Px5Eo7_aGy8cDLrhToSn_w

https://mp.weixin.qq.com/s/7VQd5sKt_PH56PFnCrUOHQ
