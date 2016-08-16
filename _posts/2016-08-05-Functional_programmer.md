---
layout: post
title: 函数式编程
categories: [blog]
tags: [Java]
description: 
---





获得排序并获得最大值

```
Collections.sort(bpmBdEvaluates, new Comparator<BpmBdEvaluate>() {
    @Override
    public int compare(BpmBdEvaluate o1, BpmBdEvaluate o2) {
        return o1.getUpdated_at().compareTo(o2.getUpdated_at());
    }
});
return bpmBdEvaluates.get(bpmBdEvaluates.size() - 1);
```