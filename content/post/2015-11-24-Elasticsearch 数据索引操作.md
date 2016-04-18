---
date: 2015-11-24T23:36:50+08:00
title: Elasticsearch 数据索引操作
tags: ["elasticsearch"]
---

开始一系列的搜索相关的学习，包括并不限于  
Django  
Haystack  
Elasticsearch  
IK中文分词  
分词字典  


## 简单搜索

<!--more-->
### 1. DSL搜索

```javascript
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

### 2. 过滤搜索

```javascript
{
    "query" : {
        "filtered" : {
            "filter" : {
                "range" : {
                    "age" : { "gt" : 30 } <1>
                }
            },
            "query" : {
                "match" : {
                    "last_name" : "smith" <2>
                }
            }
        }
    }
}
```

`filter`表示过滤条件，这里会过滤掉age大于30
`range`是区间过滤器

### 3. 全文搜索

```javascript
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
```

匹配`about`字段，中与"rock climbing"相似的文档

### 4.短语搜索

```javascript
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
```

只匹配与"rock climbing"相同的文档

### 5. 高亮

```javascript
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
```

## Elastic索引

### 创建索引
POST /indeces/types/ 自增生成id

PUT /indeces/types/id/ 指定id

### 获取数据
GET /indeces/types/id/

GET /indeces/types/id/?_source=title 只获取对象的title字段，多个字段用,分隔

GET /indeces/types/id/?_source 只要对象的数据，不要其它的元数据

### 是否存在
HEAD /indeces/types/id/ 200存在，404不存在

### 更新
PUT /indeces/types/id/ 可以完全覆盖更新，数据的`_version`值会+1，`created`如果为true，表示为新建，false表示为覆盖更新

### 创建判断
PUT /indeces/types/id/_create
PUT /indeces/types/id/?op_type=create

如果同id的对象已存在会报错409

### 删除
DELETE /indeces/types/id/ 200删除成功，404资源不存在，如果200，对象的`_version`会+1

### 局部更新
POST /website/blog/1/_update

```javascript
{
    "doc": {
        field: // 需要更新的字段
    }
}
```

## Mget

```javascript
GET /_mget
{
    "docs": [
        {
            "_indeces":
            "_types":
            "_id":
        },
        {
            "_indeces":
            "_types":
            "_id":
            "_source": []
        },
        ...
    ]
}
```
返回的数据顺序同请求的数据

```js
GET /indeces/types/_mget // 可以自定mget的默认indeces与types
```

## 批量操作
POST /_bulk

```js
{ action: { metadata }}
{ request body        }
```
cation delete create index(创建或更新) update(局部更新)
