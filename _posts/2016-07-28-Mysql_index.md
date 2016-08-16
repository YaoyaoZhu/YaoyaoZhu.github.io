---
layout: post
title: 设计数据表时的索引使用
categories: [blog]
tags: [Database]
description: 
---

# 设计数据表时的索引使用

[TOC]

## 场景

- 需要维护，所以column不应频繁变化

- 重复频率极高的应该使用聚簇索引

  ​

## 概念

聚簇索引是按照数据存放的物理位置为顺序的，能提高多行检索的速度.

非聚簇索引对于单行的检索很快。

## 原理

分页目录管理，类似于OS的内存分页管理机制。

## 使用姿势

```mysql

CREATE TABLE `arena_notice_template`.`<table_name>` (
	`id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
	`template_title` varchar(40) NOT NULL COMMENT '标题',
	`template_content` text DEFAULT NULL COMMENT '内容',
	`template_id` int(11) NOT NULL COMMENT '模板ID',
	`created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
	`updated_at` timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
	PRIMARY KEY (`id`),
	INDEX `idIndex` USING BTREE (`id`) comment ''
) ENGINE=`InnoDB` AUTO_INCREMENT=350 DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci ROW_FORMAT=COMPACT COMMENT='模板表' CHECKSUM=0 DELAY_KEY_WRITE=0;
```

