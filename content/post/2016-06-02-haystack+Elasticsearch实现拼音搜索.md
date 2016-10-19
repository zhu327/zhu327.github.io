---
date: 2016-06-02T11:05:17+08:00
title: haystack+Elasticsearch实现拼音搜索
tags: ["haystack", "elasticsearch"]
---

前一篇[Django+Elasticsearch实现搜索功能](https://zhu327.github.io/2016/05/30/djangoelasticsearch%E5%AE%9E%E7%8E%B0%E6%90%9C%E7%B4%A2%E5%8A%9F%E8%83%BD/)已经实现了搜索的基本功能，但是其实还是有一些错误，这里先纠正一下。

1. 以为通过elsticstack实现了默认的分词器
   通过查看elasticsearch types的mapping发现string类型的根本就没有加载ik analyzer
2. 拼音分词方式设置错误
   拼音分词的尝试过程中，终于实现了全拼分词

### 环境搭建

Elasticsearch安装拼音分词:

> <http://my.oschina.net/UpBoy/blog/625014?fromerr=mRvT8rzk>

Django依赖安装:

```
django-haystack==2.5.dev1
```

之前会使用elasticstack与haystack2.4配合，发现问题比较多，所以删掉elasticstack，升级haystack为github上的最新版本

<!--more-->

### Backends

```python
# coding: utf-8

from haystack.backends import elasticsearch_backend

elasticsearch_backend.DEFAULT_FIELD_MAPPING = {'type': 'string', 'analyzer': 'ik'} # 重写string类型的默认analyzer为ik分词

class EhrElasticsearchSearchBackend(elasticsearch_backend.ElasticsearchSearchBackend):
    DEFAULT_SETTINGS = { # 设置ngram_analyzer分词为拼音，用于联想标题
        'settings': {
            "analysis": {
                "analyzer": {
                    "ngram_analyzer": {
                        "type": "custom",
                        "tokenizer": "ik",
                        "filter": [
                            "full_pinyin",
                        ]
                    },
                    "edgengram_analyzer": {
                        "type": "custom",
                        "tokenizer": "ik",
                        "filter": [
                            "first_letter_pinyin",
                        ]
                    },
                },
                "filter": {
                    "full_pinyin": {
                        "type": "pinyin",
                        "first_letter": "none",
                        "padding_char": ""
                    },
                    "first_letter_pinyin" : {
                        "type" : "pinyin",
                        "first_letter" : "only",
                        "padding_char" : ""
                    }
                }
            }
        }
    }


class EhrElasticsearchSearchEngine(elasticsearch_backend.ElasticsearchSearchEngine):
    backend = EhrElasticsearchSearchBackend
```

从自定义的analyzer `ngram_analyzer`的配置中可以看出使用先使用了ik分词来做基本的分词，然后使用pinyin filter把ik分出来的词元转成全拼。

### 拼音分词的配置

> <https://github.com/medcl/elasticsearch-analysis-pinyin>

从以上说明中可以看出，可以选择是否简拼(首字母组合)，以及是否使用`padding_char`分割出来的每个拼音字符，在配置`padding_char`为空格的时候可以配合`word_delimiter`这个filter把词元通过空格分割成更小的词元。

在定义analyzer时如果同时使用了多个filter，实际上多个filter是顺序执行了，并不是并行执行

> <https://github.com/medcl/elasticsearch-analysis-pinyin/issues/27>

从上面的问题可以看出，@chengyuanheng 尝试配置出一个analyzer同时解析出一个经过ik分词词元的全拼与简拼，比如 南京: nanjing nj，但是最后只有nanjing被分出来，就是因为filter是顺序执行的。

使用拼音分词主要的目的是实现搜索建议功能，这样在搜索资讯的时候可以自动联想出资讯的标题。最终的目标就是通过中文词元，全拼词元，简拼词元都能联想出想要的资讯标题。但是在网上一直没找到同时平行使用多个filter的方法，当前只实现了全拼分词。已经尝试发邮件给@Medcl 大神描述了这个问题，在大神回复邮件前应该是没什么办法解决了。

在实践的过程中对一些搜索的概念有了更多的了解，通过不同的analyzer组合尝试了很多的分词方式，最终也没有解决简拼的问题，但是解决了基本的拼音联想，以及在重写haystack elasticsearch backends的过程中对ES的配置也学习很多。

### 补充

2016-10-19 同时支持全拼简拼方案

修改了EhrElasticsearchSearchBackend中的settings部分，使haystack中的`EdgeNgramField`支持ik分词后解析出简拼词元，对于需要同时支持全拼与简拼搜索的字段，比如文章的标题，同时创建全拼,简拼索引，搜索时使用或，以下有示例

```python
from haystack import indexes
from .models import Article


class ArticleIndex(indexes.SearchIndex, indexes.Indexable):
    text = indexes.CharField(document=True, model_attr='content', null=True)
    title_full = indexes.NgramField(model_attr='title', null=True) # 创建标题的全拼索引字段
    title_lite = indexes.EdgeNgramField(model_attr='title', null=True) # 简拼索引字段

    def get_model(self):
        return Article


from haystack.query import SQ
from haystack.query import SearchQuerySet, EmptySearchQuerySet
from haystack.inputs import AutoQuery

# 搜索北京
query = u'北京'
queryset = SearchQuerySet().filter(SQ(text=AutoQuery(query))|SQ(title_full=query)|SQ(title_lite=query))

# 搜索beijing
query = u'beijing'
queryset = SearchQuerySet().filter(SQ(text=AutoQuery(query))|SQ(title_full=query)|SQ(title_lite=query))

# 搜索bj
query = u'bj'
queryset = SearchQuerySet().filter(SQ(text=AutoQuery(query))|SQ(title_full=query)|SQ(title_lite=query))
```

实现了同时支持 汉字 全拼 简拼 搜索结果。