## 3.分析词汇
* text是文本
* analyzer是分析器，如果有IK的话还有ik_smart(粗粒度处理中文)、ik_max_word(细粒度中文)
---
POST http://{{ip}}/_analyze
```json
{
    "analyzer": "standard",
    "text": "my name is seal"
}
```
返回如下
```json
{
    "tokens": [
        {
            "token": "my",
            "start_offset": 0,
            "end_offset": 2,
            "type": "<ALPHANUM>",
            "position": 0
        },
        {
            "token": "name",
            "start_offset": 3,
            "end_offset": 7,
            "type": "<ALPHANUM>",
            "position": 1
        },
        {
            "token": "is",
            "start_offset": 8,
            "end_offset": 10,
            "type": "<ALPHANUM>",
            "position": 2
        },
        {
            "token": "seal",
            "start_offset": 11,
            "end_offset": 15,
            "type": "<ALPHANUM>",
            "position": 3
        }
    ]
}
```

## 4.创建索引

索引即mysql中的db,同时指定该索引用何种分词器，以RESTFUL风格修改请求类型会对应增删改查
---
PUT http://{{ip}}/school_idx
```json
{
    "settings":{
        "index":{
            "analysis.analyzer.default.type": "standard"
        }
    }
}
```
---
GET http://{{ip}}/school_idx  
返回信息如下，number_of_shards分片数量
```json
{
    "school_idx": {
        "aliases": {},
        "mappings": {},
        "settings": {
            "index": {
                "number_of_shards": "1",
                "provided_name": "school_idx",
                "creation_date": "1647771548613",
                "analysis": {
                    "analyzer": {
                        "default": {
                            "type": "standard"
                        }
                    }
                },
                "number_of_replicas": "1",
                "uuid": "lzbXGPPRRnW0JMFZWJPJjA",
                "version": {
                    "created": "7040299"
                }
            }
        }
    }
}
```

## 5.添加数据
http://{{ip}}/school_idx/_doc/1
```json
{
"name": "张三",
"sex": 1,
"age": 25,
"address": "广州天河公园",
"remark": "java developer"
}
```
同样也支持增删改查、排序、分页、范围查询
## 6.倒排索引查询
http://{{ip}}/school_idx/_doc/_search
```json
{
	"query":{
		"match_all":{}
	}
}
```
以如下方式可以查到
```json
{
	"query":{
		"match":{
			"remark" :"java"
		}
	}
}
```
以如下方式无法查到
```json
{
	"query":{
		"match":{
			"remark" :"jav"
		}
	}
}
```
match模糊匹配，但是要求存储的数据中分词有该结果。term精确匹配。match_phase对输入做分词。query_string和match类似但不用指定字段名
## 7.文档映射
|json数据|自动推测|
|--|--|
|true或false|boolean|
|小数|float|
|数字|long|
|日期|date|
|字符串|text|
|数组|由第一个值决定|
|json对象|Object|