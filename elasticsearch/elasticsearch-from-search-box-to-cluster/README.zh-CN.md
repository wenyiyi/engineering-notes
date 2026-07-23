# 学好 Elasticsearch，搜索问题就没那么吓人了

> 我虽然穷，但是我爱学习呀。当然，如果我有一个亿，我愿意暂时放下这个爱好，先去玩两天。

[English version](README.md)

![Elasticsearch, Made Simple](imgs/cover.png)

这篇文章不准备把 Elasticsearch 讲成一本词典。

我们就顺着一个搜索框往下挖：为什么 MySQL 查起来有点费劲？Elasticsearch 为什么快？数据怎么进去？服务器挂了怎么办？最后再看看项目中到底该不该用它。

---

## 1. 灵魂拷问：有 MySQL 了，为什么还要 Elasticsearch？

假设书库里有一本书：

```text
黑萌影帝妙探妻
```

用户却搜：

```text
影帝黑
```

用 MySQL 可以写：

```sql
SELECT id, title
FROM book
WHERE title LIKE CONCAT('%', ?, '%');
```

参数化以后，SQL 注入的问题可以避免；但搜索体验仍然不太好：

- 字的顺序一变，可能就找不到；
- 不知道哪本书更相关；
- 没有自然的高亮和分词；
- 前面带 `%` 的模糊查询，通常很难利用普通 B+Tree 索引；
- 搜索字段一多，SQL 会逐渐长成一只八爪鱼。

Elasticsearch 的强项刚好是：**先分析文本，再找相关文档，最后按相关度排序。**

![MySQL 与 Elasticsearch 的分工](imgs/01-mysql-es-roles.png)

### 闲狗言

别让 Elasticsearch 抢 MySQL 的饭碗。

- MySQL 保存订单、库存、余额等业务真相；
- Elasticsearch 保存一份为了搜索而设计的“投影”；
- 搜索结果拿到 ID 后，必要时再回 MySQL 查询最新详情。

一句话：**MySQL 管真相，Elasticsearch 管好找。**

---

## 2. 搜索不是 `LIKE` 换了个马甲

![书名搜索过程](imgs/02-book-search.png)

一次全文搜索，大致会经历四步：

1. 把查询文本交给分析器；
2. 分出可检索的词；
3. 找到包含这些词的文档；
4. 计算相关度，返回更像答案的结果。

例如：

```json
GET books/_search
{
  "query": {
    "match": {
      "title": "影帝黑"
    }
  },
  "highlight": {
    "fields": {
      "title": {}
    }
  }
}
```

`match` 会分析输入文本；如果只想精确匹配状态、手机号或订单号，通常应使用 `term` 查询配合 `keyword` 字段。

---

## 3. Elasticsearch 为什么找得快？

先想象一本纸质书。

如果要找“影帝”两个字，可以从第一页看到最后一页；也可以翻到书后的索引，先找到“影帝”出现在哪几页。

倒排索引就是第二种办法。

![倒排索引](imgs/03-inverted-index.png)

假设有三篇文档：

```text
1: 黑萌影帝妙探妻
2: 影帝今天很开心
3: 黑猫侦探社
```

简化后的索引可能像这样：

```text
黑   -> [1, 3]
影帝 -> [1, 2]
侦探 -> [3]
```

左边是词项，右边是包含它的文档列表，也叫 posting list。真实的 Lucene 还会记录词频、位置等信息，供短语匹配和相关度计算使用。

所以搜索时不用把所有文档重新读一遍，而是从词项直接跳到候选文档。

> 这里要修正一个常见误解：MySQL 的 B+Tree 索引和 Lucene 的倒排索引，解决的是不同问题。不是谁“绝对更快”，而是谁的数据结构更适合当前查询。

每个 Elasticsearch 分片本身都是一个完整的 Lucene 索引。[Elastic 官方文档](https://www.elastic.co/docs/deploy-manage/distributed-architecture/clusters-nodes-shards)也明确这样描述。

---

## 4. `text` 还是 `keyword`？别靠感觉选

![text 与 keyword](imgs/04-text-vs-keyword.png)

可以先记住这个不严谨但好用的口诀：

- 要分词、全文搜索、算相关度：`text`
- 要完整匹配、过滤、排序、聚合：`keyword`

```json
PUT books
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "raw": { "type": "keyword" }
        }
      },
      "author":   { "type": "keyword" },
      "price":    { "type": "scaled_float", "scaling_factor": 100 },
      "published_at": { "type": "date" }
    }
  }
}
```

同一个标题可以：

- 用 `title` 做全文搜索；
- 用 `title.raw` 做精确匹配、排序和聚合。

生产环境建议显式设计 mapping，避免动态推断悄悄把字段建错。字段类型一旦不合适，通常要新建索引再 reindex。

### 旧文章里的 `type` 去哪了？

以前会写：

```text
index / type / document
```

现在直接理解为：

```text
index / document
```

Elasticsearch 8 已不再支持 Mapping Type，接口使用 `/{index}/_doc/{id}`。[官方迁移说明](https://www.elastic.co/docs/manage-data/data-store/mapping/removal-of-mapping-types)。

---

## 5. 一条查询到底经历了什么？

![查询旅程](imgs/05-query-journey.png)

```json
GET books/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "影帝黑" } }
      ],
      "filter": [
        { "term": { "status": "published" } },
        { "range": { "price": { "lte": 9900 } } }
      ]
    }
  },
  "sort": [
    "_score",
    { "published_at": "desc" }
  ]
}
```

把条件分成两类会更清楚：

- `must`：这篇文档和搜索词有多像？需要计算分数；
- `filter`：状态对不对、价格超没超？只需要判断是或否。

排序、分页、聚合和高亮都可以继续加，但先别把所有功能堆在一个请求里。功能越多，不代表查询越优秀。

---

## 6. 为什么刚写入的数据不一定马上搜到？

因为 Elasticsearch 是**近实时搜索**，不是“写完这一纳秒就能搜到”。

![近实时写入过程](imgs/06-nrt-write-path.png)

简化过程：

1. 文档写入；
2. 数据进入内存缓冲区，并通过 translog 提供持久性保障；
3. `refresh` 创建并打开新的 Lucene segment；
4. 新 segment 可以被搜索；
5. 后台再把许多小 segment 合并成大 segment。

Elastic Stack 中 `index.refresh_interval` 默认通常为 `1s`；官方把这个过程称为 [near real-time search](https://www.elastic.co/docs/manage-data/data-store/near-real-time-search)。

不要为了“立刻搜到”给每次写入都加：

```text
refresh=true
```

它可能制造许多小 segment，让写入、搜索和后续合并一起交学费。[官方 refresh 参数说明](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/refresh-parameter)建议优先批量写入；确实要等待可见时，可谨慎使用 `refresh=wait_for`。

---

## 7. 分片不是切蛋糕比赛，越多不一定越好

一个 index 会被拆成若干 primary shard，每个 shard 都是一个 Lucene index。

![分片与路由](imgs/07-shards-routing.png)

### 写入

文档根据 `_routing` 计算应该进入哪个主分片。默认 routing 值是文档 `_id`。自定义 routing 可以减少某些查询需要访问的分片，但用不好也会产生热点；读取、更新和删除时还必须提供相同 routing。

### 搜索

客户端可以请求任意节点，这个节点会充当 coordinating node：

1. 把查询发到相关分片副本；
2. 每个分片返回自己的局部优胜者；
3. 协调节点合并、排序；
4. 再取回最终结果。

这就是 scatter-gather。分片过多时，一次查询要问很多“小部门”，协调和资源开销反而会上升。

### 两条容易记错的规则

- 主分片数通常在创建索引时确定，不能像副本数一样随手修改；需要通过 reindex、split 或 shrink 调整。
- 副本数可以动态修改。

官方也建议先根据数据量、节点和查询负载做容量测试，而不是看到 10 台机器就先建 1000 个分片。

---

## 8. 主分片和副本：不是复制粘贴这么简单

![主分片与副本](imgs/08-primary-replica.png)

副本有两个主要作用：

- 主分片所在节点故障时，可以提升副本继续服务；
- 搜索可以从符合条件的分片副本中选择，增加读取能力。

主分片和它的副本不应放在同一个节点，否则机器一挂，两份一起下班。Elasticsearch 会进行分片分配与再平衡；搜索时默认还会根据响应时间、队列长度等因素进行 [adaptive replica selection](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-shard-routing.html)。

集群健康状态可以简单理解为：

- `green`：主分片和副本都已分配；
- `yellow`：主分片可用，但有副本未分配；
- `red`：至少有主分片不可用。

单节点开发环境配置一个副本时经常是黄色，因为同一个节点不能同时放主分片及其副本。这不神秘，是没有第二台机器可放。

---

## 9. MySQL 的数据怎么可靠地跑到 Elasticsearch？

先看几个常见办法：

| 方法 | 优点 | 坑 |
|---|---|---|
| 定时全量同步 | 简单直观 | 慢、浪费资源、实时性差 |
| 按更新时间增量扫表 | 容易实现 | 时间边界、漏数、重复处理 |
| 业务代码双写 | 看起来直接 | 一边成功一边失败，业务耦合重 |
| Binlog / CDC | 对业务侵入小 | 仍需处理乱序、重复、失败恢复 |
| Transactional Outbox + CDC | 业务数据与事件同事务 | 系统组件更多，但可靠性更清楚 |

![MySQL 到 Elasticsearch 的可靠同步](imgs/09-mysql-sync.png)

比较稳妥的思路：

1. MySQL 事务同时写业务表和 outbox；
2. CDC 读取 binlog，把事件送入可靠队列；
3. 消费者按稳定文档 ID 写入 Elasticsearch；
4. 使用版本号或事件序号防止旧事件覆盖新数据；
5. 失败重试，多次失败进入 DLQ；
6. 定期对账和补偿。

### 灵魂拷问：既然有 CDC，还要对账吗？

要。

队列能重试，不代表系统永远不会出现程序 bug、错误 mapping、人工误删或历史脏数据。**同步链路解决日常流转，对账解决“我以为没问题”。**

---

## 10. 一个更贴近项目的搜索例子

用户检索经常会涉及昵称、手机号、标签、会员等级等多个维度。不要把 40 多张业务表原样搬进 Elasticsearch，而要按搜索场景组装成一个文档：

```json
{
  "tenant_id": "shop_1001",
  "user_id": "u_9527",
  "nickname": "闲狗同学",
  "phone": "13800000000",
  "level": "gold",
  "tags": ["java", "mysql"],
  "updated_at": "2026-07-23T10:00:00+08:00",
  "version": 37
}
```

文档 ID 可以设计成：

```text
tenant_id + ":" + user_id
```

这样重复消费同一个事件时仍然更新同一文档，更容易做到幂等。

如果数组中的多个字段必须保持“同一个对象内匹配”，再考虑 `nested`。不要一看到数组就用 `nested`：它会增加索引和查询成本。

房源搜索也一样：

- 有关键词、地铁、商圈、标签等复杂搜索时查 Elasticsearch；
- 只有主键或简单精确条件时，MySQL 可能已经够用；
- 搜到房源 ID 后，再根据一致性要求决定是否回源。

---

## 11. Elasticsearch 的优缺点

![是否需要 Elasticsearch](imgs/10-decision-map.png)

### 适合

- 全文搜索、模糊搜索；
- 相关度排序、高亮、自动补全；
- 多维过滤与聚合；
- 数据量和查询压力需要横向扩展；
- 允许秒级或可控的最终一致性。

### 不适合拿来硬扛

- 余额扣减、库存扣减等强事务；
- 复杂跨文档事务；
- 数据非常少、查询非常简单；
- 团队没有能力维护集群、监控同步和处理故障；
- 业务要求写入后绝对立即可见。

Elasticsearch 不是“查询一定比数据库快”的魔法盒。错误的 mapping、过多分片、深分页、大聚合、高基数字段、没有限制的查询，都可能把 CPU 拉到 100%。

---

## 12. 最后背一张小抄

```text
MySQL：业务真相
Elasticsearch：搜索投影

text：分词、全文搜索
keyword：精确匹配、过滤、排序、聚合

倒排索引：词 -> 文档
refresh：让新 segment 可搜索
primary shard：数据归属
replica shard：容灾 + 读扩展

同步：允许重复，必须幂等
可靠：重试 + DLQ + 对账
```

如果只记一句：

> **不要为了“用了 Elasticsearch”而用 Elasticsearch；当搜索真的成为问题时，再让它专心解决搜索。**

---

## 参考资料

- [Elastic：Mapping Type 已移除](https://www.elastic.co/docs/manage-data/data-store/mapping/removal-of-mapping-types)
- [Elastic：Clusters、nodes 与 shards](https://www.elastic.co/docs/deploy-manage/distributed-architecture/clusters-nodes-shards)
- [Elastic：Near real-time search](https://www.elastic.co/docs/manage-data/data-store/near-real-time-search)
- [Elastic：Refresh parameter](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/refresh-parameter)
- [Elastic：Search shard routing](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-shard-routing.html)
- [Elastic：`_routing` field](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/mapping-routing-field)
- [原始文章：学好 ElasticSearch，摆脱贫困](https://xiangou.blog.csdn.net/article/details/92105823)

