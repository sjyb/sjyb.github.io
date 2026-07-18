<%*
const title = await tp.system.prompt("文章标题");
const desc  = await tp.system.prompt("文章描述（一句话概括）");
await tp.file.rename(title);
-%>
---
title: "<% title %>"
date: <% tp.date.now("YYYY-MM-DD") %>
description: "<% desc %>"
ShowToc: true
TocOpen: true
weight: 50
tags: []
draft: false
---
# <% title %>

<% tp.file.cursor() %>
