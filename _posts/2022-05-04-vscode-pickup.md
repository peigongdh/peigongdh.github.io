---
layout: post
title:  "使用vscode的正则表达式提取文本"
date:   2022-05-04 00:00:00 +0800
categories: tool
tag: [vscode, regular-expression]
---

## 背景

工作中遇到需要从日志中提取用户uid，并做去重，如何使用vscode快速实现

## 解决方案

提取日志，按换行符分隔，每行一条日志，例如：

```
Info 2022-04-01 00:06:00.748+08:00 uid = 'asdasdafsss' and condition_a = '1' Cost:2.03ms
Info 2022-04-02 00:05:00.748+08:00 uid = 'asdasdasdfa' and condition_a = '1' Cost:2.03ms
Info 2022-04-03 00:04:00.748+08:00 uid = 'safafsfasxx' and condition_a = '1' Cost:2.03ms
Info 2022-04-04 00:03:00.748+08:00 uid = 'uufghfhgfhf' and condition_a = '1' Cost:2.03ms
Info 2022-04-05 00:02:00.748+08:00 uid = 'fesxaxsaffa' and condition_a = '1' Cost:2.03ms
Info 2022-04-06 00:01:00.748+08:00 uid = 'sagevyutyio' and condition_a = '1' Cost:2.03ms
```

找出日志的通用格式，使用正则表达式筛选出包含uid的关键内容：

```
uid = '[a-z]*'
```

使用快捷键选中内容：shift+command+L  
或者使用菜单选中内容：selection -> select all occurrences  
复制到新的文件中，再做两次全局替换将【uid = '】和【'】替换为空白即可  
去重使用插件Transformer，使用shift+command+p打开命令行，使用unique lines去重  