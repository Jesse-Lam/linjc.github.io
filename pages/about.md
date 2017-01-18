---
layout: page
title: About
description: 个人博客
keywords: Linjc, blog
comments: true
menu: 关于
permalink: /about/
---

作为一个码农，只想在自己的那片天地默默耕耘，  
就像田野里的稻草人，静静守护它的庄稼。

## 联系

* GitHub：[@linjc](https://github.com/linjc)
* 博客：[{{ site.title }}]({{ site.url }})
* 微博: [@林_J-C](http://weibo.com/u/2455938432)

## Skill Keywords

#### Software Engineer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_software_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### Mobile Developer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_mobile_app_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### Linux Developer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_linux_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>
