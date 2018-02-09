---
layout: page
title: About
description: 打码改变世界
keywords: Bishion GuoFangbi
comments: true
menu: 关于
permalink: /about/
---

我是郭芳碧。

仰慕「编码是沟通的艺术」。

坚信「越努力越幸运」。

## 联系

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
