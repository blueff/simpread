\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[segmentfault.com\](https://segmentfault.com/a/1190000006189253)

在 gerrit 审核中，经常会遇到开发人员提交的代码审核不通过的情况。

那么开发人员这时有两个选择：

1.  按照要求修改代码，然后重新提交一次代码
    
2.  修改原来的 Change
    

第一种方法会产生多次 commit，而这些 commit 实际上是没有太多意义的，所以不推荐使用。

所以推荐第二种方法。下面讲解如何做：

安装 commit-msg hook
------------------

安装 gerrit 的 commit-msg hook 的目的是为了能够在每次提交的时候在你的本地产生一个 Change-Id，这个 Change-Id 是将 gerrit 的 Change 和你的 commit 联系起来的纽带。

```
\# 到项目的根目录下执行
curl -Lo .git/hooks/commit-msg http://your-gerrit-server/gerrit/tools/hooks/commit-msg
chmod u+x .git/hooks/commit-msg
```

这样的话，你每次提交的时候，这个 hook 都会在 commit message 的后面添加一行 Change-Id：

```
$ git log -1
  commit 29a6bb1a059aef021ac39d342499191278518d1d
  Author: A. U. Thor <author@example.com>
  Date: Thu Aug 20 12:46:50 2009 -0700

      Improve foo widget by attaching a bar.

      We want a bar, because it improves the foo by providing more
      wizbangery to the dowhatimeanery.

      Bug: #42
      Change-Id: Ic8aaa0728a43936cd4c6e1ed590e01ba8f0fbf5b
```

修改已经提交至 gerrit 的 commit
-----------------------

```
$ git checkout 有问题的commit
$ <修改>
$ git commit --amend
$ git push origin HEAD:refs/for/master
Counting objects: 5, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 546 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
remote: Processing changes: updated: 1, done
remote:
remote: Updated Changes:
remote:   http://gerrithost:8080/68
remote:
To ssh://gerrithost:29418/RecipeBook.git
 \* \[new branch\]      HEAD -> refs/for/master
```

注意到上面的 Updated Changes 字样了吗？这说明我们是更新了 gerrit 上的 Change，而不是一般情况下的 New Changes。

然后你到 gerrit 上查看自己的 Change，是不是有变化了？

windows 下怎么弄？
-------------

commit-msg hook 在 windows 下可能无法起作用，但我们依然有办法解决这问题。

我们还是按照前面讲的步骤来：

```
$ git checkout 有问题的commit
$ <修改>
$ git commit --amend
```

注意，这个时候你需要手工添加 Change-Id 了，到 gerrit 上找到自己的 Change 的 Change-Id：

![](https://segmentfault.com/img/bVz8gK)

上图的最后一样就是 Change-Id，然后手工添加到 commit 注释的最后一行上：

```
Change-Id: I347f61e90f259c78fcaaa8367b804941005a9b2b
```

然后

```
$ git push origin HEAD:refs/for/master
Counting objects: 5, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 546 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
remote: Processing changes: updated: 1, done
remote:
remote: Updated Changes:
remote:   http://gerrithost:8080/68
remote:
To ssh://gerrithost:29418/RecipeBook.git
 \* \[new branch\]      HEAD -> refs/for/master
```