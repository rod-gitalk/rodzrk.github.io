---
title: elasticsearch scripting module
date: 2020-06-17 22:43:56
updated: 2020-08-02 22:00:00
tags: elasticsearch
categories: elasticsearch
cover: /static/images/elasticsearch.jpg
---

# ES脚本模块API兼容性问题

<p style="text-indent: 2em">在ES的API中是支持脚本的,但是在早期的版本,部分的API版本变化相对比较频繁,因此在不同版本之间是可能存在兼容性问题的,这篇文章主要用于记录部分API的变化.</p>

<!-- more -->
注: 由于项目中仅用到了5.4.x和5.6.x版本的ES,因此该文章只记录了部分版本的变化.

## 一. Store Scripts

可以使用_scripts端点将脚本存储在集群或者从集群中检索脚本.如果ES启用了安全功能,则必须拥有以下权限才能创建、检索或者删除存储的脚本。

{% note info %}
查看更多安全功能相关信息,请参阅 {% link Security privileges https://www.elastic.co/guide/en/elasticsearch/reference/master/security
-privileges.html %}
{% endnote %}
### 1. 5.3.x-5.5.x

#### 1.1 存储脚本请求

``` http request
/_scripts/{id}
```

- id在存储的脚本中是唯一的。

<p style="text-indent: 2em">

如下是一个存储Painless脚本的示例，脚本名为 `calculate-store` 

</p>

``` http request
POST _scripts/calculate-score
{
  "script": {
    "lang": "painless",
    "code": "Math.log(_score * 2) + params.my_modifier"
  }
}
```

#### 1.2 检索已创建的脚本

``` http request
GET _scripts/calculate-score
```

#### 1.3 使用存储的脚本

``` http request
GET _search
{
  "query": {
    "script": {
      "script": {
        "stored": "calculate-score",
        "params": {
          "my_modifier": 2
        }
      }
    }
  }
}
```

#### 1.4 删除存储的脚本
``` http request
DELETE _scripts/calculate-score
```

### 2. 5.6.x,6.0.x-6.8.x,7.0.x-7.3.x

#### 2.1 存储脚本请求

``` http request
/_scripts/{id}
```

- id在存储的脚本中是唯一的。

<p style="text-indent: 2em">

如下是一个存储Painless脚本的示例，脚本名为 `calculate-store` 

</p>

``` http request
POST _scripts/calculate-score
{
  "script": {
    "lang": "painless",
    "source": "Math.log(_score * 2) + params.my_modifier"
  }
}
```
**注: 此处与之前的版本不同,发生了变化,`code`修改为了`source`**

#### 2.2 检索已创建的脚本

``` http request
GET _scripts/calculate-score
```

#### 2.3 使用存储的脚本

``` http request
GET _search
{
  "query": {
    "script": {
      "script": {
        "id": "calculate-score",
        "params": {
          "my_modifier": 2
        }
      }
    }
  }
}
```
**注: 使用存储的脚本时API发生了变化,`stored`修改为了`id`**
#### 2.4 删除存储的脚本
``` http request
DELETE _scripts/calculate-score
```


## 参考文档
{% note info %}
{% link How to use scripts(master) https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-scripting-using.html %}
{% endnote %}