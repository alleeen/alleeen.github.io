---
layout: post
title:  "Github的正确使用方法"
date:   2016-08-24 16:21:39
comments: true
ads: true
categories: 软件技术
tags: [Git,Github]
---

在了解了Git的基本用法后（如果你还未了解 Git 的基本使用方法，建议你先话点时间阅读下《 Pro Git 》这本书），相信你已经开始跃跃欲试了，那么我就说下如何正确的使用 Github。下面的图描述了使用 Github 的基本流程：

![Github Flow](/img/github-flow/github-flow.png)

<!--more-->

### 第一步：Fork项目
Fork 项目其实就是在 Github 上拷贝一份他人项目的副本作为自己的项目。当你进入一个项目页面后，会在右上方看见一个*Fork*的按钮，点击它就可以 Fork 一个项目。

![Fork Project](/img/github-flow/fork-project.jpg)

需要注意的是Fork项目后，你自己的项目并不会和源项目保持自动同步，所以你需要手动进行更新，如何更新请看：*第五步：拉取源项目的更新*。

### 第二步：Clone 到本地
Fork 项目后，我们就可以把代码 Clone 到本地以便我们修改。Github 提供两种 Clone 项目的方式，SSH/HTTPS。如果选用SSH模式，你需要先在本地生成一对SSH Key并上传到Github用于身份识别，具体请参考 Github 的帮助文档：[Generating an SSH key](https://help.github.com/articles/generating-an-ssh-key/)。如果选用HTTPS模式，在更新和提交时就要输入 Github 的用户名和密码。一般来说使用 SSH 模式，在一次配置后，就可以免输密码提交代码，比较方便，但使用 HTTPS 模式更具备通用性，所以各有利弊，随意选择~

```
# 使用 ssh clone 项目到本地
$ git clone git@github.com:rvm/rvm.git

# 使用 https clone 项目到本地
$ git clone https://github.com/rvm/rvm.git
```

### 第三步：创建分支

每次开发新功能，都应该新建一个单独的分支（这方面可以参考[《Git分支管理策略》](http://www.ruanyifeng.com/blog/2012/07/git.html)）。

```
# 获取主干最新代码
$ git checkout master
$ git pull

# 新建一个开发分支myfeature
$ git checkout -b myfeature

```

### 第四步：Commit 新代码
分支修改后，就可以提交commit了。

```
$ git add --all
$ git status
$ git commit --verbose
```
- git add 命令的all参数，表示保存所有变化（包括新建、修改和删除）。从Git 2.0开始，all是 git add 的默认参数，所以也可以用 git add . 代替。
- git status 命令，用来查看发生变动的文件。
- git commit 命令的verbose参数，会列出 diff 的结果。

需要注意的是 Commit 代码必须给出简明扼要的提交信息，下面是一个范本，第一行是不超过50个字的提要，然后空一行，罗列出改动原因、主要变动、以及需要注意的问题。最后，提供对应的网址（比如Bug ticket）。

```
Present-tense summary under 50 characters

* More information about commit (under 72 characters).
* More information about commit (under 72 characters).

http://project.management-system.com/ticket/123
```

### 第五步：拉取源项目的更新
当我们在修改代码的时候，源项目肯定也会发生变化，所以在我们向源项目推送代码之前，需要先将源项目的代码更新拉取下来。

先查看我们的 Remote 配置

```
$ git remote -v
origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
```

将源项目添加为 upstream

```
$ git remote add upstream https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git
```

检查配置是否生效

```
$ git remote -v
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (fetch)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (push)
```

拉取源项目的变更

```
git fetch upstream
remote: Counting objects: 75, done.
remote: Compressing objects: 100% (53/53), done.
remote: Total 62 (delta 27), reused 44 (delta 9)
Unpacking objects: 100% (62/62), done.
From https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY
 * [new branch]      master     -> upstream/master
```

切换到 master 分支

```
$ git checkout master
```

将源项目的修改合并到本地 master 分支

```
git merge upstream/master
```

### 第六步：Rebase 本地分支并解决冲突
接着我们切换到之前的开发分支 myfeature，并同 master 分支进行同步

```
$ git checkout myfeature
$ git rebase master
```

有时我们会和主干发生冲突，那么我们需要在本地把所有冲突解决掉后才能继续合入代码。如何解决冲突，请阅：[Resolving a merge conflict from the command line](https://help.github.com/articles/resolving-a-merge-conflict-from-the-command-line/)

### 第七步：Push到Github
同步好本地分支后，我们就可以将代码推送到Github了

```
git push -u origin myfeature
```

### 第八步：发送Pull Request
点击项目页面上方的pull request按钮

![pull request button](/img/github-flow/create-pull-request-1.jpg)

我们自己的项目选择之前的开发分支，源项目选择 master 分支

![pull request](/img/github-flow/create-pull-request-2.png)

在下面的页面上填写上描述，然后点击发送即可，接着下来就是原作者的事儿了，如果他同意合入我们会在项目的 master 分支看到我们刚刚贡献的代码。
