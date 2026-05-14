# Elasticsearch 实战指南（PHP 版）

> 从索引到分词，从聚合到相似度，全 PHP 代码驱动

---

## 目录

1. [核心概念深入](#1-核心概念深入)
2. [分布式部署](#2-分布式部署)
3. [安装与连接](#3-安装与连接)
4. [索引管理](#4-索引管理)
5. [文档 CRUD](#5-文档-crud)
6. [搜索详解](#6-搜索详解)
7. [中文分词](#7-中文分词)
8. [聚合分析](#8-聚合分析)
9. [高亮与建议词](#9-高亮与建议词)
10. [相似度与推荐](#10-相似度与推荐)
11. [地理位置搜索](#11-地理位置搜索)
12. [自动补全](#12-自动补全)
13. [批量操作与性能](#13-批量操作与性能)
14. [实战：电商搜索](#14-实战电商搜索)
15. [运维与调优](#15-运维与调优)

---

## 1. 核心概念深入

### 1.1 Elasticsearch 是什么

```
Elasticsearch 是一个基于 Lucene 的分布式搜索和分析引擎。
由 Elastic 公司开发，Java 编写，通过 RESTful API 操作。

核心能力：
- 全文搜索（分词、相关性评分）
- 实时数据分析（聚合）
- 分布式存储（自动分片、故障转移）
- 近实时搜索（1秒内可查）

核心概念对应：
  MySQL       →  Elasticsearch
  ─────────      ──────────────
  Database    →  (无，靠集群隔离)
  Table       →  Index (索引)
  Row         →  Document (文档)
  Column      →  Field (字段)
  Schema      →  Mapping (映射)
  SQL         →  Query DSL (JSON 查询)

  SELECT *    →  GET /index/_search { "query": {...} }
  INSERT      →  POST /index/_doc
  UPDATE      →  POST /index/_update/{id}
  DELETE      →  DELETE /index/_doc/{id}
```

### 1.2 文档（Document）

```
文档是 ES 的最小数据单元，相当于 MySQL 的一行记录。
JSON 格式，包含一个或多个字段。

例：
{
  "id": 1001,
  "title": "PHP操作Elasticsearch",
  "content": "本文详细介绍...",
  "tags": ["php", "elasticsearch"],
  "created_at": "2026-01-15 10:00:00"
}

特性：
- 每个文档有唯一 _id（可自动生成或指定）
- 文档是不可变的（update 实际是 delete + insert）
- 文档被索引后，约 1 秒后可搜索（refresh_interval 控制）
```

### 1.3 索引（Index）

```
索引是文档的集合，相当于 MySQL 的一张表。

一个 ES Cluster 可以有多个 Index，一个 Index 可以有多个 Type（7.x 已废弃，8.x 彻底移除）

Index 包含：
┌─────────────────────────────┐
│  Index: articles            │
├─────────────────────────────┤
│  Settings（配置）             │
│  ├─ 分片数                   │
│  ├─ 副本数                   │
│  ├─ 分词器                   │
│  └─ refresh 间隔             │
├─────────────────────────────┤
│  Mappings（映射 = Schema）    │
│  ├─ title: text + keyword   │
│  ├─ content: text           │
│  ├─ view_count: integer     │
│  └─ created_at: date        │
├─────────────────────────────┤
│  Documents（数据）            │
│  ├─ Doc 1                   │
│  ├─ Doc 2                   │
│  └─ ...                     │
└─────────────────────────────┘

Index 命名限制：
- 只能小写字母
- 不能以 _ 或 - 开头
- 不能包含空格和特殊字符
```

### 1.4 分片（Shard）

```
分片是索引的物理存储单元。一个 Index 的数据分布在多个 Shard 上。

为什么需要分片：
1. 水平扩展：单节点装不下所有数据
2. 并行查询：多个分片可以同时搜索，提速
3. 写入并发：多分片并行写入

分片类型：
┌────────────────────────────────────────┐
│          Index: articles               │
│          (3 primary shards)            │
├──────────┬──────────┬─────────────────┤
│ Shard P0 │ Shard P1 │    Shard P2     │
│ 主分片0  │ 主分片1  │    主分片2      │
│ 文档 0-3 │ 文档 4-7 │    文档 8-11    │
└──────────┴──────────┴─────────────────┘

Primary Shard（主分片）：
- 每个文档只存在一个主分片上
- 写操作先写主分片
- 数量在创建 Index 时确定，不可修改
- 默认 1 个（7.x），可设置 number_of_shards

Replica Shard（副本分片）：
- 主分片的拷贝
- 提供数据冗余（容灾）
- 可以处理读请求（提高查询吞吐）
- 数量可在运行时修改

常见配置：
3 个节点 + 3 主分片 + 1 副本 = 每个节点 2 个分片（1主+1副）
```

### 1.5 分片路由算法

```
文档分配到哪个分片？

shard = hash(_routing) % number_of_primary_shards

默认 _routing = _id（即文档ID）

例：3个主分片
文档ID=1001 → hash("1001") % 3 = 0 → Shard P0
文档ID=1002 → hash("1002") % 3 = 1 → Shard P1
文档ID=1003 → hash("1003") % 3 = 2 → Shard P2

也可以指定 routing（同一 routing 的文档分到同一分片）：
PUT /articles/_doc/1001?routing=user_123
→ 同一用户的所有文章都在同一分片

⚠️ 分片数不可修改的原因：
如果分片数从 3 改为 5，hash % 3 ≠ hash % 5，数据全乱
```

### 1.6 副本（Replica）

```
副本 = 主分片的完整拷贝，提供容灾和查询负载分担。

┌──────────────────────────────────────────────┐
│              3节点集群                         │
├──────────────┬──────────────┬────────────────┤
│   Node 1     │    Node 2    │    Node 3      │
├──────────────┼──────────────┼────────────────┤
│  P0 文档0-3  │  P1 文档4-7  │  P2 文档8-11   │
│  R2 文档8-11 │  R0 文档0-3  │  R1 文档4-7    │
└──────────────┴──────────────┴────────────────┘

特性：
- 主副永不在同一节点（否则宕机全丢）
- 副本跟随主分片同步（主分片写入 → 副本同步）
- 读请求可以走副本（增加读吞吐）
- 写请求只能走主分片
- 副本数 >= 节点数 - 1 才有意义

副本数选择：
- 0：无容灾（单节点，开发环境）
- 1：1份副本（生产最低配置，数据有备份）
- 2：2份副本（高要求场景，可承受2个节点宕机）
```

### 1.7 节点（Node）

```
一个 ES 实例 = 一个节点

节点角色（可兼任）：
- Master Node（主节点）：管理集群、分配分片、跟踪节点状态
- Data Node（数据节点）：存储数据、执行搜索和聚合
  可细分：hot（热/SSD）、warm（温/HDD）、cold（冷/归档）
- Ingest Node（摄取节点）：数据写入前的预处理（Pipeline）
- Coordinating Node（协调节点）：接收请求→转发→汇总（每个节点默认都有）
- ML Node（机器学习，8.0+）：异常检测、预测

最佳实践：
- 小集群（3-5节点）：master + data 混合
- 中集群（10+节点）：独立 master 节点（3个专用防止脑裂）
- 大集群（30+节点）：全角色分离
```

### 1.8 集群（Cluster）

```
多个节点组成一个集群，共同存储全部数据并提供搜索。

集群发现：
- 配置 cluster.name 相同
- 通过 discovery.seed_hosts 列表互相发现
- 选举出一个 Master 节点

脑裂（Split Brain）：
  网络分区导致集群分裂 → 各自选举 Master → 数据不一致
  防止：cluster.initial_master_nodes 预先定义候选节点
```

### 1.9 为什么 ES 搜索快

```
倒排索引（Inverted Index）：

正排索引（MySQL）：扫描所有文档
倒排索引（ES）：词→文档的映射，O(1)查找

  php           → [Doc1, Doc3]
  elasticsearch → [Doc1]
  redis         → [Doc2, Doc3]
  缓存          → [Doc2, Doc3]
  指南          → [Doc1, Doc2]

  查"php redis" → 取交集 Doc1 + Doc3 相交 → Doc3！

加分项：
- Lucene 跳表（Skip List）加速 AND/OR 操作
- DocValues（列式存储）加速排序和聚合
- 所有数据在内存映射文件中，利用 OS Page Cache
- segment 不可变，不需要事务锁
```

### 1.10 近实时原理

```
写入流程：
1. Document 写入内存 Buffer（此时不可搜索）
2. 同时写入 Translog（事务日志，防数据丢失）
3. refresh：Buffer 写入 Segment（默认每1秒）→ 可搜索！
4. flush：Segment 持久化到磁盘 + 清空 Translog（每30分钟或满时）

           Client
             │
    ┌────────▼────────┐
    │   Memory Buffer  │  ← 写入
    └────────┬────────┘
             │  refresh（1秒）← 变可搜索
    ┌────────▼────────┐
    │    Segment       │  ← OS Page Cache
    └────────┬────────┘
             │  flush（30分钟）← 持久化
    ┌────────▼────────┐
    │     磁盘          │
    └─────────────────┘

          Translog（每次写都fsync）
    ┌─────────────────┐
    │ 写操作日志       │  ← 崩溃恢复用
    └─────────────────┘
```

---

## 2. 分布式部署

### 2.1 架构规划

```
生产环境推荐：

方案A：3节点小集群（10-50GB 数据量）
  Node 1: master+data, 8C 32G, SSD 500G
  Node 2: master+data, 8C 32G, SSD 500G
  Node 3: master+data, 8C 32G, SSD 500G

方案B：5节点中集群（100GB+ 数据量）
  Master 1-3: 仅主节点, 4C 8G
  Data 1-2:   仅数据, 16C 64G, SSD 1TB

方案C：冷热分离架构
  Hot Nodes: 16C 64G, SSD 1TB（近30天索引）
  Warm Nodes: 8C 32G, HDD 4TB（30-90天索引）
```

### 2.2 安装配置

```bash
# ═══ 每个节点都要做 ═══

# 系统调优
# /etc/sysctl.conf
vm.max_map_count = 262144
vm.swappiness = 1

# /etc/security/limits.conf
elasticsearch  soft  nofile  65536
elasticsearch  hard  nofile  65536
elasticsearch  soft  memlock unlimited

sysctl -p

# Docker 安装（推荐）
docker run -d \
  --name es-node1 \
  --net es-net \
  -p 9200:9200 -p 9300:9300 \
  -e "cluster.name=my-es-cluster" \
  -e "node.name=node-1" \
  -e "network.publish_host=<节点IP>" \
  -e "discovery.seed_hosts=node-1,node-2,node-3" \
  -e "cluster.initial_master_nodes=node-1,node-2,node-3" \
  -e "ES_JAVA_OPTS=-Xms16g -Xmx16g" \
  -e "xpack.security.enabled=true" \
  -e "ELASTIC_PASSWORD=your_password" \
  -v es-data1:/usr/share/elasticsearch/data \
  elasticsearch:8.0.0
```

### 2.3 elasticsearch.yml 完整配置

```yaml
# ═══ 集群 ═══
cluster.name: my-es-cluster
node.name: node-1

# ═══ 节点角色 ═══
# 默认所有角色: [master, data, ingest, ml]
# 专用master: [master]
# 专用data: [data]
# 冷热标记: node.attr.box_type: hot | warm

# ═══ 网络 ═══
network.host: 0.0.0.0
network.publish_host: 192.168.1.101
http.port: 9200
transport.port: 9300

# ═══ 发现 ═══
discovery.seed_hosts:
  - 192.168.1.101:9300
  - 192.168.1.102:9300
  - 192.168.1.103:9300
cluster.initial_master_nodes:
  - node-1
  - node-2
  - node-3

# ═══ 安全 ═══
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true

# ═══ 内存（通过 ES_JAVA_OPTS 设置） ═══
# -Xms16g -Xmx16g  (堆 ≤ 31GB，总内存的 50%)
```

### 2.4 冷热分离 + ILM（索引生命周期）

```
ILM 自动管理索引的整个生命周期：

Hot（热）→ Warm（温）→ Cold（冷）→ Delete（删）
```

```php
// 创建 ILM 策略
$es->ilm()->putLifecycle([
    'policy' => 'logs_policy',
    'body'   => [
        'policy' => [
            'phases' => [
                'hot' => [
                    'min_age' => '0ms',
                    'actions' => [
                        'rollover' => [
                            'max_size'  => '50GB',
                            'max_age'   => '30d',
                            'max_docs'  => 100000000,
                        ],
                    ],
                ],
                'warm' => [
                    'min_age' => '7d',
                    'actions' => [
                        'shrink'   => ['number_of_shards' => 1],
                        'forcemerge' => ['max_num_segments' => 1],
                        'allocate'  => ['require' => ['box_type' => 'warm']],
                    ],
                ],
                'cold' => [
                    'min_age' => '30d',
                    'actions' => [
                        'allocate' => ['require' => ['box_type' => 'cold']],
                        'freeze'   => new \stdClass(),
                    ],
                ],
                'delete' => [
                    'min_age' => '90d',
                    'actions' => [
                        'delete' => new \stdClass(),
                    ],
                ],
            ],
        ],
    ],
]);

// 绑定到索引模板
$es->indices()->putTemplate([
    'name' => 'logs_ilm_template',
    'index_patterns' => ['logs-*'],
    'body' => [
        'settings' => [
            'index.lifecycle.name' => 'logs_policy',
            'index.lifecycle.rollover_alias' => 'logs',
        ],
    ],
]);
```

### 2.5 Docker Compose 一键部署

```yaml
# docker-compose.yml
version: '3.8'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=false
    ulimits:
      memlock: { soft: -1, hard: -1 }
    volumes:
      - es_data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - es_net

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=false
    ulimits:
      memlock: { soft: -1, hard: -1 }
    volumes:
      - es_data02:/usr/share/elasticsearch/data
    networks:
      - es_net

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=false
    ulimits:
      memlock: { soft: -1, hard: -1 }
    volumes:
      - es_data03:/usr/share/elasticsearch/data
    networks:
      - es_net

  kibana:
    image: docker.elastic.co/kibana/kibana:8.0.0
    container_name: kibana
    ports:
      - 5601:5601
    environment:
      ELASTICSEARCH_HOSTS: '["http://es01:9200","http://es02:9200","http://es03:9200"]'
    networks:
      - es_net

volumes:
  es_data01:
  es_data02:
  es_data03:

networks:
  es_net:
    driver: bridge

# 启动：docker-compose up -d
# 检查：curl http://localhost:9200/_cluster/health?pretty
```

### 2.6 扩容操作

```bash
# ═══ 添加新节点 ═══
# 1. 新机器装 ES，配置相同 cluster.name + discovery.seed_hosts
# 2. 启动 → 自动加入集群 → 分片自动重平衡

# ═══ 下线节点 ═══
# 先让分片迁移走
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.exclude._name": "node-3"
  }
}
'
# 观察迁移完成后关停

# ═══ 查看分片分配 ═══
curl "localhost:9200/_cat/shards/articles?v&s=shard"
```

---

## 3. 安装与连接

```bash
composer require elasticsearch/elasticsearch
```

```php
use Elastic\Elasticsearch\ClientBuilder;

final class ESFactory
{
    private static ?\Elastic\Elasticsearch\Client $client = null;

    public static function get(): \Elastic\Elasticsearch\Client
    {
        if (self::$client === null) {
            self::$client = ClientBuilder::create()
                ->setHosts(['localhost:9200'])
                // ->setBasicAuthentication('elastic', 'password')  // 有认证时
                // ->setSSLVerification(false)  // 自签证书时
                ->setRetries(3)
                ->build();
        }
        return self::$client;
    }
}

$es = ESFactory::get();
```

---

## 4. 索引管理

### 3.1 创建索引（带 Mapping + 分词器）

```php
$es->indices()->create([
    'index' => 'articles',
    'body'  => [
        'settings' => [
            'number_of_shards'   => 3,    // 分片数（创建后不可改）
            'number_of_replicas' => 1,    // 副本数
            'refresh_interval'   => '5s', // 刷新频率
            'analysis' => [
                'analyzer' => [
                    // 自定义 ik 分词器
                    'ik_smart_analyzer' => [
                        'type'      => 'custom',
                        'tokenizer' => 'ik_smart',
                        'filter'    => ['lowercase'],
                    ],
                    // 拼音分词器（需安装 pinyin 插件）
                    'pinyin_analyzer' => [
                        'type'      => 'custom',
                        'tokenizer' => 'keyword',
                        'filter'    => ['pinyin_filter'],
                    ],
                ],
                'filter' => [
                    'pinyin_filter' => [
                        'type' => 'pinyin',
                        'keep_full_pinyin'     => true,
                        'keep_joined_full_pinyin' => true,
                        'keep_original'        => true,
                        'limit_first_letter_length' => 16,
                        'remove_duplicated_term' => true,
                    ],
                ],
            ],
        ],
        'mappings' => [
            'properties' => [
                'id' => ['type' => 'long'],
                'title' => [
                    'type'     => 'text',
                    'analyzer' => 'ik_max_word',   // 索引时用细粒度
                    'search_analyzer' => 'ik_smart', // 搜索时用粗粒度
                    'fields' => [
                        'keyword' => ['type' => 'keyword', 'ignore_above' => 256],
                        'pinyin'  => ['type' => 'text', 'analyzer' => 'pinyin_analyzer'],
                    ],
                ],
                'content' => [
                    'type'     => 'text',
                    'analyzer' => 'ik_max_word',
                ],
                'summary' => ['type' => 'text', 'analyzer' => 'ik_smart'],
                'tags'    => ['type' => 'keyword'],        // 精确匹配数组
                'category_id' => ['type' => 'integer'],
                'author_name' => [
                    'type'     => 'text',
                    'fields' => [
                        'keyword' => ['type' => 'keyword'],
                    ],
                ],
                'view_count'  => ['type' => 'integer'],
                'is_published' => ['type' => 'boolean'],
                'score'        => ['type' => 'float'],
                'created_at'   => ['type' => 'date', 'format' => 'yyyy-MM-dd HH:mm:ss||epoch_second'],
                'updated_at'   => ['type' => 'date'],
                // 坐标字段
                'location' => ['type' => 'geo_point'],
                // 嵌套对象
                'comments' => [
                    'type' => 'nested',
                    'properties' => [
                        'user_id' => ['type' => 'integer'],
                        'content' => ['type' => 'text', 'analyzer' => 'ik_smart'],
                        'created_at' => ['type' => 'date'],
                    ],
                ],
            ],
        ],
    ],
]);
```

### 3.2 索引别名（零停机切换）

```php
// 创建索引 v1
$es->indices()->create(['index' => 'articles_v1', 'body' => [...]]);

// 指向 v1
$es->indices()->putAlias(['index' => 'articles_v1', 'name' => 'articles']);

// 重建索引 v2 → 切换别名 → 删 v1
$es->indices()->create(['index' => 'articles_v2', 'body' => [...]]);

$es->indices()->updateAliases([
    'body' => [
        'actions' => [
            ['remove' => ['index' => 'articles_v1', 'alias' => 'articles']],
            ['add'    => ['index' => 'articles_v2', 'alias' => 'articles']],
        ],
    ],
]);

$es->indices()->delete(['index' => 'articles_v1']);
// → 用户无感知！
```

### 3.3 常用索引操作

```php
// 查看索引是否存在
$es->indices()->exists(['index' => 'articles']);

// 查看 Mapping
$es->indices()->getMapping(['index' => 'articles']);

// 查看 Settings
$es->indices()->getSettings(['index' => 'articles']);

// 重建索引（从 A 复制到 B，可修改 mapping）
$es->reindex([
    'body' => [
        'source' => ['index' => 'articles'],
        'dest'   => ['index' => 'articles_v2'],
        'script' => [
            'source' => 'ctx._source.new_field = ctx._source.old_field',
        ],
    ],
]);

// 删除索引
$es->indices()->delete(['index' => 'articles_v1']);

// 查看所有索引及大小
$es->cat()->indices(['v' => true, 's' => 'store.size:desc']);
```

---

## 5. 文档 CRUD

```php
$es = ESFactory::get();

// ═══ 创建 / 全覆盖 ═══
$response = $es->index([
    'index' => 'articles',
    'id'    => 1001,          // 不指定会自动生成
    'body'  => [
        'title'       => 'PHP操作Elasticsearch完整指南',
        'content'     => '本文详细介绍如何使用PHP客户端操作ES...',
        'summary'     => 'ES PHP 指南',
        'tags'        => ['php', 'elasticsearch', '搜索'],
        'category_id' => 3,
        'author_name' => 'khz',
        'view_count'  => 0,
        'is_published'=> true,
        'score'       => 4.5,
        'created_at'  => date('Y-m-d H:i:s'),
    ],
]);

// ═══ 查询 ═══
$doc = $es->get(['index' => 'articles', 'id' => 1001]);
$data = $doc['_source'];

// ═══ 部分更新 ═══
$es->update([
    'index' => 'articles',
    'id'    => 1001,
    'body'  => [
        'doc' => [
            'view_count' => 128,
            'updated_at' => date('Y-m-d H:i:s'),
        ],
    ],
]);

// ═══ 脚本更新（原子递增） ═══
$es->update([
    'index' => 'articles',
    'id'    => 1001,
    'body'  => [
        'script' => [
            'source' => 'ctx._source.view_count += params.count',
            'params' => ['count' => 1],
        ],
    ],
]);

// ═══ 脚本更新：添加数组元素 ═══
$es->update([
    'index' => 'articles',
    'id'    => 1001,
    'body'  => [
        'script' => [
            'source' => 'ctx._source.tags.add(params.tag)',
            'params' => ['tag' => '教程'],
        ],
    ],
]);

// ═══ 删除 ═══
$es->delete(['index' => 'articles', 'id' => 1001]);

// ═══ 条件更新（文档不存在时创建） ═══
$es->update([
    'index' => 'articles',
    'id'    => 1001,
    'body'  => [
        'script' => [
            'source' => 'ctx._source.view_count++',
        ],
        'upsert' => [
            'title'      => '新文章',
            'view_count' => 1,
        ],
    ],
]);
```

---

## 6. 搜索详解

### 5.1 基础查询

```php
// ═══ match（分词后匹配） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'match' => [
                'title' => [
                    'query'    => 'PHP Elasticsearch 操作',
                    'operator' => 'and',      // or(默认) / and
                    'minimum_should_match' => '75%',  // 至少匹配 75%
                ],
            ],
        ],
    ],
]);

// ═══ multi_match（多字段） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'multi_match' => [
                'query'  => 'php 搜索',
                'fields' => ['title^3', 'content^2', 'summary', 'tags'],  // ^3=权重3倍
                'type'   => 'best_fields',  // best_fields / most_fields / cross_fields / phrase
                'tie_breaker' => 0.3,
            ],
        ],
    ],
]);

// ═══ term（精确匹配，不分词） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'term' => [
                'tags' => 'php',  // keyword 类型
            ],
        ],
    ],
]);

// ═══ terms（多值精确匹配） ═══
$es->search([
    'index' => 'articles',
    'body' => [
        'query' => [
            'terms' => [
                'category_id' => [1, 2, 3],
            ],
        ],
    ],
]);

// ═══ range（范围） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'range' => [
                'created_at' => [
                    'gte' => '2026-01-01',
                    'lte' => '2026-06-01',
                ],
            ],
        ],
    ],
]);

// ═══ exists / missing ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => ['exists' => ['field' => 'author_name']],
    ],
]);

// ═══ prefix（前缀） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'prefix' => [
                'title.keyword' => 'PHP',
            ],
        ],
    ],
]);

// ═══ wildcard（通配符） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'wildcard' => [
                'title.keyword' => '*ES*',  // * 任意字符, ? 单字符
            ],
        ],
    ],
]);

// ═══ fuzzy（模糊/纠错） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'fuzzy' => [
                'title' => [
                    'value'          => 'Elasticseach',  // 故意拼错
                    'fuzziness'      => 'AUTO',           // 自动计算编辑距离
                    'prefix_length'  => 2,                // 前2字符必须精确
                ],
            ],
        ],
    ],
]);
```

### 5.2 Bool 组合查询

```php
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'bool' => [
                // must = AND（必须匹配，影响分数）
                'must' => [
                    ['match' => ['title' => 'php']],
                    ['term'  => ['is_published' => true]],
                ],
                // must_not = NOT（必须不匹配，不影响分数）
                'must_not' => [
                    ['term' => ['tags' => '废弃']],
                ],
                // should = OR（匹配加分）
                'should' => [
                    ['match' => ['title' => 'elasticsearch']],
                    ['match' => ['content' => 'elasticsearch']],
                ],
                // minimum_should_match: 至少匹配几个 should
                'minimum_should_match' => 1,
                // filter = 过滤（不参与打分，会缓存，性能好）
                'filter' => [
                    ['range' => ['created_at' => ['gte' => '2026-01-01']]],
                    ['term'  => ['category_id' => 3]],
                ],
            ],
        ],
        // 按分数排序
        'sort' => [
            '_score' => ['order' => 'desc'],
            'created_at' => ['order' => 'desc'],
        ],
        'from' => 0,
        'size' => 20,
    ],
]);
```

### 5.3 搜索后处理

```php
$result = $es->search([
    'index' => 'articles',
    'body'  => [
        'query'  => ['match' => ['title' => 'php']],
        'from'   => 0,
        'size'   => 20,

        // 高亮
        'highlight' => [
            'pre_tags'  => ['<em class="hl">'],
            'post_tags' => ['</em>'],
            'fields'    => [
                'title'   => ['number_of_fragments' => 0],  // 0=返回整个字段
                'content' => [
                    'fragment_size'       => 150,
                    'number_of_fragments' => 2,  // 返回2个摘要片段
                ],
            ],
        ],

        // 排除某些字段
        '_source' => [
            'excludes' => ['content'],  // 不返回正文
        ],

        // 解释评分（调试用）
        'explain' => false,
    ],
]);

// 解析结果
$total = $result['hits']['total']['value'];
$maxScore = $result['hits']['max_score'];
$took = $result['took'];  // 毫秒

foreach ($result['hits']['hits'] as $hit) {
    $id     = $hit['_id'];
    $score  = $hit['_score'];
    $source = $hit['_source'];
    $highlight = $hit['highlight'] ?? [];

    echo "ID: {$id}, Score: {$score}\n";
    echo "Title: {$source['title']}\n";
    if (!empty($highlight['title'])) {
        echo "HL: {$highlight['title'][0]}\n";
    }
}
```

---

## 7. 中文分词

### 6.1 IK 分词器

```bash
# 安装（每个节点都要装）
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v8.0.0/elasticsearch-analysis-ik-8.0.0.zip

# 自定义词典
# vim config/analysis-ik/IKAnalyzer.cfg.xml
# vim config/analysis-ik/custom/mydict.dic  一行一个词
```

```php
// ═══ 测试分词 ═══
$result = $es->indices()->analyze([
    'index' => 'articles',
    'body'  => [
        'analyzer' => 'ik_max_word',
        'text'     => '中华人民共和国今天成立了',
    ],
]);
// 输出：中华人民共和国, 中华人民, 中华, 华人, 人民共和国, 人民, 共和国, 共和, 今天, 成立

// ik_max_word vs ik_smart
// ik_max_word: 细粒度（全文索引）
// 北京大学 → 北京大学, 北京, 大学, 京大
// ik_smart:   粗粒度（搜索时用）
// 北京大学 → 北京大学
```

### 6.2 自定义分词映射

```php
// 索引时用 ik_max_word，搜索时用 ik_smart
$mapping = [
    'title' => [
        'type'            => 'text',
        'analyzer'        => 'ik_max_word',
        'search_analyzer' => 'ik_smart',
    ],
];

// 场景：用户搜 "苹果手机"
// 索引时：苹果手机 → [苹果, 手机, 苹果手机]  (ik_max_word)
// 搜索时：苹果手机 → [苹果手机]              (ik_smart)
// → 匹配到标题含"苹果手机"、"苹果"、"手机"的文档
```

### 6.3 同义词

```php
// 在 settings 中配置
$es->indices()->create([
    'index' => 'products',
    'body'  => [
        'settings' => [
            'analysis' => [
                'filter' => [
                    'my_synonym' => [
                        'type'     => 'synonym',
                        // 方式1：内联
                        'synonyms' => [
                            'php, 世界上最好的语言',
                            '计算机, 电脑, computer',
                            'mac, macbook, 苹果电脑',
                        ],
                        // 方式2：文件（可动态更新）
                        // 'synonyms_path' => 'analysis/synonym.txt',
                    ],
                ],
                'analyzer' => [
                    'ik_synonym' => [
                        'tokenizer' => 'ik_smart',
                        'filter'     => ['my_synonym', 'lowercase'],
                    ],
                ],
            ],
        ],
        'mappings' => [
            'properties' => [
                'title' => [
                    'type'     => 'text',
                    'analyzer' => 'ik_synonym',
                ],
            ],
        ],
    ],
]);

// 搜索 "电脑" → 也能匹配到 "计算机"、"computer"
```

### 6.4 拼音插件

```bash
# 安装
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v8.0.0/elasticsearch-analysis-pinyin-8.0.0.zip
```

```php
// 多字段映射：keyword + ik + pinyin
'mappings' => [
    'properties' => [
        'title' => [
            'type'   => 'text',
            'analyzer' => 'ik_max_word',
            'fields' => [
                'keyword' => ['type' => 'keyword'],
                'pinyin'  => [
                    'type'     => 'text',
                    'analyzer' => 'pinyin_analyzer',
                ],
            ],
        ],
    ],
],

// 搜索 "zhongguo" → 匹配 "中国"
// 搜索时 multi_match 同时查 title + title.pinyin
```

---

## 8. 聚合分析

### 7.1 分组统计

```php
// ═══ 按分类 group by ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'size' => 0,  // 不返回文档，只聚合
        'aggs' => [
            'group_by_category' => [
                'terms' => [
                    'field' => 'category_id',
                    'size'  => 20,
                    'order' => ['_count' => 'desc'],
                ],
                // 子聚合：每个分类下的平均阅读量
                'aggs' => [
                    'avg_views' => [
                        'avg' => ['field' => 'view_count'],
                    ],
                    'stats' => [
                        'stats' => ['field' => 'view_count'],
                    ],
                ],
            ],
        ],
    ],
]);

// 结果解析
foreach ($result['aggregations']['group_by_category']['buckets'] as $bucket) {
    echo "分类 {$bucket['key']}: {$bucket['doc_count']} 篇\n";
    echo "  平均阅读: {$bucket['avg_views']['value']}\n";
    echo "  最大: {$bucket['stats']['max']}, 最小: {$bucket['stats']['min']}\n";
}
```

### 7.2 日期直方图

```php
$es->search([
    'index' => 'articles',
    'body'  => [
        'size' => 0,
        'query' => [
            'range' => [
                'created_at' => ['gte' => '2026-01-01', 'lte' => '2026-06-30'],
            ],
        ],
        'aggs' => [
            'articles_over_time' => [
                'date_histogram' => [
                    'field'          => 'created_at',
                    'calendar_interval' => 'month',  // day / week / month / quarter / year
                    'format'         => 'yyyy-MM',
                    'min_doc_count'  => 1,
                    'extended_bounds' => [
                        'min' => '2026-01-01',
                        'max' => '2026-06-30',
                    ],
                ],
                'aggs' => [
                    'avg_score' => ['avg' => ['field' => 'score']],
                ],
            ],
        ],
    ],
]);
```

### 7.3 嵌套聚合

```php
// 文章下的评论数 Top10
$es->search([
    'index' => 'articles',
    'body'  => [
        'size' => 0,
        'aggs' => [
            'top_articles' => [
                'terms' => ['field' => 'id', 'size' => 10],
                'aggs' => [
                    'top_hit' => [
                        'top_hits' => ['size' => 1, '_source' => ['title']],
                    ],
                    // 嵌套聚合（comments 是 nested 类型）
                    'comment_count' => [
                        'nested' => ['path' => 'comments'],
                        'aggs' => [
                            'total' => ['value_count' => ['field' => 'comments.user_id']],
                        ],
                    ],
                ],
            ],
        ],
    ],
]);
```

### 7.4 百分比聚合

```php
$es->search([
    'index' => 'articles',
    'body' => [
        'size' => 0,
        'aggs' => [
            'load_time_percentiles' => [
                'percentiles' => [
                    'field'    => 'view_count',
                    'percents' => [50, 90, 95, 99],
                ],
            ],
        ],
    ],
]);
// 输出：{ "50": 128, "90": 5000, "95": 12000, "99": 50000 }
// 解读：50% 的文章阅读量 ≤128，99% ≤50000
```

---

## 9. 高亮与建议词

### 8.1 高亮

```php
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => ['match' => ['content' => 'php redis']],
        'highlight' => [
            'pre_tags'  => ['<mark>'],
            'post_tags' => ['</mark>'],
            'fields' => [
                'title' => [
                    'number_of_fragments' => 0,  // 返回完整字段
                ],
                'content' => [
                    'fragment_size'       => 100,
                    'number_of_fragments' => 3,
                    'no_match_size'       => 50,  // 无匹配时也返回前50字
                ],
            ],
            'require_field_match' => false,  // 高亮不限于查询字段
        ],
    ],
]);
```

### 8.2 搜索建议（Did You Mean）

```php
// ═══ term suggester（拼写纠错） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'suggest' => [
            'text' => 'elasticserch',
            'spell_suggestion' => [
                'term' => [
                    'field'    => 'title',
                    'suggest_mode' => 'popular',  // missing / popular / always
                    'min_word_length' => 3,
                ],
            ],
        ],
    ],
]);

// 解析建议
foreach ($result['suggest']['spell_suggestion'][0]['options'] as $opt) {
    echo "建议: {$opt['text']}, 分数: {$opt['score']}\n";
}

// ═══ phrase suggester（短语纠错） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'suggest' => [
            'text' => 'elsaticsearch fen ci',
            'phrase_suggestion' => [
                'phrase' => [
                    'field' => 'title',
                    'size'  => 3,
                    'gram_size' => 2,
                    'direct_generator' => [
                        ['field' => 'title', 'suggest_mode' => 'always'],
                    ],
                ],
            ],
        ],
    ],
]);
```

### 8.3 效果最好的做法

```php
final class SmartSearchService
{
    public function searchWithSuggestions(string $query): array
    {
        $es = ESFactory::get();

        $result = $es->search([
            'index' => 'articles',
            'body'  => [
                // 查询
                'query' => [
                    'bool' => [
                        'must'   => [['match' => ['title' => ['query' => $query, 'fuzziness' => 'AUTO']]]],
                        'should' => [['match' => ['content' => ['query' => $query, 'boost' => 0.5]]]],
                    ],
                ],
                'size' => 20,
                // 高亮
                'highlight' => [
                    'fields' => ['title' => ['number_of_fragments' => 0]],
                ],
                // 纠错建议
                'suggest' => [
                    'spell' => [
                        'text' => $query,
                        'term'  => ['field' => 'title'],
                    ],
                ],
            ],
        ]);

        // 组装结果
        $suggestions = [];
        foreach ($result['suggest']['spell'][0]['options'] as $opt) {
            $suggestions[] = $opt['text'];
        }

        return [
            'hits'       => $result['hits']['hits'],
            'total'      => $result['hits']['total']['value'],
            'suggestions'=> $suggestions,
        ];
    }
}
```

---

## 10. 相似度与推荐

### 9.1 more_like_this（相似内容）

```php
// ═══ 基于某篇文章找相似文章 ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'more_like_this' => [
                'fields' => ['title', 'content'],
                'like'   => [
                    ['_index' => 'articles', '_id' => 1001],
                ],
                'min_term_freq'        => 1,
                'max_query_terms'      => 12,
                'min_doc_freq'         => 5,     // 太罕见的词忽略
                'max_doc_freq'         => 500,   // 太常见的词忽略（如"的"）
                'minimum_should_match' => '30%',
                'boost_terms'          => 1,
            ],
        ],
        // 排除自己
        'query' => [
            'bool' => [
                'must' => [
                    ['more_like_this' => [...],]
                ],
                'must_not' => [
                    ['ids' => ['values' => [1001]]],
                ],
            ],
        ],
    ],
]);

// ═══ 基于文本找相似（不需要已有文档） ═══
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'more_like_this' => [
                'fields' => ['title', 'content'],
                'like'   => 'Redis 缓存优化与高并发方案',
                'min_term_freq' => 1,
                'min_doc_freq'  => 3,
            ],
        ],
    ],
]);
```

### 9.2 用户协同过滤推荐

```php
/**
 * 基于用户阅读历史的推荐
 */
final class ContentRecommender
{
    private array $ignoredFields = ['view_count', 'created_at', 'updated_at'];
    private int $historyLimit = 20;

    /**
     * 获取用户最近阅读的文档ID
     */
    private function getUserHistory(int $userId): array
    {
        return DB::table('user_reads')
            ->where('user_id', $userId)
            ->orderBy('read_at', 'desc')
            ->limit($this->historyLimit)
            ->pluck('article_id')
            ->toArray();
    }

    /**
     * 根据相似内容推荐
     */
    public function recommendByContent(int $userId, int $size = 10): array
    {
        $historyIds = $this->getUserHistory($userId);
        if (empty($historyIds)) return [];

        $es = ESFactory::get();

        $result = $es->search([
            'index' => 'articles',
            'body'  => [
                'query' => [
                    'bool' => [
                        'must' => [
                            ['more_like_this' => [
                                'fields' => ['title', 'content', 'tags'],
                                'like'   => array_map(fn($id) => ['_index' => 'articles', '_id' => $id], $historyIds),
                                'min_term_freq' => 2,
                                'min_doc_freq'  => 3,
                                'minimum_should_match' => '25%',
                            ]],
                        ],
                        'must_not' => [
                            ['ids' => ['values' => $historyIds]],  // 排除已读
                        ],
                        'filter' => [
                            ['term' => ['is_published' => true]],
                        ],
                    ],
                ],
                'size' => $size,
                '_source' => $this->ignoredFields,
            ],
        ]);

        return $result['hits']['hits'];
    }

    /**
     * 按标签兴趣推荐
     */
    public function recommendByTags(int $userId, int $size = 10): array
    {
        // 1. 找用户最常读的标签
        $historyIds = $this->getUserHistory($userId);
        if (empty($historyIds)) return [];

        $es = ESFactory::get();

        // 2. 聚合历史文章中的标签
        $tagResult = $es->search([
            'index' => 'articles',
            'body'  => [
                'size' => 0,
                'query' => ['ids' => ['values' => $historyIds]],
                'aggs' => [
                    'top_tags' => [
                        'terms' => ['field' => 'tags', 'size' => 10],
                    ],
                ],
            ],
        ]);

        $favoredTags = array_map(
            fn($b) => $b['key'],
            $tagResult['aggregations']['top_tags']['buckets']
        );
        if (empty($favoredTags)) return [];

        // 3. 推荐这些标签下的热门文章
        $result = $es->search([
            'index' => 'articles',
            'body'  => [
                'query' => [
                    'function_score' => [
                        'query' => [
                            'bool' => [
                                'must' => [['terms' => ['tags' => $favoredTags]]],
                                'must_not' => [['ids' => ['values' => $historyIds]]],
                            ],
                        ],
                        'functions' => [
                            // 按分数加权
                            ['field_value_factor' => [
                                'field' => 'score',
                                'factor' => 1.2,
                                'modifier' => 'log1p',
                            ]],
                            // 按时间衰减（新文章权重高）
                            ['gauss' => [
                                'created_at' => [
                                    'origin' => 'now',
                                    'scale'  => '30d',
                                    'decay'  => 0.5,
                                ],
                            ]],
                        ],
                        'boost_mode' => 'multiply',
                    ],
                ],
                'size' => $size,
            ],
        ]);

        return $result['hits']['hits'];
    }
}
```

### 9.3 语义向量搜索（kNN）

```php
// 8.0+ 支持 dense_vector + kNN 搜索
// 需要先用模型生成 embedding（如 text-embedding-ada-002）

$es->search([
    'index' => 'articles',
    'body'  => [
        'knn' => [
            'field'         => 'title_embedding',  // dense_vector 字段
            'query_vector'  => [0.12, -0.34, 0.56, ...],  // 768维向量
            'k'             => 10,
            'num_candidates' => 100,
        ],
        '_source' => ['title', 'summary'],
    ],
]);

// 混合搜索：关键词 + 向量
$es->search([
    'index' => 'articles',
    'body'  => [
        'query' => [
            'bool' => [
                'must' => [
                    ['match' => ['title' => 'php']],
                ],
            ],
        ],
        'knn' => [
            'field'        => 'title_embedding',
            'query_vector' => $embedding,
            'k'            => 10,
            'num_candidates' => 50,
        ],
        'rank' => [
            'rrf' => [  // Reciprocal Rank Fusion: 融合关键词和向量排名
                'window_size' => 50,
            ],
        ],
    ],
]);
```

---

## 11. 地理位置搜索

```php
// ═══ 创建含坐标的文档 ═══
$es->index([
    'index' => 'shops',
    'id'    => 1,
    'body'  => [
        'name'     => '星巴克人民广场店',
        'location' => ['lat' => 31.23, 'lon' => 121.47],
        'rating'   => 4.5,
    ],
]);

// ═══ 半径搜索 ═══
$es->search([
    'index' => 'shops',
    'body'  => [
        'query' => [
            'bool' => [
                'must' => [
                    'match_all' => new \stdClass(),
                ],
                'filter' => [
                    'geo_distance' => [
                        'distance'      => '3km',
                        'location'      => ['lat' => 31.23, 'lon' => 121.47],
                        'distance_type' => 'plane',  // arc(更精确) / plane(更快)
                    ],
                ],
            ],
        ],
        'sort' => [
            '_geo_distance' => [
                'location' => ['lat' => 31.23, 'lon' => 121.47],
                'order'    => 'asc',
                'unit'     => 'km',
            ],
        ],
    ],
]);

// ═══ 矩形范围 ═══
$es->search([
    'index' => 'shops',
    'body'  => [
        'query' => [
            'bool' => [
                'filter' => [
                    'geo_bounding_box' => [
                        'location' => [
                            'top_left'     => ['lat' => 31.30, 'lon' => 121.40],
                            'bottom_right' => ['lat' => 31.20, 'lon' => 121.55],
                        ],
                    ],
                ],
            ],
        ],
    ],
]);
```

---

## 12. 自动补全

### 11.1 Completion Suggester

```php
// Mapping（创建索引时）
$es->indices()->create([
    'index' => 'search_suggestions',
    'body'  => [
        'mappings' => [
            'properties' => [
                'keyword' => [
                    'type' => 'text',
                    'fields' => [
                        'suggest' => [
                            'type'            => 'completion',
                            'analyzer'        => 'ik_smart',
                            'preserve_separators' => false,
                            'preserve_position_increments' => false,
                            'max_input_length' => 50,
                        ],
                    ],
                ],
            ],
        ],
    ],
]);

// 写入建议词
$es->index([
    'index' => 'search_suggestions',
    'body'  => [
        'keyword' => [
            'input'  => ['php教程', 'php入门', 'PHP Tutorial'],
            'weight' => 100,  // 权重越高越靠前
        ],
    ],
]);

// 按拼音输入时的建议词
$es->index([
    'index' => 'search_suggestions',
    'body' => [
        'keyword' => [
            'input'  => ['php教程', 'php', 'PHP', 'phpjiaocheng'],
            'weight' => 100,
        ],
    ],
]);

// ═══ 补全查询 ═══
$es->search([
    'index' => 'search_suggestions',
    'body'  => [
        '_source' => false,
        'suggest' => [
            'keyword_suggest' => [
                'prefix'     => 'php',     // 用户输入
                'completion' => [
                    'field'           => 'keyword.suggest',
                    'size'            => 10,
                    'skip_duplicates' => true,
                    'fuzzy'           => [
                        'fuzziness' => 1,       // 允许1个字符拼错
                    ],
                ],
            ],
        ],
    ],
]);

// 解析
foreach ($result['suggest']['keyword_suggest'][0]['options'] as $opt) {
    echo "{$opt['text']} (weight: {$opt['_score']})\n";
}
```

### 11.2 Context Suggester（上下文感知）

```php
// 按分类过滤的建议词
$es->indices()->create([
    'index' => 'context_suggestions',
    'body'  => [
        'mappings' => [
            'properties' => [
                'keyword' => [
                    'type' => 'completion',
                    'contexts' => [
                        ['name' => 'category', 'type' => 'category'],
                    ],
                ],
            ],
        ],
    ],
]);

// 写入
$es->index([
    'index' => 'context_suggestions',
    'body' => [
        'keyword' => [
            'input'    => ['php框架', 'laravel'],
            'contexts' => ['category' => ['编程']],
        ],
    ],
]);

// 搜索时按分类过滤
$es->search([
    'index' => 'context_suggestions',
    'body'  => [
        'suggest' => [
            'keyword_suggest' => [
                'prefix'     => 'ph',
                'completion' => [
                    'field' => 'keyword',
                    'contexts' => [
                        'category' => ['编程'],
                    ],
                ],
            ],
        ],
    ],
]);
```

---

## 13. 批量操作与性能

### 12.1 Bulk API

```php
// ═══ 批量写入 ═══
final class ESBulkService
{
    /**
     * 分批批量写入（推荐每批 500-1000 条）
     */
    public function bulkIndex(array $documents, int $batchSize = 500): void
    {
        $es = ESFactory::get();
        $params = ['index' => 'articles', 'body' => []];

        foreach ($documents as $i => $doc) {
            // 每行一个动作元数据
            $params['body'][] = [
                'index' => [
                    '_index' => 'articles',
                    '_id'    => $doc['id'],
                ],
            ];
            // 下一行是文档内容
            $params['body'][] = $doc;

            // 每 500 条发送一次
            if (($i + 1) % $batchSize === 0) {
                $response = $es->bulk($params);
                $this->checkBulkErrors($response);
                $params['body'] = [];
            }
        }

        // 剩余的也发送
        if (!empty($params['body'])) {
            $response = $es->bulk($params);
            $this->checkBulkErrors($response);
        }
    }

    private function checkBulkErrors(array $response): void
    {
        if ($response['errors']) {
            foreach ($response['items'] as $item) {
                $index = $item['index'] ?? $item['create'] ?? [];
                if (!empty($index['error'])) {
                    Log::warning('ES bulk error', [
                        'id'    => $index['_id'] ?? 'unknown',
                        'error' => $index['error'],
                    ]);
                }
            }
        }
    }

    /**
     * 从 MySQL 同步到 ES
     */
    public function syncFromMySQL(int $lastId = 0): int
    {
        $count = 0;
        DB::table('articles')
            ->where('id', '>', $lastId)
            ->orderBy('id')
            ->chunk(500, function ($articles) use (&$count) {
                $docs = $articles->map(fn($a) => [
                    'id'           => $a->id,
                    'title'        => $a->title,
                    'content'      => strip_tags($a->content),
                    'summary'      => $a->summary,
                    'tags'         => $a->tags ? explode(',', $a->tags) : [],
                    'category_id'  => $a->category_id,
                    'view_count'   => $a->view_count,
                    'is_published' => (bool) $a->status,
                    'created_at'   => $a->created_at,
                ])->toArray();

                $this->bulkIndex($docs);
                $count += count($docs);
            });

        return $count;
    }
}
```

### 12.2 Scroll（全量导出）

```php
// 适合：导出全部数据、重建索引
$params = [
    'index' => 'articles',
    'scroll' => '5m',
    'size'   => 1000,
    'body'   => ['query' => ['match_all' => new \stdClass()]],
];

$response = $es->search($params);
$scrollId = $response['_scroll_id'];

while (count($response['hits']['hits']) > 0) {
    foreach ($response['hits']['hits'] as $hit) {
        // 处理每条数据...
    }

    $response = $es->scroll([
        'scroll_id' => $scrollId,
        'scroll'    => '5m',
    ]);
}

// 释放 scroll
$es->clearScroll(['scroll_id' => $scrollId]);
```

### 12.3 Search After（深分页）

```php
// 比 from+size 快得多，适合无限滚动
$params = [
    'index' => 'articles',
    'size'  => 20,
    'body'  => [
        'query' => ['match' => ['title' => 'php']],
        'sort'  => [
            ['created_at' => 'desc'],
            ['id' => 'desc'],          // 必须有 tiebreaker
        ],
    ],
];

$response = $es->search($params);
$hits = $response['hits']['hits'];

// 下一页：用最后一条的 sort 值
if (count($hits) > 0) {
    $lastSort = end($hits)['sort'];
    $params['body']['search_after'] = $lastSort;
    $response = $es->search($params);
}
```

---

## 14. 实战：电商搜索

```php
/**
 * 电商商品搜索完整实现
 */
final class ProductSearchService
{
    /**
     * 综合搜索（全文 + 分类 + 价格 + 排序 + 高亮 + 聚合）
     */
    public function search(ProductSearchDTO $dto): array
    {
        $es = ESFactory::get();

        $body = [
            'query' => [
                'function_score' => [
                    'query' => $this->buildQuery($dto),
                    'functions' => $this->buildScoring($dto),
                    'boost_mode' => 'multiply',
                    'score_mode' => 'sum',
                ],
            ],
            'aggs' => $this->buildAggs(),
            'highlight' => [
                'pre_tags'  => ['<em>'],
                'post_tags' => ['</em>'],
                'fields' => [
                    'name' => ['number_of_fragments' => 0],
                ],
            ],
            'sort'   => $this->buildSort($dto),
            'from'   => ($dto->page - 1) * $dto->size,
            'size'   => $dto->size,
            '_source' => ['id', 'name', 'price', 'image', 'sales', 'rating', 'shop_name'],
        ];

        $result = $es->search(['index' => 'products', 'body' => $body]);

        return $this->formatResult($result, $dto);
    }

    private function buildQuery(ProductSearchDTO $dto): array
    {
        $must = [];
        $filter = [];

        // 关键词搜索
        if ($dto->keyword) {
            $must[] = [
                'multi_match' => [
                    'query'  => $dto->keyword,
                    'fields' => ['name^3', 'description^2', 'brand^2', 'tags'],
                    'type'   => 'best_fields',
                    'fuzziness' => 'AUTO',
                ],
            ];
        }

        // 分类
        if ($dto->categoryId) {
            $filter[] = ['term' => ['category_id' => $dto->categoryId]];
        }

        // 价格区间
        if ($dto->minPrice || $dto->maxPrice) {
            $range = [];
            if ($dto->minPrice) $range['gte'] = $dto->minPrice;
            if ($dto->maxPrice) $range['lte'] = $dto->maxPrice;
            $filter[] = ['range' => ['price' => $range]];
        }

        // 品牌
        if ($dto->brand) {
            $filter[] = ['term' => ['brand.keyword' => $dto->brand]];
        }

        // 标签筛选
        if (!empty($dto->tags)) {
            foreach ($dto->tags as $tag) {
                $filter[] = ['term' => ['tags' => $tag]];
            }
        }

        // 有货
        if ($dto->inStock) {
            $filter[] = ['range' => ['stock' => ['gt' => 0]]];
        }

        return ['bool' => ['must' => $must, 'filter' => $filter]];
    }

    private function buildScoring(ProductSearchDTO $dto): array
    {
        $functions = [];

        // 销量加权
        $functions[] = [
            'field_value_factor' => [
                'field'    => 'sales',
                'factor'   => 0.001,
                'modifier' => 'log1p',  // log(1 + sales*0.001)
                'missing'  => 0,
            ],
        ];

        // 评分加权
        $functions[] = [
            'field_value_factor' => [
                'field'  => 'rating',
                'factor' => 0.2,
                'missing' => 3,
            ],
        ];

        // 新品加权（30天内权重1.5倍）
        $functions[] = [
            'gauss' => [
                'created_at' => [
                    'origin' => 'now',
                    'scale'  => '30d',
                    'decay'  => 0.5,
                ],
            ],
        ];

        return $functions;
    }

    private function buildAggs(): array
    {
        return [
            'categories' => [
                'terms' => ['field' => 'category_id', 'size' => 20],
            ],
            'brands' => [
                'terms' => ['field' => 'brand.keyword', 'size' => 20],
            ],
            'price_ranges' => [
                'range' => [
                    'field'  => 'price',
                    'ranges' => [
                        ['to' => 100],
                        ['from' => 100, 'to' => 500],
                        ['from' => 500, 'to' => 1000],
                        ['from' => 1000, 'to' => 5000],
                        ['from' => 5000],
                    ],
                ],
            ],
        ];
    }

    private function buildSort(ProductSearchDTO $dto): array
    {
        return match ($dto->sort) {
            'sales'     => [['sales'    => 'desc'], '_score' => 'desc'],
            'price_asc' => [['price'    => 'asc'],  '_score' => 'desc'],
            'price_desc'=> [['price'    => 'desc'], '_score' => 'desc'],
            'rating'    => [['rating'   => 'desc'], '_score' => 'desc'],
            'newest'    => [['created_at' => 'desc'], '_score' => 'desc'],
            default     => [['_score'   => 'desc']],
        };
    }

    private function formatResult(array $result, ProductSearchDTO $dto): array
    {
        $hits = [];
        foreach ($result['hits']['hits'] as $hit) {
            $row = $hit['_source'];
            $row['_score'] = $hit['_score'];
            $row['highlight'] = $hit['highlight'] ?? [];
            $hits[] = $row;
        }

        return [
            'products'   => $hits,
            'total'      => $result['hits']['total']['value'],
            'page'       => $dto->page,
            'size'       => $dto->size,
            'took_ms'    => $result['took'],
            'aggs'       => [
                'categories'   => $result['aggregations']['categories']['buckets'] ?? [],
                'brands'       => $result['aggregations']['brands']['buckets'] ?? [],
                'price_ranges' => $result['aggregations']['price_ranges']['buckets'] ?? [],
            ],
        ];
    }
}
```

---

## 15. 运维与调优

### 14.1 集群健康

```php
// 集群健康
$es->cat()->health(['v' => true]);   // green / yellow / red

// 节点信息
$es->cat()->nodes(['v' => true, 'h' => 'name,heap.percent,ram.percent,cpu,load_1m']);

// 索引状态
$es->cat()->indices(['v' => true, 's' => 'store.size:desc']);

// 分片分配
$es->cat()->shards(['v' => true, 'index' => 'articles']);

// 热点线程
$es->nodes()->hotThreads([]);
```

### 14.2 索引优化

```json
// 优化段（合并小段成大段）
POST /articles/_forcemerge?max_num_segments=1&only_expunge_deletes=true

// 刷新（让文档可搜索，消耗 I/O）
POST /articles/_refresh

// 清除缓存
POST /articles/_cache/clear

// 查看段
GET /articles/_segments
```

```php
$es->indices()->forceMerge(['index' => 'articles', 'max_num_segments' => 1]);
$es->indices()->refresh(['index' => 'articles']);
$es->indices()->clearCache(['index' => 'articles']);
```

### 14.3 性能调优 checklist

```
硬件：
□ 内存 ≥ 系统总数据的 50%（ES 堆最大 32GB，留给 OS cache 至少 50%）
□ SSD 磁盘（机械盘很慢）
□ 多节点集群

索引设计：
□ 合理分片数（单分片 10-50GB）
□ 关闭不需要的 _source
□ text 字段不需要 keyword 子字段时不建
□ 不需要评分的查询用 filter context
□ 日期/数字用对应类型，别当字符串

搜索：
□ 能用 filter 就用 filter（有缓存，不评分）
□ 深分页用 search_after 而不是 from/size
□ 少用 wildcard / regexp / fuzzy（慢）
□ highlight 只 highlighted 需要展示的字段
□ 聚合避免 size=0 不返回文档

写入：
□ bulk 批量（每批 500-1000 条）
□ 大批量写入时临时调大 refresh_interval: -1
□ 大批量写入时关闭副分片: number_of_replicas=0
□ 使用自动生成的 ID（比指定 ID 快）
```

### 14.4 监控告警

```php
final class ESMonitor
{
    public function checkHealth(): array
    {
        $es = ESFactory::get();
        $health = $es->cluster()->health();

        $alerts = [];

        if ($health['status'] !== 'green') {
            $alerts[] = "集群状态: {$health['status']}";
        }

        if ($health['unassigned_shards'] > 0) {
            $alerts[] = "未分配分片: {$health['unassigned_shards']}";
        }

        // 检查节点堆内存
        $nodes = $es->cat()->nodes(['v' => true, 'h' => 'heap.percent']);
        // 堆内存 > 75% 告警

        if (!empty($alerts)) {
            Log::warning('ES 集群警告', $alerts);
        }

        return ['status' => $health['status'], 'alerts' => $alerts];
    }
}
```

---

> 📝 **ES 不是数据库，是搜索引擎。数据的主存储永远是 MySQL，ES 做辅助搜索和分析。同步策略要设计好，出了问题能从 MySQL 全量重建。**
