---
title: "Hugo博客部署"
date: 2020-05-27T17:44:02+08:00
draft: true
---

### 主要步骤

1. [get start chocolately](https://chocolatey.org/install)
2. `choco install hugo-extended -confirm`安装hugo
3. hugo new post/filedir/filename.md
4. hugo server -D -w
5. hugo  生成public 文件夹
6. 提交public文件夹即可。

**新的玩法**

Github Pages解决一个用户只有一个repo的方案是活用特定的git分支，具体来说，如果我的用户名是`username`，我的一个项目叫`project`并且我创建了`gh-pages`分支，那么可以通过`http://username.github.io/project`方式访问。Github Pages会识别这类分支并进行托管。

[新玩法](https://blog.voidmain.guru/posts/2019-07-01-blog-with-hugo/)

[新玩法解释](https://zhuanlan.zhihu.com/p/37752930)

### 注意事项

1. 一些主题模板需要一些更高级的配置，在安装hugo的时候执行`choco install hugo-extended -confirm`安装hugo的扩展，否则在通过模板生成博客的时候会报错。
2. github的github pages页面在提交之后需要过十分钟左右页面更新，或者清除cookie。

### 参考链接

* [安装hugo](https://gohugo.io/getting-started/installing)
* [配置hugo博客](