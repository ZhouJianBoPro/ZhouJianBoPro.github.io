---
layout: post
title: Elasticsearch使用指南
date: 2025-05-20
tags: [elasticsearch]
---

#### 什么是Elasticsearch
> Elasticsearch是一个开源的分布式搜索和分析引擎，基于Lucene，专为处理大规模数据而设计，广泛应用于 全文检索、日志分析

#### 核心作用
1. 全文检索
   - 支持高效的文本搜索，如电商商品搜索 
   - 提供分词、模糊匹配、高亮显示、相关性排序等功能
2. 实时数据分析
   - 支持对海量数据进行聚合分析（Aggregation），如统计访问量、用户行为分析等
   - 常用于日志分析、监控系统、BI 报表生成等场景

#### Elasticsearch核心概念
1. 索引 (Index)：文档的集合，类似关系型数据库中的表，索引名称必须小写
2. 映射 (Mapping)：定义了 Index 中字段的类型、格式、是否被索引等规则，类似于数据库的 schema
3. 文档 (Document)：基本数据单元，相当于关系型数据库中的一行记录，采用 JSON 格式存储，每个文档有唯一 ID（可自动生成或指定）
4. Aggregation（聚合）：提供类似 SQL 中的 GROUP BY 和统计功能，用于数据分析
5. Shard（分片）
   - 是底层数据的物理存储单元，每个 Index 可被分成多个 Shard
   - 分片机制支持水平扩展和分布式存储
   - 有两类分片：Primary Shard（主分片） 和 Replica Shard（副本分片）
6. Replica（副本）
   - 是主分片的拷贝，用于提高查询性能和容灾恢复
   - 副本分片可以处理读请求，提升并发能力。
7. Node（节点）：多个 Node 组成一个 Cluster（集群）

#### 核心数据类型
1. text：用于全文检索的可分析字符串
2. keyword：用于精确值匹配的字符串(如ID、状态码)
3. 数值类型：long，integer，byte，double,float等
4. 日期类型：date，可以存储日期或时间戳，支持多种格式
5. 布尔类型：boolean，接受 true/false
6. 二进制类型：binary，存储Base64编码的二进制数据
7. 范围类型：integer_range，double_range等
8. 对象类型：object，用于单个JSON对象

#### 分析器的核心作用
- 分词(Tokenization)：将连续文本拆分为独立的词项(tokens)，示例："华为智能手机" → ["华为", "智能", "手机", "智能手机"]
- 标准化(Normalization)：大小写统一（如"iPhone" → "iphone"），字符转换（如"é" → "e"），格式统一（如"500g" → "500 g"）
- 语义优化(Semantic Enhancement)：词干提取（如"running" → "run"），同义词扩展（如"手机" ↔ "智能手机"），停用词过滤（移除"的"、"a"等无意义词）

#### 分析器工作流程
1. 原始文本： "华为Mate40 Pro 5G手机"
2. 字符过滤： 移除特殊字符（可选）
3. 分词阶段： ["华为", "Mate40", "Pro", "5G", "手机"]
4. 词项过滤：
   - 转小写：["华为", "mate40", "pro", "5g", "手机"]
   - 同义词扩展：["华为", "mate40", "pro", "5g", "手机", "智能手机"]
   - 拼音处理：["huawei", "mate40", "pro", "5g", "shouji"]


#### 索引（Index）操作
- 创建index：PUT /index_name，以下索引设置number_of_shards（主分片数量），number_of_replicas（每个主分片对应副本分片数量），并对索引字段设置了分析器和过滤器
```javascript
{
  "sys_user": {
    "aliases": {},
    "mappings": {
      "properties": {
        "create_by": {
          "type": "text"
        },
        "create_time": {
          "type": "date",
          "format": "yyyy-MM-dd HH:mm:ss"
        },
        "id": {
          "type": "keyword"
        },
        "realname": {
          "type": "text",
          "analyzer": "sys_user_analyzer"
        },
        "update_by": {
          "type": "text"
        },
        "username": {
          "type": "text"
        }
      }
    },
    "settings": {
      "index": {
        "routing": {
          "allocation": {
            "include": {
              "_tier_preference": "data_content"
            }
          }
        },
        "number_of_shards": "3",
        "provided_name": "sys_user",
        "creation_date": "1747799845329",
        "analysis": {
          "analyzer": {
            "sys_user_analyzer": {
              "filter": [
                "lowercase",
                "stop",
                "unique",
                "stemmer"
              ],
              "type": "custom",
              "tokenizer": "standard"
            }
          }
        },
        "number_of_replicas": "1",
        "uuid": "YpA0cmeaSdmCBch3vX-itQ",
        "version": {
          "created": "8500003"
        }
      }
    }
  }
}
```
- 检查索引是否存在：HEAD /index_name
- 获取索引信息：GET /index_name
- 删除索引：DELETE /index_name

#### 映射（Mapping）操作
- 查看已有索引的映射：GET /index_name/_mapping
- 更新/删除映射（不支持更改已有字段的映射）
- 新增字段映射：PUT /index_name/_mapping
```javascript
{
  "properties": {
    "update_by": {
      "type": "text"
    }
  }
}
```
  
#### 文档（Document）操作
- 新增文档（_id可指定或自动生成）：POST /index_name/_doc/_id
    ```javascript
    {
      "create_by": "admin",
      "create_time": "2025-05-21 13:10:11",
      "id": "1922863724834213890",
      "realname": "ES简单使用",
      "update_by": "admin",
      "username": "admin"
    }
    ```
- 根据_id查询文档：GET /index_name/_doc/_id
- 按条件查询文档（match）：GET /index_name/_search，适用于text类型字段
    ```javascript
    {
      "query": {
        "match": {
          "realname": "es"
        }
      }
    }
    ```
- 精确查询文档（term/terms）：GET /index_name/_search，适用于keyword类型字段。text类型字段如果要实现精确查询，需要在该字段设置keyword类型的子字段
    ```javascript
    {
        "query": {
            "term": {
                "id": "1922863724834213890"
            }
        }
    }
    ```
- 通过ID删除文档：DELETE /index_name/_doc/_id
- 按条件精确删除文档（term/terms）：POST /index_name/_delete_by_query
    ```javascript
    {
      "query": {
        "term": {
          "id": "1922863724834213890"
        }
      }
    }
    ```
- 通过ID更新文档：POST /index_name/_update/_id
    ```javascript
    {
      "doc": {
        "create_by": "new_admin"
      }
    }
    ```
- 按条件更新文档：POST /index_name/_update_by_query
    ```javascript
    {
      "script": {
        "source": "ctx._source.realname = '新名称'; ctx._source.username = 'new_admin'"
      },
      "query": {
        "term": {
          "id": "1922863724834213890"
        }
      }
    }
    ```
  
#### 分页查询
- 按条件简单分页查询（from/size不适合深度分页）：GET /index_name/_search，可以获取数据总量及页码
    ```javascript
    {
        "from": 0,
        "size": 5,
        "query": {
            "match": {
                "realname": "es"
            }
        }
    }
    ```
- 按条件深度分页查询（search_after）:GET /index_name/_search，需要把上一页key传过来，不支持跳页，同时也不能获取数据总量及页码
    ```javascript
    {
        "size": 5,
        "query": {
            "match": {
                "realname": "es"
            }
        },
        "sort": [
            {
                "id": "desc"
            }
        ],
        "search_after": [
            "1922863724834213895"
        ]
    }
    ```
- 聚合分页并求和（terms 聚合 + sum 求和 + after_key 分）：GET /sys_user/_search
    ```javascript
    {
      "size": 0,
      "query": {
        "match": {
          "realname": "es"
        }
      },
      "aggs": {
        "group_by_username": {
          "terms": {
            "field": "username.keyword",
            "size": 5,
            "order": {
              "total_price": "desc"
            },
            "after": {
              "key": "user_5"
            }
          },
          "aggs": {
            "total_price": {
              "sum": {
                "field": "price"
              }
            }
          }
        }
      }
    }
    ```

- 按条件全量导出数据（scroll）
  1. 初始查询：POST /sys_user/_search?scroll=2m，获取第一页数据并返回了scroll_id
      ```javascript
      {
        "query": {
          "match": {
            "realname": "ES"
          }
        },
        "size": 1000
      }
      ```
  2. 获取下一页数据（传入上一页返回的scroll_id）：POST /_search/scroll，返回下一页数据，循环执行直到无数据返回
        ```javascript
        {
          "scroll": "2m",
          "scroll_id": "FGluY2x1ZGVfY29udGV4dF91dWlkDnF1ZXJ5VGhlbkZldGNoAxY5bHd5ZlptNFQ2U1JNSWFKeHlqV1BnAAAAAAABWr0WNjBjTWhFMnBTS2F4QzE0ZnhSYW1mURY5bHd5ZlptNFQ2U1JNSWFKeHlqV1BnAAAAAAABWr4WNjBjTWhFMnBTS2F4QzE0ZnhSYW1mURY5bHd5ZlptNFQ2U1JNSWFKeHlqV1BnAAAAAAABWr8WNjBjTWhFMnBTS2F4QzE0ZnhSYW1mUQ=="
        }
        ```
  3. 删除scroll_id：DELETE /_search/scroll
      ```javascript
      {
        "scroll_id": "<your_scroll_id>"
      }
      ```


