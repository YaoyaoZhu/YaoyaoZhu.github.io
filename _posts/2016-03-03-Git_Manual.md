---
layout: post
title: git manual
categories: [blog ]
tags: [Efficiency]
description: git参考手册
---

git manual



## Repository 

### 远程到本地

`git clone git@git.elenet.me:napos.vine/vine.git  `

clone的时候rename 

`git clone git@github.com:AlexanderWangsgithub/AlexanderWangsgithub.github.io.git yours.github.io`

### 本地提远程

```shell
git init
git add .
git commit -m "init"
git remote add origin  git@github.com:AlexanderWangsgithub/AlexanderWangsgithub.github.io.git
git push -u origin master
```



## Branch

get branch info：`git branch`

create branch：`git branch branchName `

switch branch：`git checkout branchName `

create and switch branch：`git checkout -b branchName `

merge to current branch：`git merge branchName `

delete branch：`git branch -d branchName `

update branch: `git fetch`



add but not commit:

```shell
git reset HEAD ~/*/MarketManagerScoreServiceImpl.java
git checkout -- ~/*/MarketManagerScoreServiceImpl.java 
```

merge conflict

```
git reset --hard
git pull --rebase
git rebase --skip
git pull
```

## Rollback
### not pushed
`git reset [--soft | --mixed | --hard`
### pushed
`git revert c011eb3c20ba6fb38cc94fe5a8dda366a3990c61`
### compare
reset 是删除历史版本，HEAD向后移动的；revert是反向提交，HEAD是向前移动。

### 同步Fork项目作者更新

```
git remote add fork_father https://github.com/fatherProject
git pull fork_father master
```

即除了origin，再添加一个来源。

