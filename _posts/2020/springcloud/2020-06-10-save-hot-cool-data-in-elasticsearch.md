---
layout: post
title: ElasticSearch怎样保存我们的冷热数据？
category: springcloud
tags: [elasticSearch]
keywords: elasticSearch
excerpt: 使用热数据节点、冷数据节点部署ES集群，索引数据读写分离，Rolover滚动索引，索引生命周期管理
lock: noneed
---

## 1、冷热数据分离

Elasticsearch的冷热数据分离，热数据存储到SSD硬盘，冷数据存储到普通SATA硬盘，降低成本，提升ES的热数据读写性能。

实现方案

- ES节点被分为热节点和冷节点，通过节点属性配置实现
- 索引通过路由分布策略设置，装热数据的索引的分片被分布到热节点上，冷数据的索引被分布到冷节点上

- 当需要分片数据从热机迁移到冷机上，新建一个冷数据的索引做存储冷数据，或者直接迁移到冷数据索引上

### 索引路由分布策略

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
// 索引都分配到热数据节点上
PUT jd_goods/_settings
{
    "index.routing.allocation.require.tag": "hot"
}

PUT index_customer/_settings
{
     "index.routing.allocation.require.tag": "hot"
}
```

执行后发现，es会自动将两个索引的分片迁移到es-node-1和es-node-2两个热数据节点上

![](/assets/images/2020/icoding/elasticsearch/es-hot-cool2.jpg)

```json
// 创建索引
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

### 数据从热机迁移到冷机

只需更新索引配置

```json
// 将jd_goods从热机迁移到冷机上
PUT jd_goods/_settings
{
   "settings": {
     "index.routing.allocation.include.tag": "cool"
   }
}

// 查看jd_goods的设置信息
GET jd_goods/_settings
// 响应
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

我们可以基于时间创建索引，做个定时任务，将超过固定时间的索引修改setting配置，这样就可实现自动迁移冷数据到冷节点上。

### 集群路由分布策略

此策略比索引级路由策略权重高

```json
PUT /_cluster/settings
{
    "routing":{
        "allocation.awareness.attributes": "box_type"
    }
}
```

新建索引时，索引分片及副本只会分配到含有node.attr.box_type属性的节点上。（该值可以为多个，如"box_type,zone"）

```json
PUT /_cluster/settings
{
    "routing":{
        "allocation.awareness.force.box_type.values": "hot,cool"
    }
}
```

强制分离主分片与副本，若只有hot标签的节点，索引只有分片可以写入，副本无法分配；若有hot、cool两种标签节点，相同分片与其副本绝不在相同标签节点上。但是本来ES就是主分片与副本不在同一个物理机上的，没理解这个配置有啥用。

## 2、读写分离

目标：使主分片分配在SSD磁盘上，副本落在SATA磁盘上，读取时优先从副本中查询数据，SSD节点只负责写入数据。

实现步骤：

1、修改集群路由分配策略配置

```sh
PUT /_cluster/settings
{
	"routing":{
		"allocation.awareness.attributes": "box_type",
		"allocation.awareness.force.box_type.values": "hot,cool"	
	}
}
```

2、提前创建索引

```json
PUT log4x_trace_2018_08_11
{
   "settings": {
     "index.routing.allocation.require.box_type": "hot", // 可使索引所有分片都分配在SSD磁盘中(节点属性box_type=hot的节点上)
     "number_of_replicas": 0
	}
}
```

3、修改索引路由分配策略配置，索引创建好后，动态修改索引配置

```json
PUT log4x_trace_2018_08_11/_settings
{
 "index.routing.allocation.require.box_type": null,
 "number_of_replicas": 1
}
```

4、转为冷数据，动态修改索引配置，并取消副本数

```json
PUT log4x_trace_2018_08_11/_settings
{
 "index.routing.allocation.require.box_type": "cool",
 "number_of_replicas": 0
}
```

来源于： https://elasticsearch.cn/article/6127#tip3

## 3、RollOver索引滚动

当现有索引被认为太大或太旧时，滚动索引API将别名滚动到新索引，该API接受一个别名和一个条件列表。如果索引满足指定条件，则创建一个新索引，并将别名切换到指向新索引的位置。通过别名写入文档，会写入到新索引上。通过别名读取文档，会搜索所有别名关联的索引。旧索引的文档支持删除API，那更新API是否也支持，测试一下便知。
Rollover支持的条件是：

| 参数名                  | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| max_age                 | 指定索引存在的最大时间，如10s（秒），1h（小时），7d（天），1m，1y（年） |
| max_docs                | 存放的索引文档数（不包括副本）                               |
| max_size                | 索引所有主分片大小，如50gb                                   |
| max_parimary_shard_size | 索引主分片大小，索引数据一般写入到多个分片，其中一个分片达到设定的大小，则索引触发滚动，官方建议分片大小控制在30gb~50gb |

下面是ES6.x版本的Rollover API实战举例：

### 基于序号的索引管理

1、创建索引

```json
PUT /logs-000001
{
    "aliases":  {// 别名
        "logs_writes": {
    		"is_write_index": true // 表示索引是别名的当前写入索引
    	}
    }    
}
```

2、批量插入数据

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

3、发起RollOver请求，当满足规则，索引就会滚动，创建新的索引，索引别名写入指向新索引，旧索引不再写入文档数据

```json
POST /logs_write/_rollover
{
    "conditions": {
        "max_age": "7d",
        "max_docs": 5,
        "max_size": "5gb"
        //"max_primary_shard_size": "50gb"
    }
}
```

- "max_age": "7d", 最长期限 7d，超过7天，索引会实现滚动。
- "max_docs": 5, 最大文档数 5，超过 5个文档，索引会实现滚动（测试需要，设置的很小）。
- "max_primary_shard_size": "50gb"，主分片最大存储容量 50GB，超过50GB，索引就会滚动。

注意，三个条件是或的关系，满足其中一个，索引就会滚动。

```json
// 滚动成功响应，创建了新的索引
{
   "old_index": "logs-000001",
   "new_index": "logs-000002",
   "rolled_over": true,
   "dry_run": false,
   "acknowledged": true,
   "shards_acknowledged": true,
   "conditions": {
     	"[max_docs: 5]": true,
    	"[max_age: 7d]": false,
    	"[max_size: 5gb]": false
  }
}
```

<mark>注意：</mark>

索引不会自动滚动，每次需要手动触发_rollover请求去检查索引是否满足滚动条件，满足则进行滚动，需要结合定时任务触发rollover请求，实现自动滚动。

### 基于时间的索引管理

1、创建基于日期的索引

```json
// PUT /<logs-{now/d}-1> with URI encoding:
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
PUT /logs_write/_bulk
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

3、发起RollOver请求

```java
POST /logs_write/_rollover 
{
  "conditions": { // 条件
    "max_age": "7d",
    "max_docs": 5,
    "max_primary_shard_size": "50gb"  // 最大分片大小，官方建议30gb-50gb
  }
}
```

满足max_docs=5 ，索引滚动，响应数据如下：

```java
{
   "old_index": "logs-2018.08.05-1",
   "new_index": "logs-2018.08.05-000002",
   "rolled_over": true,
   "dry_run": false,
   "acknowledged": true,
   "shards_acknowledged": true,
   "conditions": {
     "[max_docs: 5]": true
  }
}
```

如果24小时候后执行，new_index的名字就是+1天后的日期：logs-2018.08.06-000002，序号会自增，跟日期没有关系。

4、插入数据，在满足滚动条件的前提下滚动索引

```json
PUT /logs_write/_bulk
{"index":{"_id":6}}
{"title":"testing 06"}
```

查询索引文档数据，验证滚动是否生效

```json
GET /log_write/_search
```

前面执行_rollover时已满足max_docs=5的条件触发了滚动创建新索引`logs-2018.08.05-000002`，所以id=6的文档写入到新索引上。

索引不会自动滚动，每次需要手动触发_rollover请求去检查索引是否满足滚动条件，满足则进行滚动，需要结合定时任务触发rollover请求，实现自动滚动。

## 4、索引生命周期管理

### 管理策略

index-lifecycle-managemen（ILM），从ES6.6开始，Elasticsearch提供索引生命周期管理功能，索引生命周期管理可以通过API或者kibana界面配置，在kibana中的索引生命周期管理位置如下图(版本6.8.2)：

![](\assets\images\2022\springcloud\es-kibana.png)

点击创建create policy，进入配置界面，可以看到索引的生命周期被分为：`Hot`, `Warm`, `Cold`, `Delete`四个阶段

- Hot phrase:

  该阶段可以根据索引的文档数，大小，时长决定是否调用rollover API来滚动索引

- Warm phrase

  索引在Hot阶段被rollover后便会进入Warm阶段，进入该阶段的索引可以设置为readonly, 用户可以为这个索引设置要使用的attribute， 如对于冷热分离策略，这里可以选择temperature: warm属性，还可以对索引进行forcemerge，shrink，set  priority设置索引优先级（值越大，优先级越高，ES节点重启，会按该优先级恢复索引），kibana配置界面如下：

  ![](\assets\images\2022\springcloud\es-kibana-phrase.png)

- Cold phrase

  索引rollover一段时间后进入cold阶段，通过设置min_age值进入该阶段。从冷热分离架构可以看出冷热属性是具备扩展性的，不仅可以指定hot, warm, 也可以扩展增加hot, warm, cold, freeze等多个冷热属性。如果想使用三层的冷热分离的话这里可以指定为temperature: cold, 此阶段支持对索引的freeze操作

- Delete phrase

  索引rollover一段时间后进入delete阶段，通过设置min_age值进入该阶段，进入该阶段的索引会自动被删除。

### 四个生命周期阶段

phrase与Action：

![](\assets\images\2022\springcloud\es-policy.png)

- Hot

  Set Priority，具有较高优先级的索引将在节点重启后优先恢复。通常，热阶段的指数应具有最高值，而冷阶段的指数应具有最低值。未设置此值的指标的隐含默认优先级为1。

- Warm

  Read-Only：设为只读索引

  Allocate：

  ![](\assets\images\2022\springcloud\es-ilm-policy-warm.png)

  | 名称               | 描述                             |
  | ------------------ | -------------------------------- |
  | number_of_replicas | 要分配给索引的副本数             |
  | include            | 为具有至少一个属性的节点分配索引 |
  | exclude            | 为没有任何属性的节点分配索引     |
  | require            | 为具有所有属性的节点分配索引     |

  ```json
  "allocate": {
      "require": {
          "box_type": "warm" // 分片数据迁移到节点属性 box_type = warm的ES节点 
      },
      "number_of_replicas": 0 // 副本数为0
  },
  "readonly": {}
  ```

  Force Merge：强制合并，`索引将变成只读`

  ```json
  PUT _ilm/policy/my_policy
  {
    "policy": {
      "phases": {
        "warm": {
          "actions": {
            "forcemerge" : {
              "max_num_segments": 1  //合并后的shard里的lucene segments数,
            }
          }
        }
      }
    }
  }
  ```

  Shrink：收缩索引，`索引将变成只读`

  收缩索引API允许您将现有索引缩减为具有较少主分片的新索引。目标索引中请求的主分片数必须是源索引中分片数的一个因子。如果索引中的分片数是素数，则只能缩小为单个主分片。

  新索引将有一个新名称：`shrink-<origin-index-name>`。因此，如果原始索引称为“logs”，则新索引将命名为“shrink-logs”。

  ```json
  PUT _ilm/policy/my_policy
  {
    "policy": {
      "phases": {
        "warm": {
          "actions": {
            "shrink" : {
              "number_of_shards": 1
            }
          }
        }
      }
    }
  }
  ```

- Cold

  Freeze：冷冻索引

  为了使索引可用且可查询更长时间但同时降低其硬件要求，它们可以转换为冻结状态。一旦索引被冻结，它的所有瞬态分片内存（除了映射和分析器）都会被移动到持久存储。

  ```json
  PUT _ilm/policy/my_policy
  {
    "policy": {
      "phases": {
        "cold": {
          "actions": {
            "freeze" : { }
          }
        }
      }
    }
  }
  ```

  <mark>注意</mark>

  冻结一个索引会close并reopen，这会导致短时间内索引不可用，集群会变red，直到这个索引的分片分配完毕。

- Delete

### 时间参数

在 ILM 中，索引滚动后基于 min_age 参数进入下一个阶段（phrase）。未设置，默认是0ms，像hot阶段不用设置min_age。

min_age通常是指从索引被创建时开始计算的时间，如果是滚动索引，min_age是从索引滚动开始计算。在索引生命周期管理检查min_age并过渡到下一个阶段之前，**前一个阶段的操作必须完成**。min_age的单位 s秒，h小时，d天，y年，举例子

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "min_age": "1d",
        "actions": {
          "allocate": {
            "number_of_replicas": 1
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

- 索引创建在 1 天后将移动到 warm 暖阶段。在此之前，索引处于等待状态。
- 进入 warm 暖阶段后，它将等到 30 天后才进入删除 delete 阶段并删除索引。

![](\assets\images\2022\springcloud\es-ilm-min-age.png)

### 节点角色

7.9+版本后，ES出现了节点角色的概念，配置好节点的冷热温节点角色，索引的分片数据迁移不再需要配置分配策略

Elasitcsearch 7.9 之前早期版本，需要配置分片分配策略机制，如下：

```json
"allocate": {
     "require": {
      	"box_type": "warm"
      }
 }
```

7.9+之后新版本相当于我们提前预设定了冷热集群架构的节点不同的角色，后台会帮我们自动迁移数据。

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

我们只需要设置好不同的横向：phrase 和 每个phrase 下的 action（rollover，freeze， number_of_replicas  delete forcemerge 操作）就可以了，其他的我们不需要关注了，后台会有定时任务轮询完整数据各个 phrase 阶段的更迭

### DSL 实战索引生命周期管理

策略不一定需要配置每个阶段，始终使用索引模板来定义具有滚动操作的策略

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
          "rollover": { // 滚动索引
            "max_age": "3d",
            // "max_docs": 5, 不一定都配置
            "max_size": "50gb"  // 数据写入达到50gb，即索引大小
          },
          "set_priority": {
            "priority": 100 // 优先加载，值越大优先级越高，ES节点重启根据该优先级恢复索引
          }
        }
      },
      "warm": {
        "min_age": "30s", // 索引滚动后，经过30秒进入该阶段，触发action操作
        "actions": {
          "forcemerge": {  // 强制合并,索引会变成只读
            "max_num_segments": 1  // 分片只有1个段
          },
           "shrink": {  // 收缩索引，索引会变成只读
                "number_of_shards":1 // 主分片数
           },
          "allocate": {
            "require": {
              "box_type": "warm" // 分片数据迁移到节点属性 box_type = warm的ES节点 
            },
            "number_of_replicas": 0 // 副本数为0
          },
          "set_priority": { // 设置优先级
            "priority": 50 
          }
        }
      },
      "cold": {
        "min_age": "90s",  // 索引滚动后，经过90秒进入该阶段，触发action操作
        "actions": {
          "allocate": {
            "require": {
              "box_type": "cold" // 分片数据迁移到节点属性 box_type = cold的ES节点 
            }
          },
         "freeze": {} // 冷冻索引
        }
      },
      "delete": {
        "min_age": "150s", // 索引滚动后，经过150秒进入该阶段，触发action操作
        "actions": {
          "delete": {} // 删除索引
        }
      }
    }
  }
}

# step3:创建模板，关联配置的ilm_policy ,ES版本7.4
PUT _template/timeseries_template
{
  "index_patterns": ["timeseries-*"],                 
    "settings": {
    	"index": {
        	"number_of_shards": 3,
      		"number_of_replicas": 1,
      		"lifecycle": {
      			"name": "my_custom_policy_filter", // ILM策略
      			"rollover_alias": "timeseries" // 别名
      		},
      		"routing": {
      			"allocation": {
      				"require":{
      				   "box_type": "hot" # 新创建的索引分配到节点属性 box_type = hot的ES节点（热节点）
      				}
      			}
      		}
    	}
    }
}

# step4:创建起始索引（便于滚动），基于序号
PUT timeseries-000001
{
  "aliases": {
    "timeseries": {
      "is_write_index": true // true表示索引是别名的当前写入索引，索引滚动后该属性会变为false
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

# 查看索引的ILM是否生效
GET /timeseries/_ilm/explain
```

| 参数   | 说明                                                         |
| :----- | :----------------------------------------------------------- |
| hot    | 该策略设置索引只要满足其中任一条件：数据写入达到50GB、使用超过3天、doc数超过5，就会触发索引滚动更新。此时系统将创建一个新索引，该索引将重新启动策略，而旧索引将在滚动更新后等待30秒进入warm阶段，这30秒索引其实还是hot阶段，处于一个等待状态 |
| warm   | 索引进入warm阶段后，ILM会将索引收缩到1个分片，强制合并为1个段，数据迁移到warm(温数据)节点。完成该操作后，索引将在70秒（从滚动更新时算起）后进入cold阶段，70-30=40，即索引处于warm阶段40秒 |
| cold   | 索引进入cold阶段后，ILM将索引从warm节点移动到cold（冷数据）节点。完成操作后，索引将在120秒（从滚动更新时算起）后进入delete阶段，120-70=50，即索引处于cold阶段50秒 |
| delete | 索引进入delete阶段后被删除。                                 |

**核心步骤**总结如下：

- 第一步：创建生周期 policy。
- 第二步：创建索引模板，模板中关联 policy 和别名。
- 第三步：创建符合模板的起始索引，并插入数据。
- 第四步: 索引基于配置的 ilm 滚动



> 统一APP消息中心，待办消息索引使用ILM

step1，设置rollover的检查频率

```json
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval": "30s"  // 30秒检查一次rollover，默认10分钟
  }
}
```

step2: 创建ILM策略

```json
PUT _ilm/policy/index_toread_policy_filter
{
  "policy": {
    "phases": {
      "hot": {      // 热阶段
        "actions": {
          "rollover": {
            "max_age": "1y",  // 索引创建1年后
            "max_docs": 20000000, // 索引最大文档数（不包含副本）2000万
            "max_size": "50gb" // 索引大小50GB
          },
          "set_priority": {
            "priority": 100  // 值越大，优先级越高
          }
        }
      },
      "warm": {         // 温阶段
        "min_age": "10d", // 索引滚动更新10天后进入warm阶段
        "actions": {
            "forcemerge": {     // 实现段合并
                "max_num_segments": 1  // 每个分片强制合并为1个段，即每个分片下只有1个段
            },
            "number_of_replicas": 0  // 分片的副本数为0，没有副本
          },
          "set_priority": {
            "priority": 50  // 优先级设为50
          }
        }
      },
      "delete": {           // 删除阶段
        "min_age": "1y", // 索引滚动更新1年后，进入删除阶段
        "actions": {
             "delete": {} // 删除索引
        }
      }
    }
  }
}
```

step3，创建索引模板，关联ILM策略

```json
PUT _template/index_toread_template
{
  "index_patterns": ["index_toread-*"],                
   "settings": {
        "index": {
            "number_of_shards": 3,  // 创建的索引有3个分片
            "number_of_replicas": 1, // 每个分片，1个副本
            "lifecycle":{
                "name": "index_toread_policy_filter", // 关联ILM策略
                "rollover_alias": "index_toread"  // 滚动索引别名
            }
            /*"routing": {
                "allocation":{
                    "require":{
                        "box_type": "hot"  // 路由的ES的hot节点，这里注释掉
                    }
                }
            }*/
        }
    },
    "mappings": {
        "properties": {
            "sourceAppName": {              // 来源系统
                "type": "keyword"
            },
            "pushAppName": {                // 推送的目标应用
                "type": "keyword"
            },
            "subject": {                    // 主标题
                "type": "text"
            },
            "title": {                  // 副标题
                "type": "text"             
            },
            "link": {                   //  待办的链接地址
                "type": "keyword",
                "index": false          // 不支持搜索
            },
            "mobileLink": {
                "type": "keyword",
                "index": false
            },
            "modelId": {                    // 来源系统待办消息业务id，
                "type": "keyword"
            },
            "docCreator": {
                "type": "keyword"           // 发起人账号
            },
            "target": {                     // 接受人账号
                "type": "keyword"
            },
            "createTime": {     // 创建时间
                "type": "date",
                "index": false
            }，
            "endTime": {        // 处理时间
                "type": "date",
                "index": false
            }，
            "extra": {          // 额外字段属性
                "type": "object"
            }
        }
    }
 }
```

stpe4，创建起始索引

```json

// PUT /<index_toread-{now/d}-000001>
PUT /%3Cindex_toread-%7Bnow%2Fd%7D-000001%3E
{
 "aliases": {
    "index_toread": {
      "is_write_index": true  // 通过索引别名进行数据写入
    }
  }
}
```

step5，检查索引IML配置是否生效

```json
GET 索引名称/_ilm/explain
GET /index_toread/_ilm/explain
```

插入数据，观察索引滚动。

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

### 策略更新启用关闭

1. 如果没有index应用这份策略，那么我们可以直接更新该策略。
2. 如果有index应用了这份策略，那么当前正在执行的阶段不会同步修改，当当前阶段结束后，会进入新版本策略的下个阶段。
3. 如果更换了策略，当前正在执行的阶段不会变化，在结束当前阶段后，将会由新的策略管理下一个生命周期。

错误处理：

当在生命周期策略处理中出现异常时，会进入错误阶段，停止策略的执行。

```
GET /myindex/_ilm/explain
```

使用上述API可以看到异常的原因，当解决这个问题，并更新策略后，可以通过下面的API进行重试：

```
POST /myindex/_ilm/retry
```

ILM的状态查看：

```
GET _ilm/status
{
  "operation_mode": "RUNNING"
}
```

| 名称     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| RUNNING  | Normal operation where all policies are executed as normal   |
| STOPPING | ILM has received a request to stop but is still processing some policies |
| STOPPED  | This represents a state where no policies are executed       |

ILM的启用关闭

```json
POST _ilm/start
POST _ilm/stop
```

参考：

https://mp.weixin.qq.com/s/Px5Eo7_aGy8cDLrhToSn_w

https://mp.weixin.qq.com/s/7VQd5sKt_PH56PFnCrUOHQ

https://www.alibabacloud.com/help/zh/elasticsearch/latest/use-ilm-to-separate-hot-data-and-cold-data

https://www.csdn.net/tags/MtTagg0sNDM2NjgtYmxvZwO0O0OO0O0O.html

### 滚动踩坑

ILM触发索引滚动后，旧索引会添加两个设置，通过别名进行写操作会被阻塞（新增、更新、删除）

```json
// 查询索引相关信息
GET /index_todo-2022.07.31-000002
```

![](\assets\images\2022\springcloud\es-ilm-block.png)

索引 index_todo-2022.07.31-000002 不支持删除文档请求了，不支持删除待处理消息的业务场景，我把索引的`blocks.write`属性设置为false，就支持delete请求了

```json
PUT /index_todo-2022.07.31-000002/_settings
{
    "index":{
        "blocks": {
                "write": "false"
            },
    }
}
```

但也导致了ILM 自动滚动索引失败，索引满足滚动条件也不会自动滚动，查看索引的ILM信息

```json
GET /index_todo/_ilm/explain
```

![](\assets\images\2022\springcloud\es-ilm-rollover-error.png)

因为修改索引 index_todo-2022.07.31-000002 允许删除导致。

那我们就手动触发rollover请求

```json
POST /index_todo/_rollover
```

滚动后创建了新的索引 `index_todo-2022.08.03-000004`，已被ILM管理了

```json
GET /index_todo/_ilm/explain
```

![](\assets\images\2022\springcloud\es-ilm-check-rollover-ready.png)

而旧索引`index_todo-2022.08.01-000003`没有被加上`blocks.write=true`的属性，

![](\assets\images\2022\springcloud\es-is-write-index-false.png)

`is_write_index=false`不支持新增文档，但没有了`blocks.write=true`，应该支持删除文档，这是与ILM自动滚动的不同点，测试一下

经过测试确实支持删除文档操作。因为配置ILM策略，在warm阶段，增加forcemerge强制合并或者shrink收缩索引操作，导致索引变成只读，ILM就会给旧索引增加属性`blocks.write=true`，阻塞写操作（不能删除，更新）。

