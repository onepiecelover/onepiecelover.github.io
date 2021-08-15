---
title: git简介
date: 2021-08-15 17:24:17
tags: git
---

### git介绍
> Git是目前世界上最先进的分布式版本控制系统。能记录每次文件的改动。

### git的诞生
> 开发Samba的Andrew试图破解BitKeeper的协议（这么干的其实也不只他一个），被BitMover公司发现了（监控工作做得不错！），于是BitMover公司怒了，要收回Linux社区的免费使用权。Linus花了两周时间自己用C写了一个分布式版本控制系统，这就是Git！一个月之内，Linux系统的源码已经由Git管理了。


### 安装git
> 可以参考`https://github.com/git/git`。但是一般不。
> - 大部分linux系统都已经自带了git，如果没有的话执行 centos下 `yum install git` or ubuntu下 `apt-get install git`
> - 在Windows上，二进制安装


### 设置git

> 在我们能用git工作之前，我们需要做个一次性的配置。为了git能跟踪到谁做了修改，我们需要设置你的用户名。

```
git config --global user.name "your_username"
git config --global user.email "your_email@domain.com"
```

### 创建一个本地代码库

> - 作为例子，我们会假装我们有一个网站（无所谓技术）存在于我们机器上的`workspace`文件夹下的`my_site`文件夹内。在命令行中，去到你的站点的根文件夹。在OS X和Linux上:

```
# *nix系统
cd ~/workspace/my_site/

#在Windows上：
cd c:\workspace\my_site
```
> - 我们首先需要告诉git这个文件夹是我们需要跟踪的项目。所以我们发送这个命令来初始化一个新的本地Git代码库

```
git init
```
> - Git会在`my_site`文件夹内创建一个名为`.git`的隐藏文件夹，那就是你的本地代码库。


### 加载（Stage）文件
> - 我们现在需要命令git我们需要加载（stage）所有项目文件。发送：

```
git add .
```
> - 最后的“.”符号的意思是“所有文件、文件夹和子文件夹”。假如我们只想要把特定文件添加到源代码控制中去，我们可以指定它们：

```
git add my_file, my_other_file
```


### 提交文件
> 现在，我们想要提交已加载（staged）的文件。阅读“添加一个时间点，在这里你的文件处在一个可还原的状态”。我们提交我们的文件时，总是附带着有意义的注释，描述了它们现在的状态。我一直用“initial commit”来作为第一个提交的注释。

```
git commit -m "initial commit"
```
> 就这样。现在你随时都可以回滚到这个提交状态。如果你有需要检查你现在的已加载（staged）和未加载（unstaged）文件的状态、提交等，你可以询问git的状态：

```
git status
```

### 推送文件
> 在第一次你想推送一个本地代码库到远程代码库时，你需要把它添加到你的项目配置里。像这样做：

```
git remote add origin git@github.com:xxx/yyy.git
```
> 注意这里的“origin”只是一个习惯。它是你的远程代码库的别名，但是你可以用其他任何你喜欢的词。你甚至可以有多个远程代码库，你只需要给它们起不同的别名。
之后，你想要推送你的本地代码库的主干分支到你的远程代码库：

```
git push origin master
```