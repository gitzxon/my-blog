---
title: hexo + next 显示乱码
date: 2023-01-23 00:53:46
tags: 
---

###
关键字：
hexo，next，启动，访问，乱码，hexo-renderer-swig

# hexo + next 显示乱码

在使用 hexo + next 搭建静态网站的过程中，发现启动成功，但是访问 http://localhost:4000 的时候显示乱码
```js
{% extends '_layout.swig' %} {% import '_macro/post.swig' as post_template %} {% import '_macro/sidebar.swig' as sidebar_template %} {% block title %}{{ config.title }}{% if theme.index_with_subtitle and config.subtitle %} - {{config.subtitle }}{% endif %}{% endblock %} {% block page_class %} {% if is_home() %}page-home{% endif -%} {% endblock %} {% block content %}
{% for post in page.posts %} {{ post_template.render(post, true) }} {% endfor %}
{% include '_partials/pagination.swig' %} {% endblock %} {% block sidebar %} {{ sidebar_template.render(false) }} {% endblock %}
```
原因是 hexo 在 5.0 之后把 swig 给删除了，需要自己手动安装，执行下面命令：
```js
npm i hexo-renderer-swig
```
重启后正常展示网页了。


