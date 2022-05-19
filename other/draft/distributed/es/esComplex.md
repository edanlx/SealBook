default 是索引和搜索时用的默认的 analyzer，default_index 是索引时用的默认的 analyzer ， default_search 是查询时用的默认analyzer。
- 通过inner_hits实现折叠内容里面的排序
```json
{
    "collapse": {
        "field": "traceId",
        "inner_hits": 
            {
                "name": "resultCode",
                "size":1,
                "sort": [
                    {
                        "resultCode.keyword": {
                            "order": "desc"
                        }
                    }
                ]
            }
        
    },
    "sort": [
        {
            "createdTime": "desc"
        }
    ]
}
```
- 折叠后的统计total
```json
{
    "query": {
        "term": {
            "index": "92f68e00-bbb1-11ec-96ce-ebe28d2fb59b"
        }
    },
    "collapse": {
        "field": "traceId"
    },
    "aggs": {
        "totalSize": {
            "cardinality": {
                "field": "traceId"
            }
        }
    }
}
```
- 复合查询
```json
{
  "query": {
    "bool": {
      "must": [
        {
            "term": {
                "index": "88888-bbb1-11ec-96ce-ebe28d2fb59b"
            }
        },
        {
            "term": {
                "index": "92f68e00-bbb1-11ec-96ce-ebe28d2fb59"
            }
        }
      ]
    }
  }
}

```
- 指定字段类型
```json
{
        "aliases": {},
        "mappings": {
            "properties": {
                "content": {
                    "type": "keyword"
                },
                "createdTime": {
                    "type": "long"
                },
                "index": {
                    "type": "text",
                    "analyzer": "whitespace"
                },
                "label": {
                    "type": "text",
                    "analyzer": "whitespace"
                },
                "local": {
                    "properties": {
                        "app": {
                            "type": "keyword"
                        }
                    }
                }
            }
        },
        "settings": {
            "index": {
                "number_of_shards": "1",
                "provided_name": "trace_log",
                "creation_date": "1650936491709",
                "analysis": {
                    "analyzer": {
                        "default": {
                            "type": "whitespace"
                        }
                    }
                },
                "number_of_replicas": "1",
                "uuid": "LMQwk7k0R66Qi_Udw001Eg",
                "version": {
                    "created": "7060299"
                }
            }
        }
    }
```