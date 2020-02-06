---
title: hexo和next内置标签
date: 2020-2-6 12:26:00
tags: 
- hexo
- next
categories: blog
---

![](https://raw.githubusercontent.com/stanxia/blog-pics/master/20200206122722.png)

<!-- more -->

## note

标签使用

```
{% note class_name %} blabla {% endnote %}
class_name:
	- default
	- primary
	- success
	- info
	- warning 
	- danger
```

效果演示

{% note default %} default {% endnote %}

{% note primary %} primary {% endnote %}

{% note success %} success {% endnote %}

{% note info %} info {% endnote %}

{% note warning %} warning {% endnote %}

{% note danger %} danger {% endnote %}

## tabs

标签使用

```
{% tabs unique_name,default_index %}
<!-- tab [tag_name] -->
这是tab1
<!-- endtab -->
<!-- tab [tag_name] -->
这是tab2
<!-- endtab -->
...
{% endtabs %}
```

效果演示

```
{% tabs code,1 %}
<!-- tab java -->
java code
<!-- endtab -->
<!-- tab scala -->
scala code
<!-- endtab -->
<!-- tab python -->
python code
<!-- endtab -->
{% endtabs %}
```

{% tabs code,1 %}
<!-- tab java -->

```java
public class Hello{
  //
}
```

<!-- endtab -->
<!-- tab scala -->

```scala
object Hello{
  //
}
```

<!-- endtab -->
<!-- tab python -->

```python
def hello:
  #
```

<!-- endtab -->
{% endtabs %}

