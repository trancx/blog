# Thinking in Git

## Preface

keke 😊，套用一下老规范作为标题，本章的是 Git 的三棵树延续

{% hint style="info" %}
`git` 中 `commit id` 和 三棵树 是非常重要的概念
{% endhint %}

## git checkout，reset

以前一直不懂为什么叫  `checkout`，因为我理解的 `checkout` 应该是作为一种保障或者说一种状态，比如订酒店最后结账的时候，我们就可以标记为  `checkout` 状态，或者票已经过闸等，后来偶尔想到，其实 `checkout` 就是把本地仓库的某个子仓库（也可以理解为 Branch）加载到 `Index Tree`，`Working Tree`，这个动作就可以理解为 `checkout`

这俩个 `Trees` 是公用的，只有一份，所以在你做了修改而没有提交到仓库（`commit` \) 的时候，这时候有俩种情况，一种是修改只在 Working Tree中，那么就会有警告，但是一般把修改带到下一个 `Working Tree`中，

看看下面引用所说的 `checkout` 到底会做什么

> Switching branches or cloning goes through a similar process. When you checkout a branch, it changes **HEAD** to point to the new branch ref, populates your **index** with the snapshot of that commit, then copies the contents of the **index** into your **working Directory**.

### checkout branch & checkout path

对于 `checkout` 一个 `branch`，只是把当前的 `HEAD` 调整到对应的 `branch` 的 `HEAD`，这时候同时会拷贝内容到 `Index Tree` 和 `Working Tree`，但是它是 `Working Tree safe` 的，因为在之前的分支如果有不同的内容，那么会做一个 `merge` （针对 `Working Tree`）

> First, unlike `reset --hard`, `checkout` is working-directory safe; it will check to make sure it’s not blowing away files that have changes to them. Actually, it’s a bit smarter than that — it tries to do a trivial merge in the working directory, so all of the files you _haven’t_ changed will be updated. `reset --hard`, on the other hand, will simply replace everything across the board without checking.

而 `checkout` 一个 `path` 则是一个危险的动作，它从指定的 一个 `commit` 中拿出指定的一个文件，并同时覆盖 `Index Tree` 和 `Working Tree`，跟 `reset --hard` 是异曲同工，而且，`reset --hard` 仍然可以把文件找回，但是被 `checkout` 的在工作目录的文件，如果之前没有提交过，那么就无法找回了

{% hint style="info" %}
如果 commit 正好是 HEAD，那么只会从 Index Tree 中覆盖 Working Tree
{% endhint %}

参考 `reset --hard` 的下面的解释，非常类似，

![](.gitbook/assets/image%20%28112%29.png)

这里不占出其它的 reset 操作了，链接中讲的非常明白，图也画的非常的好

#### checkout conflict

刚才说了，`checkout` 不带 path 的时候，只是不同的分区之间操作，而且是对于工作目录来说是非常安全的，因为会自动把缓存区和工作目录的修改带到下一个分支的工作目录中去，我理解其中做了两次`merge`

```c
merge  (merge index, wdir) dst_branch  => wdir
```

这个操作如果出现了冲突，那么就无法切换分支，当然这也是保证当前工作目录能顺利得到保护

### branch binding

![](.gitbook/assets/image%20%2822%29.png)

每个分支都可以绑定一个远程分支，当你敲 `git status` 或者是切换到 该分支的时候，就会将这个分支的记录与绑定的远程分支做对比，当然，这个远程分支需要你用 `fetch` 才能更新

![](.gitbook/assets/image%20%2865%29.png)

![](.gitbook/assets/image%20%28185%29.png)

```text
git branch (--set-upstream-to=<upstream> | -u <upstream>) [<branchname>]
git branch --unset-upstream [<branchname>]
```

实际上 建立这种联系的好处就在于

> Having an upstream branch registered for a local branch will:
>
> * tell git to **show the relationship between the two branches in `git status` and `git branch -v`**.
> * directs **`git pull`** _**without arguments**_ **to pull from the upstream when the new branch is checked out**.

当我们查看 .config  文件的时候就会发现有这么几条

```text
[remote "origin"]
        url = https://github.com/xx/xx.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
[branch "test"]
        remote = origin
        merge = refs/heads/ns/test
```

之所以可以简写也是因为通过查看这个文件，帮我们拓展了，所以可以偷一会懒，但是偷懒的后果就是有时候一些奇怪的 BUG 会让人匪夷所思

## refspec

{% hint style="info" %}
请读者先阅读一下引用中关于 `refspec` 的内容
{% endhint %}

前面也提到了 .config 文件，其实我们很多简写的操作都是通过它来实现的

当我们用  git clone 的时候，会拷贝远程分支，同时在本地建立一个 master 分支和远程的master分支绑定，所以当我们想要推到远程的时候，直接 git push 就可以完成操作，但是如果我们把这个绑定给取消了

```text
$ git branch  --unset-upstream
$ 
$ git status
On branch master
nothing to commit, working tree clean
```

接着我们增加新的修改，然后这时候我们敲

```text
$ git push
fatal: The current branch master has no upstream branch.
To push the current branch and set the remote as upstream, use

    git push --set-upstream origin master

```

就会发现无法实现，这在帮助文档有关 `push.default` 讲的很详细，并且如果建立了绑定，但是分支名不同，也会冲突

```text
$ git push (on branch test  --- remote branch /ns/test )
fatal: The upstream branch of your current branch does not match
the name of your current branch.  To push to the upstream branch
on the remote, use

    git push origin HEAD:ns/test

To push to the branch of the same name on the remote, use

    git push origin HEAD

To choose either option permanently, see push.default in 'git help config'.
```

实际上，这些都是通过 `refs` 目录下的文件来实现的，之前看到的奇奇怪怪的 `res/xx/xx` 都是指这个目录（但是可能指的是服务器端）

```bash
$ cd .git/refs
$ ls -Rf
.:
heads/  remotes/  tags/

./heads:
help  master  test

./remotes:
origin/

./remotes/origin:
HEAD  master  ns/

./remotes/origin/ns:
test

./tags:
```

下面来看一个 `git push` 指令

> git-push - Update remote refs along with associated objects

实际上，我们 `push` 应该带上两个 `refs`

`push remote_name <src>:<dst>`

`remote_name`  需要的原因，是因为通过 `.config` 才能得到远程的服务器地址，不然无法确定，确定了服务器的地址，当然还要确认就是到底指向的是哪一个 `refspec` ，同时我们在确定本地需要上传哪个 `refspec`，然后在上传特定的文件，也就是说，实现 `push` 操作，关键就是确定两个 `refspec`，一个是本地，一个是在服务器端

当然，我们可以直接用 HEAD 默认指定当前分支对应的 `refs`

`push remote_name HEAD:<dst>`

甚至，我们还能直接在 `.config` 中加上 `push` 的默认操作，借用官方的一个例子

```text
[remote "origin"]
	url = https://github.com/schacon/simplegit-progit
	fetch = +refs/heads/*:refs/remotes/origin/*
	push = refs/heads/master:refs/heads/qa/master
```

这里说的就是当你在 `master` 分支的 `push` 操作默认确定了两个`refspec`，所以就只需要敲 `push` 即 OK，当你明白`push`操作关键的三点，就是找到服务器地址以及两个`refspec`文件，那么对于这些操作就会有一个新的理解

对于fetch，则正好相反，可以看到，`src` 代表的是服务器的目录，`dst` 则是本地，因为对于本地的远程分支记录，我们都默认放在 `remotes` 下面，所以看起来会这样，我们从宏观上理解它的方向就是从 `src -> dst`，这一点是没有改变的

再借用一个例子，

```text
$ git fetch origin master:refs/remotes/origin/mymaster
```

更精确的写法是

```text
$ git fetch origin refs/heads/master:refs/remotes/origin/mymaster
```

{% hint style="info" %}
还是那三要素，远程地址，远程`refs`，本地`refs`
{% endhint %}

![](.gitbook/assets/image%20%28189%29.png)

当你明白上面所说的，看这句话就会不同的理解，在 `git` 分支的概念完全是虚构出来的，实际的操作就是这些文件，当然了，这里还有 `objects` 的实际文件内容，但是这层我们可以屏蔽，所以无论什么默认操作，其实最终都是确定了这些玩意儿~~~

而一些默认的配置，或者是通过配置文件实现，或者是通过默认的操作，比如当前的分支名对应本地同名目录，并且对应远程的相应分支名。

{% hint style="info" %}
详细的配置可以看官方的手册，非常的详尽
{% endhint %}

## git log, diff

`git-log - Show commit logs`   

```text
$ git log --pretty=format:'%h %s' --graph
* 734713b Fix refs handling, add gc auto, update tests
*   d921970 Merge commit 'phedders/rdocs'
|\
| * 35cfb2b Some rdoc changes
* | 1c002dd Add some blame and merge stuff
|/
* 1c36188 Ignore *.gem
* 9b29157 Add open3_detach to gemspec file list
```

```text
$ git show HEAD^
commit d921970aadf03b3cf0e71becdaab3147ba71cdef
Merge: 1c002dd... 35cfb2b...
Author: Scott Chacon <schacon@gmail.com>
Date:   Thu Dec 11 15:08:43 2008 -0800

    Merge commit 'phedders/rdocs'
```

`git-diff - Show changes between commits, commit and working tree, etc`

**Various ways to check your working tree**

```text
$ git diff            (1)
$ git diff --cached   (2)
$ git diff HEAD       (3)
```

1. Changes in the working tree not yet staged for the next commit.
2. Changes between the index and your last commit; what you would be committing if you run "git commit" without "-a" option.
3. Changes in the working tree since your last commit; what you would be committing if you run "git commit -a"

**Comparing with arbitrary commits**

```text
$ git diff test            (1)
$ git diff HEAD -- ./test  (2)
$ git diff HEAD^ HEAD      (3)
```

1. Instead of using the tip of the current branch, compare with the tip of "test" branch.
2. Instead of comparing with the tip of "test" branch, compare with the tip of the current branch, but limit the comparison to the file "test".
3. Compare the version before the last commit and the last commit.

**Comparing branches**

```text
$ git diff topic master    (1)
$ git diff topic..master   (2)
$ git diff topic...master  (3)
```

1. Changes between the tips of the topic and the master branches.
2. Same as above.
3. Changes that occurred on the master branch since when the topic branch was started off it.

{% hint style="info" %}
记住一点，Git 是基于 commit ID ，然后这些ID之间的变化也记录了下来。
{% endhint %}

## git rm

`git-rm - Remove files from the working tree and from the index`

```text
--cached
Use this option to unstage and remove paths only from the index. 
Working tree files, whether modified or not, will be left alone.
```

if all you really want to do is to remove from the index the files that are no longer present in the working tree \(perhaps because your working tree is dirty so that you cannot use `git commit -a`\), use the following command:

```text
git diff --name-only --diff-filter=D -z | xargs -0 git rm --cached
```

## merge conflict

当出现冲突，git 会把冲突写在源文件上，不会产生新的commit，我们需要做的就是利用 `git status` 检查冲突文件，然后确认无误后 `git add` 接着提交。

## important

It’s important to understand that `git checkout -- <file>` is a dangerous command. Any local changes you made to that file are gone — Git just replaced that file with the most recently-committed version. Don’t ever use this command unless you absolutely know that you don’t want those unsaved local changes

 Remember, anything that is _committed_ in Git can almost always be recovered. Even commits that were on branches that were deleted or commits that were overwritten with an `--amend` commit can be recovered \(see [Data Recovery](https://git-scm.com/book/en/v2/ch00/_data_recovery) for data recovery\). However, anything you lose that was never committed is likely never to be seen again.

## Reference

{% embed url="https://git-scm.com/doc" %}

{% embed url="https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified" %}

{% embed url="https://git-scm.com/book/en/v2/Git-Internals-The-Refspec" %}



