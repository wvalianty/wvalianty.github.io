---
layout: page
title: About
description: 淡泊明志
keywords: Yong Wang, 王勇
comments: true
menu: 关于
permalink: /about/
---

我是王勇，静心做事，淡泊明志。

坚信熟能生巧，努力改变人生。

## 联系

* 邮箱：18513169404@163.com

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
