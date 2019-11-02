# 记录 Git 的使用笔记

# 本地删除文件后让git服务器也删除这个文件
在Git 2.0版本之前，本地删除文件时，想让git服务器也删除这个文件，需要使用下面的命令来添加改动：
- 直接使用 `git rm` 命令来删除文件，不仅会删除本地文件，还会自动添加改动。
- 当使用shell自身的rm命令删除文件时，可以执行下面的命令来添加改动。
    - git add -A
    - git add -u
    - 不要执行 `git add .` 命令

## git rm
使用 `git rm` 命令可以从本地删除文件，同时自动添加被删除文件到git的staged区域，后续直接执行 git commit 即可，不需要先执行 git add 命令：
```bash
$ ls
delete_by_git_rm  delete_by_rm
$ git rm delete_by_git_rm
rm 'delete_by_git_rm'
$ ls
delete_by_rm
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        deleted:    delete_by_git_rm
```
可以看到，执行 git rm delete_by_git_rm 命令后，用 ls 命令查看，没有再看到 delete_by_git_rm 文件，该文件已经从本地删除。而用 git status 命令查看，删除delete_by_git_rm文件的这个改动已经添加到git的staged区域，等待被commit。那么后续执行 `git commit` 和 `git push` 命令后，远端服务器上的同名文件也会被删除。其他人从服务器pull代码，不会再看到这个文件。

## git add -A
当使用shell自身的rm命令删除本地文件时，这个改动不会自动添加到git的staged区域。使用 `git status` 命令查看，会提示"Changes not staged for commit"：
```bash
$ rm delete_by_rm
$ git status
On branch master
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        deleted:    delete_by_rm
```
此时，需要使用 *git add* 命令来添加改动。  
一般常用 `git add .` 命令来添加本地改动到staged区域，但是针对用shell自身rm命令删除文件的情况来说，`git add .` 命令不会添加已删除文件到staged区域，执行时会打印如下警告信息：
```bash
$ git --version
git version 1.9.1
$ git add .
warning: You ran 'git add' with neither '-A (--all)' or '--ignore-removal',
whose behaviour will change in Git 2.0 with respect to paths you removed.
Paths like 'delete_by_rm' that are
removed from your working tree are ignored with this version of Git.

* 'git add --ignore-removal <pathspec>', which is the current default,
  ignores paths you removed from your working tree.

* 'git add --all <pathspec>' will let you also record the removals.

Run 'git status' to check the paths you removed from your working tree.
```
可以看到，在Git 1.9.1版本上，执行 `git add .` 后，再用 `git status` 查看，删除的本地文件还是没有添加到git的staged区域。如果我们没有注意到这一点，后续执行 `git commit` 和 `git push` 命令提交到远端服务器，那么远端服务器上的同名文件不会被删除，其他人从服务器pull代码还是会看到那个文件。

**即，在Git 1.9.1版本上，用rm命令删除本地文件后，要添加这个改动到git的staged区域，然后commit、push，远端服务器才会同步删除这个文件，`git add .`命令不会把已删除文件添加到git的staged区域。**

参考上面执行 `git add .` 命令时打印的警告信息，可以使用 `git add --all` 选项来添加已删除文件的改动，`--all` 也可以写为 `-A`，这两者是等效的。在Git 1.9.1版本上，查看 man git-add 对 -A 选项的说明如下：
> **-A --all --no-ignore-removal**  
Update the index not only where the working tree has a file matching \<pathspec\> but also where the index already has an entry. This adds, modifies, and removes index entries to match the working tree.  
If no \<pathspec\> is given, the current version of Git defaults to "."; in other words, update all files in the current directory and its subdirectories. This default will change in a future version of Git, hence the form without \<pathspec\> should not be used.

## git add -u
如果觉得要输入大写的A比较麻烦，也可以使用 `-u` 选项，该选项同样会添加已删除文件的改动：
```bash
$ git add -u
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        deleted:    delete_by_rm
```
查看Git 1.9.1版本 man git-add 对 -u 选项的说明如下：
> **-u --update**  
Update the index just where it already has an entry matching \<pathspec\>. This removes as well as modifies index entries to match the working tree, but adds no new files.  
If no \<pathspec\> is given, the current version of Git defaults to "."; in other words, update all tracked files in the current directory and its subdirectories. This default will change in a future version of Git, hence the form without \<pathspec\> should not be used.

`git add -A` 和 `git add -u` 都可以添加已删除文件的改动，它们的区别在于，-A 选项会添加新增的文件，而 -u 选项不会添加新增的文件。

**注意**：上面描述了Git 1.9.1版本上 `git add .` 命令不会添加已删除文件的改动。但是在当前最新的Git 2.23版本上，`git add .`命令可以添加已删除文件的改动。有一些Linux系统上可能还是使用老版本的git，为了兼容，对于用shell自身的rm命令删除文件的情况，建议都加上 -u 选项。

## 最后说一个突然发现自己记录的知识已经过时的小故事
我在几年前使用git的时候，记录 man git-add 里面对 -u 选项的说明如下：
> **-u, --update**  
Only match \<filepattern\> against already tracked files in the index rather than the working tree. That means that it will never stage new files, but that it will stage modified new contents of tracked files and that it will remove files from the index if the corresponding files in the working tree have been removed.  
If no \<filepattern\> is given, default to "."; in other words, update all tracked files in the current directory and its subdirectories.

这个说明跟上面Git 1.9.1版本 man git-add 里面的说明有所差异，跟当前最新的Git 2.23版本 man git-add 里面的说明更是差异巨大 (这里没有贴出Git 2.23版本的说明)。  
同时 `git add .` 在不同版本上的行为还不一样，顿时有种日新月异、地覆天翻之感。  
我不得不多次修改文章内容，添加Git版本号的说明，可以说是三易其稿。  
我已经不记得之前使用的git软件版本是多少，感觉像是过时很久的老古董。  
经过查找，Git 1.7.1版本对 git add -u 选项的说明跟我的记录一致，其链接是: <https://git-scm.com/docs/git-add/1.7.1#git-add--u>  
我之前用的应该就是Git 1.7.1版本罢。

# 使用git checkout和git reset覆盖本地修改
我们在本地修改文件、或者删除文件后，如果想恢复这些文件内容为git仓库保存的版本，可以使用下面几个命令：
- `git checkout [--] <filepath>`：可以恢复还没有执行 git add 的文件，但不能恢复已经执行过 git add 的文件
- `git reset [--] <filepath>`：把文件从git的staged区域移除，即取消git add，再使用 git checkout 进行恢复
- `git reset --hard`：恢复整个git仓库的文件内容为当前分支的最新版本

## git checkout [--] \<filepath\>
当修改文件内容、或者删除本地文件后，如果还没有执行 `git add` 命令添加改动，可以执行 `git checkout [--] <filepath>` 命令来把 filepath 指定的文件恢复成当前git分支的最新版本。对该命令的参数说明如下：
- `[--]`：表示 `--` 是可选参数，该参数用于指定后面跟着的参数只是文件路径，而不是branch分支名或者commit信息。
> 如果当前有一个branch分支名是 hello.c，且当前目录下有一个 hello.c 文件，那么不加 `--` 参数时，`git checkout hello.c` 表示切换到 hello.c 分支，而不是覆盖 hello.c 文件的改动。这种场景下，必须用 `git checkout -- hello.c` 指定覆盖 hello.c 文件的改动，`--` 参数可以消除歧义。  
> 在没有名称歧义的情况下，可以不提供 `--` 参数。例如使用 `git checkout hello.c` 覆盖 hello.c 文件。
- `<filepath>`：filepath 指定要被覆盖的文件路径，基于当前shell工作目录进行寻址。例如要覆盖当前目录下 *src* 子目录的 *utils.c* 文件，要写为 *src/utils.c*.
> 虽然filepath参数提供的文件路径是基于当前shell工作目录进行寻址，但这只是描述文件目录结构关系，并不代表在当前工作目录底下一定存在这个文件。例如删除本地的 src/utils.c 文件后，可以用 `git checkout src/utils.c` 命令恢复这个文件，但是执行这个命令时，本地并不存在 src/utils.c 文件，执行之后才会重新生成。
- 注意：这里的`[]`表示是可选参数，`<>`限定非命令选项的参数名，它们都不是参数自身的内容，在输入的时候不要加`[]`或者`<>`。
- 在Bash shell下面，`.` 表示当前目录路径。当不提供 `--` 参数、filepath 参数写为 `.` 时， `git checkout .` 命令表示覆盖当前目录及其子目录的改动。这是 git checkout 命令常用的形式。

注意：当不带有 `--` 参数时，在 bash shell 下使用 Tab 键补全，默认只会列出本地分支名、远端分支名、和 git refs，不会列出本地文件名。即，默认无法用 Tab 键来补全文件名，即使当前目录下只有一个文件也无法补全，可以先输入文件名开头的部分，才能用 Tab 键补全。  
当带有 `--` 参数时，在 bash shell 下使用 Tab 键补全，默认只会列出本地文件名，不会列出分支名和 git refs，方便补全本地文件名。

## git reset
当修改文件内容、或者删除本地文件后，如果已经执行过 git add 命令来添加该文件的改动，那么用 git checkout 命令无法恢复这个文件为当前git分支的最新版本，需要使用下面的 *git reset* 命令进行处理：
- `git reset [--] <filepath>`：把 filepath 文件从git的staged区域移除，相当于还没有执行 git add 命令。该命令不会恢复文件内容，需要再使用上面描述的 git checkout 命令来恢复文件。
> 当使用已删除的文件路径作为参数时，git checkout 命令可以不加 `--` 参数，但是 git reset 不加 `--` 参数会报错，必须使用 `--` 来表示后面跟着的是文件路径。例如 hello.c 文件被删除，git checkout hello.c 可以正常恢复该文件，而 git reset hello.c 会报错，git reset -- hello.c 不会报错。
- `git reset --hard`：将整个git仓库的文件都恢复成当前分支的最新版本，会直接恢复文件内容，不需要再执行 git checkout 命令。
> 该命令不需要传入任何文件路径，默认对git仓库跟踪的工作目录、所有子目录、以及所有文件都生效。例如，当前shell的工作目录位于某个子目录下，要恢复其父目录下的改动，不需要先执行 cd 命令切换到父目录，直接在子目录下执行 `git reset --hard` 命令就能恢复父目录下的改动。

# 使用git checkout和git reset回退到历史版本
Git是一个分布式版本控制系统，它会保存文件修改的历史版本，可以使用下面的命令回退文件到某个历史版本：
- `git checkout <commit>`：把整个git仓库文件回退到 commit 参数指定的版本
- `git checkout [<commit>] [--] <filepath>`：回退 filepath 文件为 commit 参数指定的版本
- `git reset <commit>`：把git的HEAD指针指向到 commit 对应的版本，本地文件内容不会被回退
- `git reset --hard <commit>`：把git的HEAD指针指向到 commit 对应的版本，本地文件内容也会被回退

## git checkout \<commit\>
`git checkout <commit>` 命令把整个git仓库文件回退到 commit 参数指定的版本，该参数值可以是具体的commit hash值，也可以通过HEAD index来指定。例如，`HEAD^` 对应最新版本的上一个版本，那么 `git checkout HEAD^` 命令回退git仓库下的文件内容到上一个版本，同时从当前分支脱离，处在一个未命名分支下面：
```bash
$ git checkout HEAD^
Note: checking out 'HEAD^'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b new_branch_name

HEAD is now at 8ba05ec...
$ git branch
* (detached from 8ba05ec)
  master
```
此时用 git log 命令查看log信息，最新的log已经变成前面 HEAD^ 指定的commit信息。  
由于脱离了原先的分支，修改本地文件，并执行 git commit 进行提交，不会影响原先分支。  
即，如果想在原先分支上直接回退代码，不能使用这个命令。

## git checkout [\<commit\>] [--] \<filepath\>
`git checkout [<commit>] [--] <filepath>` 命令只回退 filepath 文件到 commit 参数指定的版本，不影响其他文件，`[--]` 表示 `--` 是可选参数，用于指定后面跟着的参数只是文件路径，而不是branch分支名或者commit信息。
> 如果当前有一个branch分支名是 hello.c，且当前目录下有一个 hello.c 文件，那么不加 `--` 参数时，`git checkout hello.c` 表示切换到 hello.c 分支，而不是覆盖 hello.c 文件的改动。这种场景下，必须用 `git checkout -- hello.c` 指定覆盖 hello.c 文件的改动，`--` 参数可以消除歧义。  

当前 `git checkout [<commit>] [--] <filepath>` 命令的行为跟 `git checkout [<commit>]` 命令有所差异，具体说明如下：
- `git checkout [<commit>]` 回退整个git仓库的文件。而当前命令只回退指定文件
- `git checkout [<commit>]` 会从原先分支脱离，并影响git log显示的commit信息。而当前命令会停留在原先分支下，不影响git log显示的commit信息
- `git checkout [<commit>]` 切换之后，用git status查看，会提示"nothing to commit, working directory clean"；而当前命令切换后，用git status查看，提示"Changes to be committed"，指定回退的文件被添加到了git的staged区域，执行git commit，会多出一条新的commit信息

## git reset \<commit\>
`git reset <commit>` 命令把git的HEAD指针指向到 commit 对应的版本，本地文件内容不会被回退，会停留在原先分支下。此时，用 git log 命令查看log信息，最新的log已经变成commit对应的信息。用 git status 命令查看，一般会提示有些文件被改动，即本地文件内容和git的staged区域内容不一致，类似于本地文件在当前git仓库的版本上进行了一些修改：
```bash
$ git reset HEAD^
Unstaged changes after reset:
M       hello.c
$ git status
On branch new_branch_name
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   hello.c

no changes added to commit (use "git add" and/or "git commit -a")
```
在执行这个 git reset 命令之后，如果想让本地文件也回退到指定的版本，可以再使用 `git checkout [--] <filepath>` 命令来覆盖本地文件内容。  
执行 git reset 命令后不会从当前分支脱离，如果想在当前分支上，将本地代码回退，就可以使用这种方法。

## git reset --hard \<commit\>
`git reset --hard <commit>` 命令把git的HEAD指针指向到 commit 对应的版本，本地文件内容也会被回退，不需要再执行 git checkout 命令来回退本地文件内容。

由于 git reset 会回退 git log 显示的commit信息，使用 `git log` 命令看不到原先最新的commit hash值，无法使用该commit hash值来恢复成原先最新的版本，可以使用 `git log -g` 命令来看到所有操作过程的commit信息，就能看到原先最新的commit hash值，然后再次恢复。

# git log 命令用法实例
使用 `git log` 命令查看提交记录时，默认打印commit hash值、作者、提交日期、和提交信息。如果想要查看更多内容，可以提供不同的参数来指定查看的信息。具体实例说明如下。

## 查看提交记录具体的改动
执行 `git log` 命令会打印提交信息，默认不会列出具体的改动内容。可以使用 `git log -p` 命令来显示具体的改动内容。查看 man git-add 对 -p 选项说明如下：
> **-p, -u, --patch**  
Generate patch (see section on generating patches).

即，`git log -p` 命令默认以patch的形式来显示改动内容，会显示修改前、修改后的对比。

在 `git log -p` 后面还可以提供 commit hash 值来从指定的 commit 开始查看，但不是只查看这个 commit 的改动。例如，git log -p 595bd27 命令是从 595bd27 这个 commit 开始显示代码修改，继续往下翻页，可以看到后面的 commit 改动。

如果只想查看指定 commit 的改动，可以使用 `git show` 命令。例如，git show 595bd27 只显示出 595bd27 这个 commit 自身的改动，翻看到最后就结束打印，不会继续往下显示后面 commit 的改动。

## 查看提交记录改动的文件名
使用 `git log --name-status` 命令来查看提交记录改动的文件名，但不会打印具体的改动，方便查看改动了哪些文件。
查看 man git-log 对 --name-status 的说明如下：
> **--name-status**  
Show only names and status of changed files.

## 只查看前面几条提交信息
使用 `git log` 命令查看使用提交记录，会显示全部的提交信息。如果只想查看前面几条提交信息，可以执行
 `git log -<number>` 命令，number 参数值是数字，指定要查看多少条信息，例如 1、2、3 等。查看 man git-log 对 `-<number>` 选项说明如下：
> **-\<number\>, -n \<number\>, --max-count=\<number\>**  
Limit the number of commits to output.

例如，`git log -3` 命令会只列出前面三条提交信息。注意，这个命令并不是只列出第三个提交信息。  
可以把 `-<number>` 选项和其他选项结合使用。例如 `git log -p -3` 命令只查看前面三条提交信息的具体改动。

## 在git log中显示committer信息
`git log` 命令默认显示的里面只有author，没有committer，类似于下面的信息：
```bash
$ git log
commit b932a847f564c441d68fe954b19b2b275fd1e38d
Author: John <john@xxxx.com>
Date:   Mon Oct 21 16:18:09 2019 +0800

    hello release
```
如果要显示committer的信息，可以使用 `--pretty=full` 选项。例如下面显示的信息：
```bash
$ git log --pretty=full
commit b932a847f564c441d68fe954b19b2b275fd1e38d
Author: John <john@xxxx.com>
Commit: John <john@xxxx.com>

    hello release
```
查看 man git-log 对 --pretty 选项说明如下：
> **--pretty[=\<format\>], --format=\<format\>**  
Pretty-print the contents of the commit logs in a given format, where \<format\> can be one of oneline, short, medium, full, fuller, email, raw and format:\<string\>.

默认的 medium 格式样式如下：
```
    medium
        commit <sha1>
        Author: <author>
        Date:   <author date>

        <title line>

        <full commit message>
```
可以显示 committer 信息的 full 格式样式如下：
```
    full
        commit <sha1>
        Author: <author>
        Commit: <committer>

        <title line>

        <full commit message>
```
这里的 author 和 committer 的区别是，author 是进行这个修改的人，而 committer 是把这个修改提交到git仓库的人。  
一般来说，我们自己修改代码，然后执行 git commit，那么既是 author，又是 committer。  
如果别人用 git format-patch 生成 git patch，在patch文件里面会包含修改者的名称和邮箱信息。例如：
```bash
From 033abaaecd9a3133cfcc028726aa37ebdbe6bff4 Mon Sep 17 00:00:00 2001
From: Jobs <jobs@xxxx.com>
Date: Mon, 21 Oct 2019 16:18:09 +0800
Subject: [PATCH] hello release
```
我们拿到这个patch文件，用 git am 命令把patch合入本地仓库，那么 author 是这个patch文件的修改者 *Jobs*，而 committer 是我们自己。

## 只查看某个人的提交历史
使用 `git log --author=<pattern>` 命令来查看某个作者的提交历史。  
使用 `git log --committer=<pattern>` 命令来查看某个提交者的提交历史。  
查看 man git-log 对这两个选项的说明如下：
> **--author=\<pattern\>, --committer=\<pattern\>**  
Limit the commits output to ones with author/committer header lines that match the specified pattern (regular expression). With more than one --author=\<pattern\>, commits whose author matches any of the given patterns are chosen (similarly for multiple --committer=\<pattern\>).

即，所给的 pattern 参数可以用正则表达式来匹配特定模式。举例如下：  
使用 `git log --author=John` 查看 John 的上库信息，如果有多个人名都带有 John，会匹配到多个人的提交历史。  
使用 `git log --author=john@xxxx.com` 来查看 `john@xxxx.com` 这个邮箱的提交历史。  
使用 `git log --author=@xxxx.com` 来查看 `@xxxx.com` 这个邮箱后缀的提交历史。

## 查找特定的commit信息
使用 `git log --grep=<pattern>` 命令在commit信息中查找指定的内容。查看 man git-log 对 --grep 的说明如下：
> **--grep=\<pattern\>**  
Limit the commits output to ones with log message that matches the specified pattern (regular expression). With more than one --grep=\<pattern\>, commits whose message matches any of the given patterns are chosen (but see --all-match).

这里说的commit信息指的是在执行 git commit 命令时填写的信息，不包含文件改动的内容。  
例如，为了方便标识提交的修改对应哪个模块，要求在commit信息里面写上模块名，假设应用UI代码的模块名是APPLICATION_UI，就可以使用 `git log --grep=APPLICATION_UI` 来过滤出所有 APPLICATION_UI 模块的提交历史。

# 使用 git commit --amend 修改历史 commit 信息
在一些受管控的项目上，提交代码到 git 服务器后，还需要经过审核确认才正式合入版本，一般常用 gerrit 来进行审核确认操作。

如果提交的代码审核不通过，需要再次修改提交。由于是修改同一个问题，我们可能不希望生成多个 commit 信息，会显得改动分散，看起来改动不完善，所以想要在本地已有的 commit 信息上再次提交改动，而不是在已有的 commit 上再新增一个 commit。

使用 `git commit --amend` 命令可以达到在现有最新 commit 上再次提交改动的效果。

在本地提交改动后，我们再次修改代码，执行 git add 命令添加改动，如果执行 `git commit -m` 命令，默认会创建新的空 commit 信息，填写相应的修改说明，提交之后，会新增一个 commit 信息；而执行 `git commit --amend` 命令会弹出当前最新 commit 的信息，我们可以修改这个信息，也可以不修改，提交之后，用 git log 命令查看，会看到没有增加新的 commit，原先 commit 的 hash 值也没有变，这一次的修改是跟之前的修改一起提交的。

查看 man git-commit 对 --amend 选项的关键说明如下：
> **--amend**  
Replace the tip of the current branch by creating a new commit.

即，`--amend` 选项创建一个新的 commit 来替换当前最新的 commit，如同当前最新的 commit 信息被修改了一样。

# 解决git status显示中文文件名乱码问题
使用 git status 查看有改动但未提交的中文文件名时，发现会显示为一串数字，没有显示中文的文件名。具体如下所示：
```bash
$ git status
# 位于分支 master
# 尚未暂存以备提交的变更:
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#   修改:      "\224\257\346\216\247\345\210\266\346\265\201.txt"
```
解决方案是设置git的 *core.quotepath* 选项为false：  
`git config --global core.quotepath false`

查看git在线帮助手册 (<https://git-scm.com/docs/git-config>)，里面对 *core.quotepath* 选项说明如下：
> **core.quotePath**  
Commands that output paths (e.g. ls-files, diff), will quote "unusual" characters in the pathname by enclosing the pathname in double-quotes and escaping those characters with backslashes in the same way C escapes control characters (e.g. \t for TAB, \n for LF, \\ for backslash) or bytes with values larger than 0x80 (e.g. octal \302\265 for "micro" in UTF-8). If this variable is set to false, bytes higher than 0x80 are not considered "unusual" any more. Double-quotes, backslash and control characters are always escaped regardless of the setting of this variable. A simple space character is not considered "unusual". Many commands can output pathnames completely verbatim using the -z option. The default value is true.

即，*core.quotepath* 选项为true时，会对中文字符进行转义再显示，看起来就像是乱码。  
设置该选项为false，就会正常显示中文。  
在一些终端上还要检查它自身的语言设置，看终端是否可以正常显示中文。

# 强制覆盖服务器的git log信息
当我们使用 git reset 命令回退本地 git log 显示的commit信息后，使用 `git push` 提交改动到远端服务器会报错，打印类似下面的错误信息：
> 提示：更新被拒绝，因为您当前分支的最新提交落后于其对应的远程分支。

此时，如果想强制用本地git log的commit信息覆盖服务器的git log，可以使用 `git push -f` 命令。查看 man git-push 对 -f 选项说明如下：
> **-f, --force**  
Usually, the command refuses to update a remote ref that is not an ancestor of the local ref used to overwrite it.  
    This flag disables these checks, and can cause the remote repository to lose commits; use it with care.

# 删除远端服务器分支
可以使用下面两个命令来删除远端服务器上的分支：
- `git push origin :<branchName>`
> 在冒号":"前面是要推送的本地分支名，这里没有提供，相当于推送一个本地空分支。在冒号":"后面推送到服务器的远端分支名。  
这个命令推送一个空分支到远端分支，相当于远端分支被置为空，从而删除 branchName 指定的远端分支
- `git push origin --delete <branchName>`
> 这个命令本质上跟 `git push origin :<branchName>` 是一样的。查看 man git-push 对 --delete 选项说明如下：  
> **-d --delete**  
All listed refs are deleted from the remote repository. This is the same as prefixing all refs with a colon.  
> 即，--delete 选项相当于推送本地空分支到远端分支。

# 设置git命令别名
我们可以使用 git config 命令来设置git命令别名，减少输入。例如下面的命令设置 st 为 status 命令的别名：
```bash
$ git config --global alias.st status
```
设置之后，执行 `git st` 相当于执行 `git status` 命令。

在Linux系统上，如果有多个命令都需要设置别名，可以直接编辑home目录的下 `.gitconfig` 文件，手动添加如下设置项：
```
[alias]
    co = checkout
    ci = commit
    st = status
    l = log --stat
    b = branch
```
可以看到，不但可以为命令设置别名，还可以在命令后面加上选项。

# 在git pull时只拉取当前branch的信息
执行 `git pull`命令默认会拉取远端服务器上的改动、以及各个branch和tag的信息。当远端服务器上有新增的branch或tag，就会拉取到，并打印出来，有时候会打印很多这些信息。

如果想要只拉取当前branch的信息，需要加上远端仓库名和branch名作为参数。例如，将远端origin仓库的master分支合并到本地当前branch，可以执行下面的命令：
```bash
$ git pull origin master
```
如果还要不拉取tag信息，可以再加上 --no-tags 选项：
```bash
$ git pull --no-tags origin master
```
使用这种方法更新代码，即使远端服务器上有新增的branch，在本地执行 `git branch -r` 命令也不会看到新增的branch。

# 查看文件mode属性是否发生改变
当修改文件时，特别是在Windows下修改Linux的文件，可能会改变文件的mode属性值，例如从 644 变成 755，然后使用 git add 命令添加文件，会提示 *file mode change*，但是这个提示不太明显，容易被忽略。

在执行 git add 命令之前，如果想查看文件mode属性是否发生改变，可以使用 git diff 命令的 --summary 选项。查看 man git-diff 对 --summary 选项的说明如下：
> **--summary**  
Output a condensed summary of extended header information such as creations, renames and mode changes.

例如，如果本地文件的mode改变了，执行 `git diff --summary` 命令，会看到类似下面的信息：
> mode change 100755 => 100644 file_name

这个命令不会列出文件内容的改动，而只列出文件mode变化，方便只查看文件mode的变化。

对于已经执行过 git commit 提交的文件，在 git log 命令里面也可以使用 --summary 选项查看已经提交的文件mode变化。

# 执行 git pull 时是否打印改动的文件信息
在公司的Android代码目录里面，使用 `git pull` 命令，发现不会打印发生改变的文件信息。例如不会打印类似下面的信息：
```bash
Fast-forward
 res/values-zh-rCN/strings.xml                     | 5 +++--
 res/values/strings.xml                            | 4 ++--
 src/com/android/SoftwarePreferenceController.java | 2 +-
 3 files changed, 6 insertions(+), 5 deletions(-)
```
但是之前公司的 git pull 命令会打印改动的文件信息，现在想要确认出现这种差异的原因。

经过排查，这是因为 git pull 执行的是 git rebase 所引起。  
在git仓库目录下执行 `git config -l` 命令，看到有如下配置：
> pull.rebase=true

这是当前代码目录的git仓库里面自行配置的。使用 `git config --global -l` 查看没有这个全局配置.

查看 man git-config 命令的说明，*pull.rebase=true* 表示 git pull 使用 git rebase，而不是使用 git merge：
> **pull.rebase**  
When true, rebase branches on top of the fetched branch, instead of merging the default branch from the default remote when "git pull" is run.

再查看 man git-rebase 命令的说明：
> **rebase.stat**  
Whether to show a diffstat of what changed upstream since the last rebase. False by default.

> **--stat**  
Show a diffstat of what changed upstream since the last rebase. The diffstat is also controlled by the configuration option rebase.stat.

> **-n, --no-stat**  
Do not show a diffstat as part of the rebase process.

即，git rebase 默认不会打印发生变动的文件名。如果想要打印，需要添加 `--stat` 选项。

作为对比，git merge 命令默认会打印修改前后的文件名。下面是 man git-merge 命令的说明：
> **--stat, -n, --no-stat**  
Show a diffstat at the end of the merge. The diffstat is also controlled by the configuration option merge.stat.  
With -n or --no-stat do not show a diffstat at the end of the merge.  

> **merge.stat**  
Whether to print the diffstat between ORIG_HEAD and the merge result at the end of the merge. True by default.

总的来说，git pull 命令其实是先使用 git fetch 获取远端代码，然后调用 git merge 把获取到的远端代码合并到本地分支。如果提供了 --rebase 选项，会用 git rebase 来替代 git merge。配置 pull.rebase 为true，git pull 默认会用 git rebase。  
当 git pull 使用 git merge 时，默认会打印变动的文件信息。  
而 git pull 使用 git rebase 时，默认不会打印变动的文件信息。

如果不确定使用的是 git merge、还是 git rebase，又想要每次执行 git pull 都能打印变动的文件信息，可以加上 `--stat` 选项，例如 `git pull --stat`。

# git pull的多次测试方法
执行 git pull 后，本地仓库已经跟远端服务器保存一致，如果想要多次测试 git pull 的效果，需要先让本地仓库代码落后于远端服务器代码。具体方法说明如下。

先执行 `git checkout -b local_branch_name HEAD~2` 命令，或者执行 `git checkout -b local_branch_name commit` 命令：
- HEAD 指向当前最新版本，`HEAD~1` 指回退一个版本，也就是上一个版本，可以简写为 `HEAD~`，`HEAD~2` 指回退两个版本。依次类推，可以指定本地分支要落后于服务器多少个版本。
- commit 参数是 git commit 的hash值，通过 commit 参数来指定要回退到哪一个提交。
- 这两个命令的作用是，创建一个新的分支，且新分支的代码已经落后于远端服务器代码

接下来用 `git pull remote_repository remote_branch_name` 来更新代码，就能看到 git pull 的更新效果。这时不能只执行 `git pull` 命令，否则会报下面的错误：
> fatal: No remote repository specified.  Please, specify either a URL or a remote name from which new revisions should be fetched.

当这样pull之后，当前本地分支代码已经最新，再次pull就没有新的改动。如果想要再测试pull效果，可以再创建新的本地分支。

上面执行 git checkout 的时候也可以先不加 -b 选项，先执行 `git checkout HEAD~2` 命令或者 `git checkout commit` 命令，执行之后，当前就不处于任何分支下，git 提示可以用 `git checkout -b new_branch_name` 来创建新的本地分支，然后再用上面的方法来进行pull。

如果不想多次创建新的分支，想在当前的本地分支上多次测试 git pull，可以参考下面方法。

先执行 `git reset commit` 命令，commit 参数是 git commit 的hash值，指定要回退到哪一个提交。那么本地代码会被回退，用 git status 命令查看，会看到有一些文件还没有被提交，此时无法执行 git pull，会提示 "Cannot pull with rebase: You have unstaged changes."

接下来需要重新执行 git add、git commit 进行提交，之后就可以用 `git pull remote_repository remote_branch_name` 命令来更新。此时，如果 git pull 用的是 git merge 就会提示 branch merge，自动弹出 merge comment，需要手动确认。
如果 git pull 用的是 git rebase，不会提示 branch merge，不需要填写或手动确认 merge comment.

# git pull 命令的选项顺序问题
实际使用 git pull 的时候，遇到这样一个问题，当把 --stat 写在 --no-tags 后面执行会报错：
```bash
$ git pull --no-tags --stat aosp remote_branch_name
error: unknown option `stat'
```
但是把 --stat 和 --no-tags 的顺序调换，执行 git pull 命令不会报错：
```bash
$ git pull --stat --no-tags aosp remote_branch_name
From platform/packages/apps/Settings
 * branch            remote_branch_name -> FETCH_HEAD
Current branch remote_branch_name is up to date.
```
即，--stat 必须写在 --no-tags 前面，否则 git pull 就会报错。查看 man git-pull 的帮助说明，对此说明如下：
> More precisely, git pull runs git fetch with the given parameters and calls git merge to merge the retrieved branch heads into the current branch. With --rebase, it runs git rebase instead of git merge.

> Options meant for git pull itself and the underlying git merge must be given before the options meant for git fetch.

而 --stat 是 merge/rebase 的选项，--no-tags 是 fetch 的选项。基于上面说明，merge选项必须写在fetch选项前面。所以当 --stat 写在 --no-tags 后面时，git pull 会报错，它应该是把 --stat 传给 git fetch 处理，但是 git fetch 没有这个选项，导致报错提示 "unknown option".

# 打印且只打印本地分支名
使用 git branch 查看分支，会打印仓库下的所有分支名，通过 '*' 星号来标识当前分支。  
如果想打印且打印当前本地分支名，可以用 `git symbolic-ref --short HEAD` 命令。
```bash
$ git branch
* curent_branch_xxx
  enable_func
$ git symbolic-ref --short HEAD
curent_branch_xxx
```
也可以使用 `git rev-parse --abbrev-ref HEAD` 命令来打印且只打印当前分支名。
```bash
$ git rev-parse --abbrev-ref HEAD
curent_branch_xxx
```
这两个命令可用在shell脚本中获取当前分支名，做一些自动化处理。

## git symbolic-ref --short HEAD
使用 man git-symbolic-ref 查看该命令的帮助信息，说明如下：
> **git symbolic-ref: Read, modify and delete symbolic refs**  
**git symbolic-ref [-m \<reason\>] \<name\> \<ref\>**  
**git symbolic-ref [-q] [--short] \<name\>**  
Given one argument, reads which branch head the given symbolic ref refers to and outputs its path, relative to the .git/ directory. Typically you would give HEAD as the \<name\> argument to see which branch your working tree is on.

> --short  
When showing the value of <name> as a symbolic ref, try to shorten the value, e.g. from refs/heads/master to master.

即，git symbolic-ref 命令可以查看 symbolic ref 的信息。而 HEAD 就是一个 symbolic ref 的名称，可用于查看当前工作分支。

使用 man git 可以查看到 git refs、HEAD 的一些说明：
> Named pointers called refs mark interesting points in history. A ref may contain the SHA-1 name of an object or the name of another ref. Refs with names beginning ref/head/ contain the SHA-1 name of the most recent commit (or "head") of a branch under development. SHA-1 names of tags of interest are stored under ref/tags/. A special ref named HEAD contains the name of the currently checked-out branch.

查看 github 网站上的开发者文档 (<https://developer.github.com/v3/git/refs/>)，有如下说明：
> A Git reference (git ref) is just a file that contains a Git commit SHA-1 hash. When referring to a Git commit, you can use the Git reference, which is an easy-to-remember name, rather than the hash. The Git reference can be rewritten to point to a new commit. A branch is just a Git reference that stores the new Git commit hash.

即，git ref 保存了git commit hash值，git ref 本身有一个名称，被用作分支名。一个分支其实就是一个 git ref。不同分支的差异是分支head指向的 git commit hash 不同。

HEAD 是一个指向当前工作分支的 git ref，切换工作分支，会改变 HEAD 的指向。这个文件在git仓库中的路径是 `.git/HEAD`，可以用cat命令查看它的内容，下面的 HEAD 指向 master分支：
```bash
$ cat .git/HEAD
ref: refs/heads/master
```
各个 git ref 存在 `.git/refs/` 目录下，本地分支head存在 `.git/refs/heads/` 目录下：
```bash
$ ls .git/refs/heads
great  master
```
可以看到，git branch 的分支名跟 `.git/refs/heads/` 目录下的文件名相同：
```bash
$ git branch
  great
* master
```
结合这几个说明，对 `git symbolic-ref --short HEAD` 命令分解说明如下：
- `git symbolic-ref` 命令可以解析获取 git ref 的信息
- `--short` 表示获取 symbolic ref 的名称
- `HEAD` 是指向当前工作分支的 git ref，解析HEAD文件信息，就能获取当前分支名

## git rev-parse --abbrev-ref HEAD
`git rev-parse --abbrev-ref HEAD` 命令也能获取当前分支名。查看 man git-rev-parse 的说明如下：
> **git rev-parse: Pick out and massage parameters**  
**git rev-parse [ --option ] \<args\>...**  
**--abbrev-ref[=(strict|loose)]**  
A non-ambiguous short name of the objects name.

在man手册里面没有具体说明这个命令的表现是什么。在git的在线参考链接上找到一些描述: <https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection>
> If you want to see which specific SHA-1 a branch points to, or if you want to see what any of these examples boils down to in terms of SHA-1s, you can use a Git plumbing tool called rev-parse; basically, rev-parse exists for lower-level operations and isn’t designed to be used in day-to-day operations.

大致理解，git rev-parse 是一种git管道(plumbing)工具，可用于处理SHA-1 hash值。  
我们在上面看到，`git rev-parse --abbrev-ref HEAD` 打印当前分支名，--abbrev-ref 表示输出所给对象不会混淆的短名，类似于 git symbolic-ref 的 --short 选项的输出结果。

不加 --abbrev-ref 时，会打印出HEAD对应的hash值：
```bash
$ git rev-parse HEAD
8ebf0117f9545187d3368adc1ce629608214984a
```

这两个命令都可以打印当前分支名，如果当前处于未命名分支下面时，它们的行为会有一些差异。下面用 git branch 命令查看，可以看到当前处在分离的HEAD状态下，当前分支没有命名：
```bash
$ git branch
* (detached from 65c6917)
  great
  master
```
在这种情况下，HEAD不是符号引用，git symbolic-ref 会以错误退出：
```bash
$ git symbolic-ref --short HEAD
fatal: ref HEAD is not a symbolic ref
```
而 git rev-parse –abbrev-ref 将HEAD解析为自身：
```bash
$ git rev-parse --abbrev-ref HEAD
HEAD
```

# 获取当前本地分支对应的远端服务器分支
可以使用下面命令查看本地分支在远端服务器的分支名：
```bash
$ git rev-parse --abbrev-ref local_branch_name@{upstream}
```
把 local_branch_name 换成要查询的本地分支名，例如 master 等。下面通过例子来说明这个命令各个参数的含义。

先创建一个新的本地分支，名为 *new_local_branch*，关连到远端服务器的 *Remote_Branch_U* 分支：
```bash
$ git checkout -b new_local_branch aosp/Remote_Branch_U
Branch new_local_branch set up to track remote branch Remote_Branch_U from aosp.
Switched to a new branch 'new_local_branch'
```
查看本地分支 new_local_branch 在远端服务器的分支名：
```bash
$ git rev-parse --abbrev-ref new_local_branch@{upstream}
aosp/Remote_Branch_U
```
如果所给的本地分支名没有关连到远端服务器分支，会打印报错信息：
```bash
$ git rev-parse --abbrev-ref great@{upstream}
fatal: No upstream configured for branch 'great'
```
**注意**：`@{upstream}` 这一整串本身是命令的一部分，直接输入即可，不是要把 upstream 或者 
{upstream} 替换成远端服务器仓库名。查看 man git-rev-parse 有如下说明：
> **\<branchname\>@{upstream}, e.g. master@{upstream}, @{u}**  
The suffix @{upstream} to a branchname (short form <branchname>@{u}) refers to the branch that the branch specified by branchname is set to build on top of. A missing branchname defaults to the current one.

即，`@{upstream}` 可以缩写为 `@{u}`。如果不提供分支名，默认用当前本地分支名。  
另外，如果不加 --abbrev-ref 选项，会打印分支head的hash值，而不是打印分支名。
```bash
$ git rev-parse --abbrev-ref new_local_branch@{u}
aosp/Remote_Branch_U
$ git rev-parse --abbrev-ref @{u}
aosp/Remote_Branch_U
$ git rev-parse @{u}
66355f171f5ba7dbc66465e761b97afe2395b06e
```
这个命令可在shell脚本中自动获取到远端服务器分支名，而不是只能用默认值、或者要手动输入分支名。
