---
title: Hello World. 在 github 上搭建 hexo 博客
---
**写在前面**

想搭一个博客很久了。

日常工作中一些琐碎但有思考的东西还是需要记录一下，可能还会包括一些有趣的事情。

当然，写博客这种事情最终还是要靠坚持。

## 在 github 上搭建 hexo 博客

* hexo 是一个通过 nodejs 构建的博客系统。特点在于可以将博客生成静态页面，并且有许多有才的开发者为其贡献漂亮整洁的主题。另外，作为一款有逼格的博客系统，它当然也是必须支持 CLI 和 MarkDown 的。
* github 有个功能叫做 github pages。只要在自己的 repository 下建立 [yourname].github.io 这样的项目，并部署静态网站的内容到这个项目下，就可以通过 https://[yourname].github.io 这个域名来访问你的页面。省去了自己置备服务器和域名的成本。
* hexo 博客具体的写作流程和如何部署到 github 上，网上都有大量的教程，这里就不再赘述了。

### hexo 源文件的版本控制

hexo 官方介绍的部署方式，只是将编译后的博客发布到 github 项目下，并没有对 hexo 博客的源文件做版本控制。要实现源文件和编译后的博客均在 github 项目下，主要有如下的步骤：

1. 推送 hexo 源文件至 github 仓库

   ```bash
   $ cd <LOCAL_HEXO_REPO_PATH>
   $ git init
   $ git add .
   $ git commit -m "init hexo"
   $ git branch -m hexo // 这里会将默认的 master 分支重命名为 hexo 分支
   $ git remote add origin <REMOTE_GITHUB_REPO_PATH>
   $ git push -u origin hexo // 推送至远程仓库 hexo 分支，并跟踪。此时远程仓库的主分支会变成 hexo
   ```

2. 推送编译后的博客至 github 仓库的 master 分支

   ```bash
   $ vim _config.yml // 配置远程仓库及分支信息
   $ hexo generate
   $ hexo deploy // 编译后博客的内容已被推送到 master 分支
   ```



需要指出的是 ``hexo deploy`` 主要用到了 ``hexo-deployer-git`` 这个插件。它基本的工作原理是，从 hexo 源文件的 public 目录下拷贝出编译后的博客内容，写入 ``.deploy_git`` 目录下，并通过 ``git init`` 将该目录初始化成一个 git 仓库，最后通过 ``git push -f`` 的方式，强制推送至远程仓库。

```bash
$ git push -u <REMOTE_GITHUB_REPO_PATH> HEAD:<REMOTE_GITHUB_REPO_BRANCH> -f
```

### 关于 git remote 命令

开始想通过 ``git remote set-head`` 命令修改远程仓库的主分支名，发现行不通。远程仓库会默认将第一个建立的分支认为主分支，因此只需要在第一次向服务端推送内容时，修改分支名即可。

```bash
$ git remote set-head <name> <branch>
```

这条命令修改的是**本地仓库**中，``remotes/<REMOTE_REPO_NAME>/HEAD`` 的指向。查看方式如下。

```bash
$ git branch -a

// 结果类似
remotes/origin/HEAD -> origin/hexo
remotes/origin/hexo
remotes/origin/master
```
事实上，``git remote`` 修改的都是本地仓库中的 ``.git/refs/remotes`` 里的内容，不涉及到远程仓库的操作[^1]。

[^1]: [git remote set-head not working?](http://git.661346.n2.nabble.com/git-remote-set-head-not-working-td4187465.html)

在上面的结果中 ``remotes/origin/HEAD`` 指向的是 ``origin/hexo`` 分支。类似于 ``origin/<branch>`` 是特殊的本地分支，用来存放通过 ``git fetch`` 获取到的相应远程分支的内容[^2]

[^2]: [What is origin/master in git compared to origin master?](https://stackoverflow.com/questions/19321584/what-is-origin-master-in-git-compared-to-origin-master)

而 ``git remote set-head`` 这条命令的英文解释为：

> Sets or deletes the default branch (i.e. the target of the symbolic-ref `refs/remotes/<name>/HEAD`) for the named remote. Having a default branch for a remote is not required, but allows the name of the remote to be specified in lieu of a specific branch. For example, if the default branch for `origin` is set to `master`, then `origin` may be specified wherever you would normally specify `origin/master`.

大致意思即：当需要用 ``origin/master`` 的时候，如果 ``remotes/origin/HEAD``指向了 ``origin/master``，那就可以用 ``origin`` 来替代。

最典型的例子是  ``git merge origin/master`` 可以写成 ``git merge origin``

### 一点尾巴

一些 git 的生僻命令，确实不太容易了解清楚其真正的含义，有时甚至会因为一些解释而产生误解。好在有学生时代教会我的思辨的模式和按图索骥的能力。