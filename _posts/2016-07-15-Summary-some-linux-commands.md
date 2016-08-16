---
layout: post
title: Summary some linux commands 
categories: [blog]
tags: [Efficiency]
description: 
---

summary some linux commands 

#### awk

```
awk 'BEGIN{FS="/";OFS="\0"}{print "https://wallpapers.wallhaven.cc/wallpapers/full/wallhaven-",$5,".jpg"}' test
```

FS 输入字段分隔符（缺省为:space:）

OFS输出字段分隔符（缺省为:space:）

RS输出字段分隔符（缺省为:\n）

ORS输出字段分隔符（缺省为:\n）

#### sed

#### tcpdump

#### lsof -i

#### ps -ef

#### curl

```shell
curl -o filename url
```



