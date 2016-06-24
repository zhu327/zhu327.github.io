---
date: 2016-05-30T09:51:22+08:00
title: Django+Elasticsearch实现搜索功能
tags: ["haystack", "elasticsearch"]
---

在项目中实现了资讯搜索功能，用到了Django Tastypie haystack Elasticsearch ik分词，覆盖了我对搜索了解的所有姿势。其实也就了解一些简单的概念，不过haystack+elasticsearch并不需要太多搜索基础，只要看看haystack文档就就能实现简单的搜索需求了。

### 搭建Elasticsearch环境

> <http://www.sojson.com/blog/81>  

参考以上文档搭建了自己的ES环境，然后再安装Django实现搜索的相关库:

```
django-haystack==2.4.1
elasticsearch==2.3.0
elasticstack==0.4.1
```

haystack不用说，在Django中实现搜索基本上都会用到，因为要用到Elasticsearch的BackEnds，所以要装elasticsearch，最后elasticstack会在下面说到。

### haystack使用ik分词

在elasticsearch中可以在创建了indexs后可以在settings中自定义analyzer，在创建types时可以设置各个field使用的analyzer，但是在haystack中就比较麻烦了。

> <https://github.com/django-haystack/django-haystack/issues/639>  

从上面的issues中可以看出，如果需要解决这个问题，需要重写elasticsearch backends中相关设置，在查看了[elasticsearch backends](https://github.com/django-haystack/django-haystack/blob/master/haystack/backends/elasticsearch_backend.py)代码后，发现除了`DEFAULT_SETTINGS`中配置自定义analyzer，`DEFAULT_FIELD_MAPPING`与`FIELD_MAPPINGS`分别用于设置field在mapping中analyzer，重写相关配置与方法可达到定义不同字段的分词器。

~~但是重写配置方法还是太麻烦了，能不能更简单点呢，所以找到了elasticstack，用上了这个工具后，配置默认的analyzer就非常简单了。~~

```python
HAYSTACK_CONNECTIONS = {
    'default': {
        'ENGINE': 'elasticstack.backends.ConfigurableElasticSearchEngine',
        'URL': 'http://127.0.0.1:9200/',
        'INDEX_NAME': 'haystack',
    },
}

ELASTICSEARCH_DEFAULT_ANALYZER = 'ik' # 设置默认分词器为ik
```

<!--more-->
### 搜索实例

现有资讯Model:

```python
import uuid
from django.db import models

class Article(models.Model):
    """
    资讯
    """

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, db_column='c_id')
    add_dt = models.DateTimeField(db_column='c_add_dt', auto_now_add=True)
    update_dt = models.DateTimeField(db_column='c_update_dt', blank=True, null=True)
    title = models.CharField('标题', max_length=100, db_column='c_title')
    summary = models.CharField('摘要', max_length=400, blank=True, null=True, db_column='c_summary')
    content = models.TextField('内容', db_column='c_content')
```

创建索引对象:

```python
from haystack import indexes
from models import Article


class ArticleIndex(indexes.SearchIndex, indexes.Indexable):
    text = indexes.CharField(document=True, use_template=True)
    publish_dt = indexes.DateTimeField(model_attr='publish_dt')
    title_auto = indexes.NgramField(model_attr='title')

    def get_model(self):
        return Article

    def index_queryset(self, using=None):
        return self.get_model().all()
```

全文索引模板:

```html
{{ object.title }}
{{ object.content }}
```

搜索api实现:

```python
from django.conf.urls import url
from tastypie.paginator import Paginator
from tastypie.exceptions import BadRequest
from tastypie.resources import ModelResource

from haystack.query import SearchQuerySet, EmptySearchQuerySet
from haystack.inputs import AutoQuery

from models import Article

class ArticleResource(ModelResource):
    '''
    文章
    '''

    class Meta:
        queryset = Article.objects.order_by('-publish_dt')
        resource_name = 'article'
        fields = ['id', 'add_dt', 'update_dt', 'title', 'content']
        list_allowed_methods = ['get']
        detail_allowed_methods = ['get']
        paginator_class = Paginator
        include_resource_uri = False
        always_return_data = True

    def alter_list_data_to_serialize(self, request, data):
        for obj in data['objects']:
            del obj.data['content']
        return data

    def obj_get(self, bundle, **kwargs):
        obj = super(ArticleResource, self).obj_get(bundle, **kwargs)
        obj.__class__.objects.filter(pk=obj.pk).update(read_count=F('read_count')+1)
        return obj

    def prepend_urls(self):
        return [
            url(r"^(?P<resource_name>%s)/search/$" % self._meta.resource_name, self.wrap_view(self.get_search), name="search"),
            url(r"^(?P<resource_name>%s)/autocomplete/$" % self._meta.resource_name, self.wrap_view(self.autocomplete), name="autocomplete"),
		]

    def get_search(self, request, **kwargs):
        '''
        搜索api
        '''
        query = request.GET.get('q', None)
        if not query:
            raise BadRequest('Please supply the search parameter (e.g. "/v1/article/search/?q=css")')

        qs = SearchQuerySet().models(Article).order_by('-publish_dt')
        results = qs.auto_query(query)

        if not results:
            results = EmptySearchQuerySet()
        
        paginator = Paginator(request.GET, results, resource_uri='/v1/article/search/')

        bundles = []
        for result in paginator.page()['objects']:
            bundle = self.build_bundle(obj=result.object, request=request)
            bundles.append(self.full_dehydrate(bundle))

        data = paginator.page()
        data['objects'] = bundles
        data['search'] = {
            'q': query,
        }

        data = self.alter_list_data_to_serialize(request, data)

        return self.create_response(request, data)

    def autocomplete(self, request, **kwargs):
    	  '''
        搜索关键词建议，只输出资讯的title
        '''
        query = request.GET.get('q', None)
        if not query:
            raise BadRequest('Please supply the search parameter (e.g. "/v1/article/search/?q=css")')

        qs = SearchQuerySet().models(Article).order_by('-publish_dt')
        results = qs.autocomplete(title_auto=query)[:5]
        suggestions = [result.title_auto for result in results]
        return self.create_response(request, suggestions)
```

### 遇到的问题

在使用过程中发现elasticsearch上的日志一直报:

> [rest.suppressed          ] /haystack_ehr_dev/_mapping/modelresult Params: {index=haystack_ehr_dev, type=modelresult}
MapperParsingException[Root mapping definition has unsupported parameters:  [_boost : {name=boost, null_value=1.0}]]

发现是因为Elasticsearch在2.0版本以后不再支持_boost这个东东，所以解决办法是重写haystack backsends里面的mapping生成逻辑或者用github上haystack当前的最新的版本。具体见下面问题的第一个回答:

> <http://stackoverflow.com/questions/35443179/django-haystack-locationfield-created-as-string-instead-of-geo-point-in-elastics>  

### ~~关于拼音搜索~~

在上面实现的api中，实现了一个简单搜索建议的api，本来计划是在获取建议标题时支持拼音搜索的，但是有一些分词与mapping上的问题，暂时还未实现。

安装pinyin分词器:

> <http://my.oschina.net/UpBoy/blog/625014?fromerr=mRvT8rzk>

同时使用ik分词与pinyin

> <https://github.com/medcl/elasticsearch-analysis-pinyin/issues/27>

在settings中定义ik_pinyin分词器

```python
 ELASTICSEARCH_INDEX_SETTINGS = {
     'settings': {
         "analysis": {
             "analyzer": {
                 "ik_pinyin" : {
                     "type": "custom",
                     "tokenizer" : "ik_smart",
                     "filter" : ["my_pinyin","word_delimiter"]
                 },
             },
             "filter": {
                 "my_pinyin" : {
                     "type" : "pinyin",
                     "first_letter" : "none",
                     "padding_char" : ""
                 }
             }
         }
     }
 }
```

### 关于Whoosh backends

之前也用haystack+Whoosh+jieba实现过全文检索，基本上与别人总结的方案一致:

> <http://chenyvehtung.github.io/2015/09/11/django-haystack-search.html>  

但是也遇到过2个Whoosh的坑，同样的问题在Elasticsearch上就没有碰到过

1. 定义了多个Model的索引后，SearchQueySet结果出现None
  > <http://stackoverflow.com/questions/10454367/haystack-queryset-contains-none-elements>  
  同上面的问题，不知道是什么原理，但是第二个回答好像能解决问题
2. 更新索引时不能删除已删除的文章

```python
# 会像这样来定时更新索引
from django.core.management import call_command
call_command('update_index', remove=True)
```

但是实际上remove=True并没有作用，规避方法是定义索引类的时候重写read_queryset方法，使被删除的文章也能搜索出来。

### 参考

> <http://www.sojson.com/blog/81>  
> <http://my.oschina.net/UpBoy/blog/625014?fromerr=mRvT8rzk>  
> <http://django-tastypie.readthedocs.io/en/latest/cookbook.html>  
> <https://gist.github.com/skakri/9090925>  
> <http://django-haystack.readthedocs.io/en/v2.4.1/autocomplete.html>  
> <http://stackoverflow.com/questions/35443179/django-haystack-locationfield-created-as-string-instead-of-geo-point-in-elastics>  
> <https://github.com/django-haystack/django-haystack/issues/639>  
> <https://github.com/medcl/elasticsearch-analysis-pinyin/issues/27>  
> <http://chenyvehtung.github.io/2015/09/11/django-haystack-search.html>  
> <http://stackoverflow.com/questions/10454367/haystack-queryset-contains-none-elements>