---
title: mongoDB Schema Validation
date: 2018-04-28 14:51:26
tags: 数据库, Mongodb
---
![Mongodb](http://opxvbng4q.bkt.clouddn.com/mongodb.png)



## Schemaless
mongoDB作为现在很流行文档型数据库，在各大公司收到欢迎。schemaless作为mongodb的一大特点，既给MongoDB带来了简单、灵活的使用优势，同时也给数据的一致性带来了挑战。

作为以Python为主要技术栈的开发团队，因为Python本身的弱类型特点，再加上前期快速迭代造成的遗留问题。mongoDB内部出现了大量的数据问题。甚至有金额字段数字和字符串混用。

为了解决这些类型模糊和字段缺失问题，mongoDB在3.2版本引入了`Schema Validation`，并在3.6版本中得到增强。
<!--more-->

## Schema Validator
Schema Validator允许用户来自己定义document的校验格式。Schema Validator是作用在`collection`上面的，所以我们可以通过设置`collection`的属性来控制schema校验。

下面通过例子来讲解一下：
假设我们要建立一个电商网站，需要建立一张商品集合`sku`的collection，sku里面可能会有以下属性。
```bson
{
id: 900,    
name: "玻璃杯",    
description: "一个好看的玻璃杯",
details: {
  weight: "470g",
  color: "绿色",
  material: "玻璃"
},
price: 100,
price_history: [
  {
    price: 80,
    date: "2018-4-1"
  },
  {
    price: 100,
    date: "2018-4-5"
  }
]
}
```
那么现在假设上面的字段都是必须的，我们由此来建立这个这个colletion的schema。

mongoDB的schema分为两种模式分别为`strict`和`moderate`，在`strict`模式下，mongoDB会将shcema检测应用于所有的文档上面。而`moderate`模式下，当进行`UPDATE`操作时，只会应用于所有检测字段都已经满足条件的文档上面。

## 创建Schema
创建schema相当简单，相信使用过`mongoDB`能看懂下面的创建格式。
```bson
db.createCollection(
"sku",
"Validator": {"$and" : [
                    {
                        "id" : {
                            "$type" : "int",
                            "$exists" : true
                        }
                    },
                    {
                        "name" : {
                            "$type" : "string",
                            "$exists" : true
                        }
                    },
                    {
                        "description" : {
                            "$type" : "string",
                            "$exists" : true
                        }
                    },
                    {
                        "details" : {
                            "$type" : "object",
                            "$exists" : true
                        }
                    },
                    {
                        "details.weight" : {
                            "$type" : "string",
                            "$exists" : true
                        }
                    },
                    {
                        "details.color" : {
                            "$type" : "string",
                            "$exists" : true
                        }
                    },
                    {
                        "details.material" : {
                            "$type" : "string",
                            "$exists" : true
                        }
                    },
                    {
                        "price" : {
                            "$type" : "int",
                            "$exists" : true
                        }
                    },
                    {
                        "price_history" : {
                            "$type" : "array",
                            "$exists" : true
                        }
                    }
                ]
            }
)
```
这样一个`collection`就创建好了，schema默认是`strict`模式。

## 非法格式的处理

对非法格式的处理，mongoDB提供了`validationAction`这个参数来控制。默认为`error`，当出现非法格式直接报错。用户可以修改为`warn`，允许非法格式的更改，但对非法格式的修改记上日志。
