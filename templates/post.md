<%* const t = await tp.system.prompt("文章标题"); await tp.file.rename(t); %>---
title: "<% t %>"
date: <% tp.date.now("YYYY-MM-DD") %>
tags: []
draft: false
---

# <% t %>

> 一句话摘要

## 正文

<% tp.file.cursor() %>
