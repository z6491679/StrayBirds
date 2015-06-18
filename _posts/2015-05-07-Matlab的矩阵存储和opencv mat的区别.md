---
layout: post
title: Matlab的矩阵存储和opencv mat的区别
category: 科研
comments: true
---

在matlab中是按列存储的，其[x y]对应于opencv的为[纵坐标 横坐标]，所以对于opencv的Rect(int x,int y,int width,int height)结构而言，在matlab中应该是pos[y,x,height,width].
一张宽720 长526的图片在matlab中为526*720.
![这里写图片描述](http://img.blog.csdn.net/20150507104358624)
总而言之：**matlab中第一个坐标代表的是长（纵轴、y），第二个坐标代表的是宽(横轴、x)**