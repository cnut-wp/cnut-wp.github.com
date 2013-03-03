---
layout: post
title: "all pull few push for git repo"
date: 2013-03-03 20:40
comments: true
categories: 
---
这学期我担了一门课的助教。课上需要为学生建立一个 git 仓库。该仓库需要所有学生都有 pull 权限，但是又要赋予几个 TA push 权限。这个要求花了自己一段时间配置。

由于是低年级的学生，并且人数也比较多，所以复杂的授权认证不合适，最好学生什么都不需要做就可以 pull 代码。

git 支持许多协议，例如 http/https，git， ssh。http 可以很方便的让所有学生都能 pull 代码，但是却不能允许 TA push 代码 （或者我不知道如何配置），而 ssh 认证有很好的授权认证，但是它却比较烦。于是我选择了 git 协议。（以上分析来自 Pro Git）

首先，我用 gitolite 去管理所有 git 仓库。gitolite 操作十分地方便。具体参见 gitolite 官网。
需要注意的是，在使用 gitolite 配置仓库权限的时候可以这样写：
	repo    yourRepository
		RW+     =   TA
		R       =   daemon
这样 gitolite 会自动在仓库 yourRepository 中创建一个名字为 daemon-export-ok 的空文件，该文件会让 git daemon 把该仓库 share 出去。
然后再启动 git daemon。
	git daemon --reuseaddr --base-path=/example-repositories-dir-path /example-repositories-dir-path
之后就可以通过下面命令正常 clone 代码了。
	git clone git://your-server-name/yourRepository

说明：

* --reuseaddr 参数使得服务无须等到旧的连接尝试过期以后再重启，--base-path 参数使得 clone 项目的时候不用给出完整的路径，而最后面的路径告诉 Git  Daemon 进程需要导出仓库的位置。
* 确保启动 git daemon 的用户有读取仓库文件的权限。
* 可以在系统启动脚本中启动 daemon。

参考文档：

*  Pro Git
*  [gitolite](http://gitolite.com/gitolite/qi.html)
*  [http://gitolite.com/gitolite/qi.html](http://gitolite.com/gitolite/qi.html)
*  [http://granjow.net/git-read-access.html](http://granjow.net/git-read-access.html)
*  [http://norbu09.org/2009/08/02/git-via-HTTP-(startup-automation-3).html](http://norbu09.org/2009/08/02/git-via-HTTP-(startup-automation-3\).html)
