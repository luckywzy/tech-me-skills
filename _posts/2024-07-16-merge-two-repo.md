---
title: "合并两个git仓库"
date: 2024-07-16
---

我有一个用来做笔记的仓库，做课程lab的时候又有一个lab自己的仓库，现在我想把lab的仓库合并到我的笔记仓库里面，同时保留历史变更（如果不需要那么直接copy代码就行了）

找了一圈发现写的乱七八糟的，经过实践，发现需要四步。

首先你有两个仓库，repoNote, repoLab

目标：将 repoLab merge into repoNote
## 第一步
在repoNote 的根目录下，执行clone，在本地 将repoLab目录下的文件 clone 到 repoNote目录下
```commandline
 git clone /xxx/repoLab/
```

## 第二步
现在在repoNote下有一个repoLab的目录，但是你会发现repoLab的远程git仓库还是之前的，现在我们要把远程仓库切换为repoNote的远程仓库。
```commandline
git remote rm origin
git remote add origin [url]
```
将 `[url]`替换为repoNote的远程仓库地址。也可以直接使用IDE提供图形界面修改。

确保修改后两者是一样的，如下图所示：

![two-repos-same-remote-url](imgs/two-repos-remote-url.png)

## 第三步
重新拉取仓库
```commandline
git fetch 
```
此时本地仓库里面就是两个仓库的内容了。

## 推送到远程（merge）
直接在本地merge，然后再推送。
目录要在repoNote的目录, merge的过程中你会发现两者没有相同的历史，所以无法合并，需要加一个参数：
```commandline
git merge --allow-unrelated-histories repoLabBranch
```
此时两者合并成功