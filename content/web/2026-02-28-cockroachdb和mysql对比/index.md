---
title: cockroachDB和mysql对比
slug: crdb-vs-mysql
description: 尝试用了下cockroachdb，直接docker拉下，单节点启动。语法和pg基本一样，也当做pg和mysql的对比了，毕竟pg用的还是少，值得不停翻阅。
date: 2026-02-28T07:11:44.744Z
preview: ""
draft: false
tags:
    - 数据库
    - cockroach
categories: []
---

先说下cockroach的优点吧，天然的分布式数据库...巴拉巴拉，官网可查，对我来说，最直观的优点就是自带数据库仪表盘，拥有完善的数据库监视面板，常用的监控功能都有，细化到每一条sql执行都有记录，省了很多事。而且兼容postgre，pg能干的它也能干。

从实用主义出发，从最常用的场景出发，主要感受了四个方面的差异：语法，数据结构，默认事务隔离，权限控制

## 语法

没什么好说的，基本一样，pg和mysql的差异就是它的差异。

- 比如自增主键，mysql是auto_increment，cockroach是serial
- 比如字符串连接，pg和cockroach是||，mysql是concat()。
- 比如类型转换，pg和cockroach是::，mysql是cast()。

此外，对于表的定义，mysql建表后通常支持的表级选项是 **存储引擎、字符集、排序规则、行格式、分区** ,在 CockroachDB 中，只有WITH (...)，通常是`schema_locked = true`
表示 锁定 schema，禁止普通用户 `ALTER TABLE` 修改表结构。cockroach的存储引擎默认是kv分布式，编码是utf-8,这些都是默认的，不需要指定。

## 数据结构

只说关键点，比mysql多了jsonb结构。

```sql
INSERT INTO data_json_test (name, attributes) VALUES
    ('Alice', '{"age": 30, "city": "Beijing", "skills": ["Go", "SQL"]}'::JSONB),
    ('Bob', '{"age": 25, "city": "Shanghai", "skills": ["Python", "Docker"]}'::JSONB),
    ('Charlie', '{"age": 35, "city": "Shenzhen", "skills": ["Java", "K8s"]}'::JSONB);
```

可以通过这样的方式用过滤条件查询jsonb字段：

```sql
-- 查询 skills 包含 "Go" 的用户
SELECT name, attributes->>'skills' AS skills
FROM data_json_test
WHERE attributes->'skills' @> '["Go"]'::JSONB;
```

`@>` 操作符表示 包含,不同于`IN`,它是仅针对jsonb字段的，表示左边的jsonb包含右边的jsonb。也就是说，`attributes->'skills' @> '["Go"]'::JSONB` 的意思是，`attributes` 字段中的 `skills` 数组包含 `"Go"` 这个元素。

## 默认事务隔离

mysql默认是REPEATABLE READ（可重复读），cockroach默认是READ COMMITTED（提交读）。
mysql可以避免脏读、不可重复读，并且它通过gap lock也可以避免幻读，而cockroach只能避免脏读，会出现不可重复读和幻读，这也是pg会出现的情况。

> 脏读：看到别人没提交的“草稿”
>
> 不可重复读：同一个事务里两次查询，结果不同
>
> 幻读：同一个事务里，条件查询结果中突然多了别人新插入的行

但这个默认隔离级别的差异对大多数应用来说影响不大，除非需要非常严格的数据一致性要求，如金融场景，否则默认的READ COMMITTED已经足够了。

### 不可重复读（Non-Repeatable Read）

同一事务中两次读同一条记录，结果可能不同

在 PG 默认 READ COMMITTED 下，每次读取的数据都是 **已提交的最新状态**，所以不会脏读，不会读到未提交错误数据。

不可重复读只是“看到别人提交的最新数据”，对于大多数应用（查询/统计/展示）完全可以接受

### 幻读（Phantom Read）

同一事务条件查询多次，结果行数可能不同（有人插入新行）

PG 默认 READ COMMITTED 下，事务内部的数据是 ACID 的，仅仅是多次查询看到的数据变化，不会破坏事务一致性。

如果业务需要严格保证“查询范围不变”，可以使用更高隔离等级（SERIALIZABLE），但性能开销较大，通常不建议默认使用。

## 权限控制

mysql的权限控制比较细粒度，可以针对不同的用户和数据库对象设置不同的权限。
cockroach的权限控制相对简单一些，主要是基于角色的权限控制，角色可以拥有不同的权限，但没有mysql那么细粒度的权限设置。

超级用户权限 vs 普通用户

- CockroachDB 的 root / admin 可以绕过很多限制（如 schema_locked）
- MySQL 的 root 也很强，但有 host 限制，分布式场景不适用

分布式一致性

- CRDB 的权限表在每个节点上都是 全局一致
- MySQL 在主从复制时，权限表会同步，但存在延迟和 host 限制

schema_locked 与权限

- CRDB 引入 schema_locked，普通用户无法修改表结构
- MySQL 没有类似概念，只能通过 权限控制 实现表保护

默认行为

- MySQL root 默认允许所有操作（在 host 上）
- CRDB root/admin 默认全局权限，普通用户严格受限
