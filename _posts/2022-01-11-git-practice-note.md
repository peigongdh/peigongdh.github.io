---
layout: post
title:  "git实践 笔记"
date:   2022-01-11 00:00:00 +0800
categories: cs
tag: [git]
---

## rebase

之前rebase用的不多，现在新的代码规范要求在自己的分支上rebase主干后再提交commit  
重新梳理了一遍rebase的含义，其关键可以从字面理解，re-base，base就是我们checkout的那个commit  

```
mine   :             -> D -> E
master : A -> B -> C -> F
```

当我们完成自己的开发想把自己的分支合并上去的时候，此时主干上已经有了新提交  
如果我们不希望使用merge使主干变得杂乱，就用rebase将原来commit的基点从C变为F，即变基

```
mine   :                  -> D -> E
master : A -> B -> C -> F
```

虽然这样历史提交的真实过程被篡改了，但是如果只是在自己的分支上使用，可以使提交记录维护地非常干净  

## squash

像上面的rebase过程中，经常会遇到反复fix同一处conflict的问题，为了解决类似的问题，可以使用squash来压缩commit  

```
mine   :             -> D1 -> D2
master : A -> B -> C
```

以这个场景为例，我们使用rebase merge的方式，将mine的内容合并到master中  

```
git checkout mine
git rebase -i master

# git rebase -i master 类似于下面的操作
# git rebase -i HEAD~N
# git rebase master
```

命令行提示如下  

```
pick 442ea7d 10th commit in feature/reabase-practice
pick 266af78 11th commit in feature/reabase-practice

# Rebase e8252be..266af78 onto e8252be (2 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
```

我们编辑pick的选项为需要的效果，比如squash或者fixup（丢失log）  
squash后的commit会在顶部的pick中保留全部log  

```
mine   :             -> D'
master : A -> B -> C
```

即D1和D2合并为D'  

此时再提交merge request，进行后续code review流程  