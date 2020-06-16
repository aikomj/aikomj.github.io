---
layout: post
title: ElasticSearch怎样保存我们的冷热数据？
category: icoding-gavin
tags: [icoding-gavin]
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

# 重启es节点
```

![](/Users/xjw/Documents/code/aikomj.github.io/assets/images/2020/icoding/elasticsearch/es-hot-cool1.jpg)

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

![](/Users/xjw/Documents/code/aikomj.github.io/assets/images/2020/icoding/elasticsearch/es-hot-cool2.jpg)

```shell
# 创建
PUT /index_test
{
    "settings": {
        "index": {
            "number_of_shards": "3",
            "number_of_replicas": "0"
        }
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

![](/Users/xjw/Documents/code/aikomj.github.io/assets/images/2020/icoding/elasticsearch/es-hot-cool3.jpg)



```shell
# 创建索引
PUT /index_test
{
    "settings": {
        "index": {
            "number_of_shards": "1",
            "number_of_replicas": "0",
            "routing" : {
            "allocation" : {
              "require" : {
                "tag" : "cool"
              }
          }
        }
    }
}
```



## 3、数据读写分离

