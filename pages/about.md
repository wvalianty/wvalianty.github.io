---
layout: about
title: About
description: 一片冰心在玉壶
keywords: Wang Yong, 王勇
comments: true
menu: 关于
permalink: /about/
---

弱水三千，只取一瓢饮，道法自然。

急停急起，腾云入霄，当空斩虹，见血收刀，那一夜，我也曾梦见百万雄兵。

## 联系
* 微信：1015445504
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
