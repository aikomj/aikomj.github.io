---
layout: post
title: 数据库查询变慢的场景分析与解决方案
category: mysql
tags: [mysql]
keywords: mysql
excerpt: 
lock: noneed
---

## 1、数据库查询流程

现在我们有一张数据表

```sql
CREATE TABLE `user` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(100) NOT NULL DEFAULT '' COMMENT '名字',
  `age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄',
  `gender` int(8) NOT NULL DEFAULT '0' COMMENT '性别',
  PRIMARY KEY (`id`),
  KEY `idx_age` (`age`),
  KEY `idx_gender` (`gender`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

