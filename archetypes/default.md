---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}

lastmod: {{ .Date }} # 最后修改时间
draft: false                       # 是否是草稿？
tags: []  # 标签
categories: ["index"]              # 分类
author: "大菠萝"                  # 作者

# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: false       # 关闭文章目录
reward: false	 # 关闭打赏
mathjax: false    # 打开 mathjax
---

