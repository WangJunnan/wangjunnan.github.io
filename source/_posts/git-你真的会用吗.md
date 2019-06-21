---
title: git 你真的会用吗
tags:
  - git
categories:
  - git
date: 2019-06-21 10:26:10
---
## 记录原由

最近因为公司新来的同事，在使用`Git`时犯了一些非常低级的错误，导致团队为了解决这些问题浪费了很多时间。究其原因其实还是对于`Git`内部实现不清晰，仅仅知道敲几个git命令，但是却不知道敲了这个命令`Git`会发生什么！这里根据[git官方文档](https://git-scm.com/book/en/v2)节选了一些重要概念分享出来。

## 几个重要概念

### 三种状态

1. 工作区状态
	* 就是修改了文件还没有做 `git add`的文件状态
2. 暂存区状态
	* 已经`git add`但还未`git commit`的文件状态
3. 已提交状态
	* 已经`git commit`的文件状态，也就是真正存储到`git`仓库


### Branch指针和HEAD指针

`Git`的分支特性是其最强大最独特的功能，也正是因为这个特性让`git`得以在众多的版本控制系统中脱颖而出，在理解`branch`之前，有必要先对`git commit`命令做一个简单的介绍

* 当使用`git commit`进行提交操作时，`Git`便会创建一个提交对象，这个对象会包含一个指向本次提交的指针，指针指向本次`commit`的快照。指针也就是我们通常所说的`commit id(长度为 40 的 SHA-1 值字符串)`。如此一来，`Git`就可以在需要的时候根据`commit id`来回退版本

`Branch`的本质其实仅仅是指向提交对象的**某一个可变指针**。所以根据这个概念，我们可以知道`master`并不是一个特殊的分支，他跟我们众多自定义的分支没有任何不同，唯一区别是它是`Git`的默认分支(初始化的时候总得有一个默认分支)

下面的图是单个`master`分支时的结构，这里`master`仅仅是一个指向`f30ab`提交对象的指针

![](http://img.souche.com/f2e/037ab2a225f8b9457af9a40324313f84.jpg)


执行 `$ git branch testing`创建一个 `testing`分支，会在当前所在的提交对象上创建一个名为`testing`的指针，也就是`testing`分支
![](http://img.souche.com/f2e/0efbc0ed98b12c7aadfe4a97d82d2315.jpg)

注意：此时两个指针指向了相同的提交
那么`Git`是如何来知道当前处于哪个分支的呢？
这就引出了`Git`中的另一个特殊的指针`HEAD`，`HEAD`用来指向当前所在的分支，也就是一个用来指向分支指针的指针（有点拗口），就像下图这样，当前是在`master`分支，因为`git branch`并不会移动`HEAD`指针切换分支

![](http://img.souche.com/f2e/bc84a1c0611edbcf64756ad321f0d84c.jpg)

执行 `$ git checkout testing`命令才会将`HEAD`指针指向`testing`分支
![](http://img.souche.com/f2e/d09338fd78877491ec49f7fcb191a491.jpg)

当我们在`testing`分支上继续提交几个commit之后，`testing`指针和`HEAD`指针都会跟着向前移动，但是`master`指针并不会移动，依旧指向`f30ab`这个提交
![](http://img.souche.com/f2e/8b8a8539aa7dc810a9a9f371f8751bc3.jpg)

现在执行`git checkout master`切回`master`分支，`HEAD`指针会重新指向`master`，也就是说现在又回到了一个旧的版本，这也是`Git`的神奇之处
![](http://img.souche.com/f2e/a172111db3abe0d8c6f82c7b3a99607f.jpg)

接下来我们在`master`分支上做一些修改，提交一个commit，`HEAD`和`master`指针会继续向前移动，并且这个项目的提交（快照版本）已经产生了分叉
![](http://img.souche.com/f2e/237c20e005bac4135577ffd143eeb033.jpg)

> 保存分支信息的目录在`.git/refs/heads`下，也可以在这个目录下查看或修改分支指针

## merge 命令

在上文的例子中，项目有两个分支，且已经产生了分叉，这时候需要把`testing`分支上的内容合并至`master`时需要用到`merge`命令
在`master`执行`git merge testing`，会生成一个新的提交`9c67a`，并且`HEAD`和`master`指针继续向前移动

![](http://img.souche.com/f2e/0c40f40917743bcc75b4871ad7a76bf8.jpg)


## reset 命令

假如上面的`merge`操作有问题，需要撤销，可以使用`reset`命令，但首先需要明确回退的版本是哪一个，例如要回退的版本是`c2b9e`，执行`git reset --hard c2b9e`后，`HEAD`和`master`指针会回退到版本`c2b9e`

![](http://img.souche.com/f2e/cfd70efa478ee439cc8db564681b22c2.jpg)

## ...


