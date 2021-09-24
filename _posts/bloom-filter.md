---
layout: post
title:  "BloomFilter浅析"
date:   2021-06-08 00:00:00 +0800
categories: cs
tag: BloomFilter
---

本质上布隆过滤器是一种数据结构，比较巧妙的概率型数据结构（probabilistic data structure），特点是高效地插入和查询，可以用来告诉你 “某样东西一定不存在或者可能存在”。

> https://zhuanlan.zhihu.com/p/43263751