# 记录 repo 的使用笔记

# 在Android源码中执行repo forall -c时列出git仓库名称
Android 源码使用 repo 命令来管理所有 git 仓库，当使用 repo forall -c 命令在所有 git 仓库上执行指定的 git 命令时，默认不会列出每个git 仓库的 project 名称。

**例如使用 repo forall -c git status 命令来查看各个 git 仓库的改动时，打印出来的内容没有包含仓库的路径名，有时候根本看不出来某些改动发生在哪个仓库下**。

**如果想要打印出 git 仓库的名称，可以使用 repo forall 的 -p 选项**。查看 repo help forall 命令打印的帮助信息，对 -p 选项说明如下：
> **-p**  
Show project headers before output

根据帮助信息里面的例子，建议先写 -p，再写 -c，即 `repo forall -p -c`，执行该命令，会打印类似下面的信息：
```bash
$ repo forall -p -c git pull
project buildroot/
Already up-to-date.
project tools/
Already up-to-date.
```
可以看到，它先打印 git仓库信息 “project buildroot/”，再打印该仓库的状态。

如果不加 -p 选项， 执行 repo forall -c git pull 命令，只会打印下面的信息：
```bash
$ repo forall -c git pull
Already up-to-date.
Already up-to-date.
```
可以看到，只提示 “Already up-to-date.”，没有说明是哪个仓库，如果某个仓库更新代码，比较难知道是哪个仓库更新了代码。相比之下，加了 -p 选项的打印了 git 仓库信息，比较直观。

**注意**：实际使用中遇到了使用 -p 选项没有打印一些报错信息的例子，具体描述如下，下面在 `#` 后面的内容是注释说明，不是命令内容的一部分。
```bash
$ ls # 下面的 brandy 目录是后来新增的,原先的repo没有跟踪这个仓库
brandy buildroot tools
$ repo forall -c git pull # 执行该命令,会看到报错信息,提示有个仓库没有指定
fatal: No remote repository specified. Please, specify either a URL or a
remote name from which new revisions should be fetched.
Already up-to-date.
Already up-to-date.
Already up-to-date.
$ repo forall -p -c git pull
project buildroot/
Already up-to-date.
project tools/
Already up-to-date.
```
上面的 `repo forall -p -c git pull`  命令，加了 -p 选项，没有看到打印 “fatal: No remote repository specified” 的报错信息，应该是被过滤掉了。

可见，使用 -p 选项可能会漏掉某些报错信息。但不加 -p 又打印不出具体的仓库名字。所以要根据实际情况来选择是否使用 -p 选项。

如果仓库 project 不经常发生变动，可以使用 -p 选项。确认新增仓库 project 时，先不加 -p 选项，更新代码，下载好新的 project 后，就可以继续使用 -p 选项。

# repo status 指定多线程数目
在用 repo status 命令查看 Android 源码的所有 git 仓库改动时，一般执行起来都比较慢，像是单线程执行，但实际上默认会启用2个选项来同步执行。

我们可以使用 repo status 的 -j 选项来指定执行时的多线程数目。查看 repo help status 对 -j 选项的帮助说明如下：
> **-j JOBS, --jobs=JOBS**  
number of projects to check simultaneously

> Description  
The -j/--jobs option can be used to run multiple status queries in parallel.

即，可以使用该选项来加快 repo status 命令的执行速度。例如 `repo status -j 4`。

查看 `.repo/repo/subcmds/status.py` 的源码，如果没有提供 -j 选项，默认启用2个线程来执行，如下面的 `default=2` 所示：
```python
p.add_option('-j', '--jobs',
             dest='jobs', action='store', type='int', default=2,
             help="number of projects to check simultaneously")
```

**注意**：repo status 命令启用多线程执行时，打印出来的信息概率会出现错乱，类似于下面的效果：
```
project test/vts-testcase/vndk/  branch local_branchproject toolchain/binutils/
branch local_branch
```
可以看到，上面的 *project toolchain/binutils/* 本该另起一行打印，但是它跟前面的内容打印在了同一行。这是多线程同时输出导致的错乱。

**之前编写 shell 脚本来过滤 repo status 命令的打印结果，想要打印只且打印发生了改动的信息，就遇到了这种输出信息错乱影响解析的情况**。

当时还奇怪没有用 -j 选项来指定启用多线程，为什么会有这个问题，查看 repo help status 的帮助信息，也没有说明默认会启用多线程。后来查看了上面的 repo 源码，才确认默认会启用2个线程来同步执行。

为了避免这种问题，建议在 shell 脚本里面用 `repo status -j 1` 命令明确指定为单线程执行，避免打印的信息错乱而影响解析。但是这样执行会比较慢，根据实际需求来取舍。

# 介绍 repo sync 同步的是远端服务器哪个分支
查看 repo help sync 命令的帮助说明，该命令的格式如下：
- Usage: repo sync [\<project\>...]

可以看到，它没有提供参数来指定要同步的远端服务器分支。那么在执行 repo sync 时，它同步的是远端服务器的哪个分支？

实际上，`repo sync` 默认同步在 `repo init` 时由 -b 选项指定的分支，这也是 repo 所跟踪的分支。

**注意**：如果本地的 git 仓库切换过分支，当前分支名和 `repo init -b` 指定的分支名不一样，那么执行 repo sync 会改变本地分支指向，需要注意到这个分支的变化，避免后续操作错分支。

下面具体举例说明 `repo sync` 后本地分支的变化，在这个例子一开始，本地当前分支名是 *branch_m*，这不是 `repo init -b` 所指定的分支。

1. 使用 git branch 命令，打印出当前分支名是 branch_m：
```bash
$ git branch
  other_branch_xxx
* branch_m
```
2. 在当前代码目录下执行 repo sync 命令：
```bash
$ repo sync .
Fetching project platform/packages/apps/Settings
packages/apps/Settings/: leaving branch_m; does not track upstream
```
3. 再次执行 git branch 命令，会看到当前处于没有命名的分支下：
```bash
$ git branch
* (detached from f15a7be)
  other_branch_xxx
  branch_m
```

基于这个现象，建议在本地所有分支都关联到远端服务器分支时，才用 `repo sync` 来同步代码。

如果本地当前分支没有关联到远端服务器分支，使用 `repo sync` 同步之后，分支指向会发生变化，后续修改代码，并不是位于原来分支下面，如果要提交修改到原来的分支，会提示需要 merge，容易造成代码冲突，比较麻烦。

# repo sync 时减少同步时间和减小代码空间
在使用 repo sync 同步 Android 源码时，可以添加一些选项来减少同步时间和要下载的代码空间。具体的命令是 `repo sync -c --no-tags --prune -j 4`。

查看 repo help status 的帮助信息，对所给的各个选项具体说明如下：
- **-c, --current-branch**  fetch only current branch from server.  
这个选项指定只获取执行 repo init 时 -b 选项所指定的分支，不会获取远端服务器的分支信息。  
例如服务器上新增了其他分支，使用 -c 选项同步后，在本地 git 仓库执行 `git branch -r` 命令看不到服务器新增的分支名。如果不加 -c 选项，那么同步的时候，会打印 *[new branch]* 这样的信息，使用 `git branch -r` 命令可查看到服务器新增的分支。
- **--no-tags**             don't fetch tags.  
该选项指定不获取服务器上的tag信息。
- **--prune**               delete refs that no longer exist on the remote.  
如果远端服务器已经删除了某个分支，在 repo sync 时加上 `--prune` 选项，可以让本地仓库删除对这个分支的跟踪引用。  
查看 repo 的 `.repo/repo/project.py` 源码，这个选项实际上是作为 git fetch 命令的选项来执行。查看 man git-fetch 对自身 `--prune` 选项的说明如下，可供参考：  
> -p, --prune  
  After fetching, remove any remote-tracking references that no longer exist on the remote.
- **-j JOBS, --jobs=JOBS**  projects to fetch simultaneously (default 2).  
指定启用多少个线程来同步。  
例如上面的 `-j 4` 指定用4个线程来同步。如果没有提供该选项，默认是用2个线程。

总的来说，在 `repo sync -c --no-tags --prune -j 4` 命令中，使用 -c 和 --no-tags 选项可以减少需要同步的内容，从而减少要占用的本地代码空间，也可以减少一些同步时间。

使用 -j 选项来指定启用多线程进行同步，可以加快执行速度，也就减少了同步时间。  

使用 --prune 选项去掉已删除分支的跟踪引用，一般不会用到，这个选项可加可不加。

# 详解 repo sync 如何指定只同步Android源码的某个仓库
执行 `repo sync` 命令默认会同步 Android 源码的所有 git 仓库。如果想要单独同步一个、或多个 git 仓库，可以提供一个、或多个 project 参数来指定要同步的 git 仓库路径。具体命令格式如下：
- Usage: repo sync [\<project\>...]

关键是，如何知道某个 git 仓库对应的 project 参数值是什么，是 git 仓库所在的子目录名，还是 git 仓库在 repo 中保存的完整目录路径，或是其他？ 

经过实际测试发现，这里提供的 project 参数值是基于当前 shell 的工作目录寻址到目标 git 仓库的目录路径，而不是 git 仓库在 repo 中保存的完整目录路径，也不是 git 仓库的子目录名。

例如执行 `repo status` 命令打印了下面的 git 仓库信息：
```
project android/packages/apps/Settings/ branch branch_m
 -m     res/values-zh-rCN/strings.xml
```
可以看到，这个 git 仓库在 repo 中保存的完整路径是 *android/packages/apps/Settings/*，所在的子目录名是 *Settings*。

我们使用 `cd` 命令进入到 *android* 子目录，测试如下。

1. 在当前的 *android* 子目录下执行 `repo sync android/packages/apps/Settings/` 命令，会执行报错：
```bash
$ repo sync android/packages/apps/Settings/
error: project android/packages/apps/Settings/ not found
```
即，当前 shell 的工作目录在 *android* 子目录下，把要同步的 git 仓库在 repo 中的完整目录路径 *android/packages/apps/Settings/* 作为参数传给 repo sync 命令，会执行报错，基于这个路径不能定位到要同步的 git 仓库。

2. 传入 *packages/apps/Settings/* 这个路径可以正常执行：
```bash
$ repo sync packages/apps/Settings/
Fetching project platform/packages/apps/Settings/
```
基于当前所在的 *android* 子目录，可以正常定位到所给 *packages/apps/Settings/* 路径下的 git 仓库。

3. 使用 `cd` 命令进入到 *android/packages/apps/Settings/* 目录下，然后执行 `repo sync .` 命令不会报错：
```bash
$ repo sync .
Fetching project platform/packages/apps/Settings/
```

基于这几个测试结果可以发现，repo sync 后面跟着的 project 参数值应该是基于当前 shell 工作目录能够寻址到该 git 仓库的目录路径，类似于 cd 命令的路径寻址方式。

查看 repo help sync 命令打印的帮助信息，对 project 参数的说明如下，符合上面的验证结果：
> 'repo sync' will synchronize all projects listed at the command line. Projects can be specified either by name, or by a relative or absolute path to the project's local directory. If no projects are specified, 'repo sync' will synchronize all projects listed in the manifest.

即，可以提供 project name、或者提供能够寻址到该 project 本地目录的相对路径或绝对路径来指定要同步的 project。

这里说的 project name 并不是 git 仓库的子目录名，具体值要在 repo 命令生成的 `.repo/` 目录下查看。例如，查看 `.repo/` 目录下的 manifest.xml 文件，有如下信息:
```xml
<project groups="p-fs-release,pdk-fs" name="platform/packages/apps/Settings"
    path="android/packages/apps/Settings"  />
```

可以看到，*Settings* 子目录的 git 仓库在 repo 中保存的完整路径是 *android/packages/apps/Settings*，跟上面例子打印的信息相符。而它的 name 是 *platform/packages/apps/Settings*。

那么只要当前 shell 的工作目录是 Android 源码的子目录，不管在哪个子目录下，执行 `repo sync platform/packages/apps/Settings` 命令都会同步 Settings 这个 git 仓库的代码，这里不再举例，可以自行验证。

# repo sync 的 -d 选项说明
使用 `repo sync` 命令来同步远端服务器的 Android 代码，如果本地修改了代码但还没有 commit，会提示无法 sync：
> error: android/frameworks/base/: contains uncommitted changes

此时，可以使用 `git reset` 命令丢弃本地修改，然后再执行 `repo sync` 来同步代码。

如果想要不丢失本地修改，强制同步远端服务器代码，可以加上 `-d` 选项，`repo sync -d` 命令会将 HEAD 强制指向 repo manifest 版本，而忽略本地的改动。

查看 repo help sync 的帮助信息，对 -d 选项的说明如下：
> **-d, --detach**  
detach projects back to manifest revision

**注意**：加上 `-d` 选项只表示忽略本地改动，可以强制同步远端服务器的代码，但是本地修改的文件还是保持改动不变，不会强制覆盖掉本地修改。而且同步之后，本地的分支指向会发生变化，不再指向原来的分支。具体举例如下。

1. 下面是执行 `repo sync -d` 之前的分支信息：
```bash
$ git branch
* curent_branch_xxx
```

2. 下面是执行 `repo sync -d` 之后的分支信息：
```bash
$ git branch
* (detached from 715faf5)
  curent_branch_xxx
```
即，从远端服务器同步的代码，是同步到跟踪远端服务器的分支，还没有从 git 仓库把代码 checkout 到本地，而当前本地修改的代码处在未命名分支下，是不同的分支，互不干扰，才能在不丢弃本地修改的情况下，强制同步远端服务器代码。

3. 执行 `git status` 命令，可以看到本地还是有修改过且还没有 commit 的文件，同步远端服务器代码后，并不会强制覆盖本地文件的修改：
```bash
$ git status
HEAD detached at 715faf5
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
        modified:   vendor/chioverride/default/g_pipelines.h
        modified:   vendor/topology/g_usecase.xml
```
即，如果想要丢弃本地修改、让本地代码跟同步后的 git 仓库代码一致，`repo sync -d` 命令达不到这个效果。

另外，repo sync 有一个 `--force-sync` 选项，具体说明如下：
> **--force-sync**  
overwrite an existing git directory if it needs to point to a different object directory. WARNING: this may cause loss of data

从说明来看，像是可以强制同步，且可能丢失本地改动。但是实际测试发现，这个选项并不能强制覆盖本地的改动。如果本地文件发生改动，加上这个选项也是会 sync 报错：
```bash
$ repo sync --force-sync .
Fetching project tools/
error: tools/: contains uncommitted changes
```
同时提供 `-d` 和 `--force-sync` 两个选项，还是不能强制覆盖本地修改。

目前没有找到 `repo sync` 命令可以强制覆盖本地修改的选项。

# 用repo丢弃本地修改，强制同步远端服务器代码
目前所知，使用 repo sync 同步远端服务器代码，不能强制覆盖本地修改。如果想要强制覆盖本地修改，可以用 repo forall -c 来执行git丢弃本地修改的命令，git checkout 和 git reset 命令都可以丢弃本地修改。

一般来说，可以使用 `git checkout .` 命令来丢弃当前目录下的改动，但是实际执行 `repo forall -c git checkout .` 命令会报错：
```bash
$ repo forall -c git checkout .
error: pathspec '.' did not match any file(s) known to git.
```
而使用 `repo forall -c 'git reset --hard'` 命令不会报错，可以用这个命令来丢弃所有git仓库的本地改动，然后再用 repo sync 命令来同步远端服务器代码。

在实际项目开发时，可能会在本地创建开发分支，这个分支没有关连到服务器分支。我们同步远端服务器代码，可能只是想获取到最新代码。此时，并不想丢弃本地开发分支的改动。但是上面的 `repo forall -c 'git reset --hard'` 命令无法区分这种情况，会丢弃开发分支的改动。

下面这个看起来很复杂的命令就用于解决这个问题，它先判断当前本地分支在远端服务器存在同名分支时，才强制覆盖本地修改并pull服务器代码：
> repo forall -c "branch_name=$(git rev-parse --abbrev-ref HEAD) && git rev-parse --abbrev-ref ${branch_name}@{upstream} && git reset --hard && git pull --stat --no-tags" |& cat

对这个命令的各个部分说明如下。
- `repo forall -c`：对所有git仓库都执行后面的跟着的命令，这里使用双引号把多个git命令都括起来，作为一个整体传给repo，可以做一些复杂的操作。
- `branch_name=$(git rev-parse --abbrev-ref HEAD)`：这是bash shell的语法, `$(cmd)` 会执行 cmd 命令，得到它的输出，这里把输出结果赋值给 *branch_name* 变量。`git rev-parse --abbrev-ref HEAD` 命令输出且只输出本地分支名。
- `&&`：bash shell的与操作，前一个命令执行结果为true，才会执行下一个命令。
- `git rev-parse --abbrev-ref ${branch_name}@{upstream}`：`${branch_name}` 表示获取 *branch_name* 变量的值。*branch_name* 变量在前面被赋值为当前本地分支名。整个git命令获取本地分支在远端服务区分支名。如果获取不到，执行结果是false。
- `git reset --hard`：基于 `&&` 操作符的特性，如果上一个git命令获取不到本地分支在远端服务器的分支名，就不会往下执行。所以执行到这里时，说明本地分支名跟远端服务器分支名相同，我们假设本地开发分支没有关连到远端服务器分支，那么当前分支就不是开发分支，用 git reset 丢弃本地修改。
- `git pull --stat --no-tags`：同步服务器代码，--stat 表示要打印发生改动的文件信息，--no-tags 表示不获取远端服务器仓库的tag信息。如果需要pull远端服务器的tag，可以不加 --no-tags 选项。
- `|& cat`：把整个repo命令的标准输出、错误输出都重定向到 cat 命令。如果不重定向，repo 会用 less 命令来显示输出内容，需要手动按q退出才会继续输出，重定向到 cat 后，直接打印全部输出内容，不需要再手动进行其他操作。
