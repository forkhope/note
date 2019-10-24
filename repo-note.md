# 记录 repo 的使用笔记

# 执行 repo forall -c 命令时打印project名称
当使用 repo forall -c 命令在所有project上执行git命令时，默认不会打印出每个project名称。例如使用 `repo forall -c git status` 命令来查看各个project的改动时，打印出来的内容没有包含仓库的名字，有时候根本看不出来某些改动发生在哪个仓库下。

如果想要打印project名称，可以使用 repo forall 的 -p 选项。查看 repo help forall 命令打印的帮助信息，对 -p 选项说明如下：
> **-p**                  Show project headers before output

根据帮助信息里面的例子，建议先写 -p，再写 -c，即 `repo forall -p -c`。

**注意**：实际使用中遇到了使用 -p 选项没有打印一些报错信息的例子，具体描述如下，下面在 `#` 后面的内容是注释说明，不是命令内容的一部分。
```bash
$ ls       # 下面的 brandy 目录是后来新增的,原先的repo没有跟踪这个仓库
brandy    buildroot    tools
$ repo forall -c git pull  # 执行该命令,会看到报错信息,提示有个仓库没有指定
fatal: No remote repository specified.  Please, specify either a URL or a
remote name from which new revisions should be fetched.
Already up-to-date.
Already up-to-date.
Already up-to-date.
$ repo forall -p -c git pull    # 加了 -p 选项后,没有看到上面的报错信息,应
project buildroot/              # 应该是被过滤掉了.可见,使用 -p 选项可能会
Already up-to-date.             # 漏掉某些报错信息.但不加 -p 又打印不出具体
project tools/                  # 的仓库名字.所以要根据实际情况来选择是否使
Already up-to-date.             # 用 -p 选项.
```

# repo status 指定多线程数目
 "repo status"的 "-j" 选项
查看 repo help status 命令的帮助说明，里面提到 -j 选项可以指定执行时的多线程数目：
> **-j JOBS, --jobs=JOBS**  number of projects to check simultaneously

> Description  
The -j/--jobs option can be used to run multiple status queries in parallel.

所以，可以使用该选项来加快 repo status 命令的执行速度。例如 `repo status -j 4`。  
查看 `.repo/repo/subcmds/status.py` 的源码，如果没有提供 -j 选项，默认使用2个线程来执行：
```python
p.add_option('-j', '--jobs',
             dest='jobs', action='store', type='int', default=2,
             help="number of projects to check simultaneously")
```

**注意**：repo status 命令多线程执行时，打印出来的信息概率会出现错乱，类似于下面的效果：
```
project test/vts-testcase/vndk/         branch local_branchproject toolchain/binutils/
branch local_branch
```
可以看到，上面的 *project toolchain/binutils/* 本该另起一行打印，但是它跟前面的内容打印在了同一行。  
如果在shell脚本里面对 repo status 命令的打印结果进行过滤，这种情况下就会过滤出错。为了避免这种问题，建议在shell脚本里面用 `repo status -j 1 ` 命令指定为单线程执行，避免打印的信息错乱，方便解析。但是这样执行会比较慢，根据实际需求来取舍。

# repo sync 时减少同步时间和减小代码空间
在用 repo sync 命令同步服务器内容时，可以使用 `repo sync -c --no-tags --prune -j 4` 命令来减少同步时间和减小代码空间。查看 repo help status 里面对所给各个选项的说明如下：
- **-c, --current-branch**  fetch only current branch from server
> 这个选项指定只获取 repo init 时 -b 选项所指定分支，不会获取远端服务器的分支信息。例如服务器上新增了其他分支，使用 -c 选项同步后，本地执行 `git branch -r` 命令看不到服务器新增的分支名。如果不加 -c 选项，那么同步的时候，会看到 *[new branch]* 这样的信息，使用 `git branch -r` 命令可查看服务器新增的分支。
- **--no-tags**             don't fetch tags
> 该选项指定不获取服务器上的tag信息。
- **--prune**               delete refs that no longer exist on the remote
> 如果远端服务器已经删除了某个分支，在 repo sync 时加上 --prune 选项，可以让本地仓库删除对这个分支的跟踪引用。查看 repo 的 `.repo/repo/project.py` 源码，这个选项实际上是作为 git fetch 命令的选项来执行。查看 man git-fetch 对自身 --prune 选项的说明如下：  
> -p, --prune  
  After fetching, remove any remote-tracking references that no longer exist on the remote.
- **-j JOBS, --jobs=JOBS**  projects to fetch simultaneously (default 2)
> 使用多线程来同步，例如上面的 `-j 4` 指定用4个线程来同步。如果没有提供该选项，默认是用2个线程。

总的来说，使用 -c 和 --no-tags 选项可以减少需要同步的内容，从而减少同步时间，减小本地的代码空间。  
使用 -j 选项来使用多线程进行同步，加快执行速度，也就减少了同步时间。  
使用 --prune 选项去掉已删除分支的跟踪应用，一般不会用到，这个选项可加可不加。

# repo sync 所同步的远端服务器分支
查看 repo help sync 命令的帮助说明，该命令的格式如下：
> Usage: repo sync [\<project\>...]

可以看到，它没有提供参数来指定要同步的远端服务器分支。实际上，默认会同步 repo init 时 -b 选项指定的分支。当本地分支名和 repo init 的分支名不同时，执行 repo sync 会改变本地分支指向。举例说明如下。

使用 git branch 命令，打印出当前分支名是 branch_m：
```git
$ git branch
  other_branch_xxx
* branch_m
```
在当前代码目录下执行 repo sync 命令：
```git
$ repo sync .
Fetching project platform/packages/apps/Settings
packages/apps/Settings/: leaving branch_m; does not track upstream
```
再次执行 git branch 命令，会看到当前处于没有命名的分支下：
```git
$ git branch
* (detached from f15a7be)
  other_branch_xxx
  branch_m
```
基于这个现象，建议在本地所有分支名跟服务器分支名都相同时，才用 repo sync 来同步代码。避免同步之后，分支指向发生变化，此时修改的代码不是位于原来分支下面，后续如果要提交修改到原来的分支，会提示需要merge，容易造成代码冲突。

# repo sync 指定同步某个仓库project
执行 `repo sync` 命令默认同步所有git仓库，可以提供一个或多个 project 参数来指定需要同步的git仓库路径。关键是，如何知道某个git仓库的 project 参数值是多少。

经过实际测试发现，这里提供的 project 参数值是基于shell当前的工作目录寻址到目标git仓库的目录路径，而不是git仓库在 repo 中保存的完整路径。

例如 `repo status` 命令打印了下面的git仓库信息。可以看到，这个git仓库在 repo 中保存的完整路径是 "android/packages/apps/Settings/"。
```
project android/packages/apps/Settings/ branch branch_m
 -m     res/values-zh-rCN/strings.xml
```
然后在 "android/" 目录下执行 `repo sync -d android/packages/apps/Settings/` 命令会报错：
```bash
$ repo sync -d android/packages/apps/Settings/
error: project android/packages/apps/Settings/ not found
```
即，执行命令时，shell的工作目录在 *android* 目录下，然后传入git仓库在 repo 中的完整目录路径 "android/packages/apps/Settings/"，repo sync 命令执行会报错。

而传入 "packages/apps/Settings/" 这个路径就可以正常执行：
```bash
$ repo sync -d packages/apps/Settings/
Fetching project platform/packages/apps/Settings/
```
如果本身已经在 "android/packages/apps/Settings/" 目录下面。直接执行 `repo sync .` 命令即可：
```bash
$ repo sync .
Fetching project platform/packages/apps/Settings/
```
总的来说，repo sync 后面跟着的 project 参数值应该是基于当前shell工作目录能够寻址到该project的目录路径，类似于cd命令的寻址方式。

查看 repo help sync 对 project 参数的说明如下，符合上面的验证结果：
> 'repo sync' will synchronize all projects listed at the command line. Projects can be specified either by name, or by a relative or absolute path to the project's local directory. If no projects are specified, 'repo sync' will synchronize all projects listed in the manifest.

基于这个说明，可以提供 project name、project本地目录的相对路径或绝对路径来指定要同步的project。

这里说的 project name 可以在 repo 生成的 .repo/ 目录下查看。例如，查看这个目录下的 manifest.xml 文件，有如下信息:
```xml
<project groups="p-fs-release,pdk-fs" name="platform/packages/apps/Settings"
    path="android/packages/apps/Settings"  />
```
可以看到 Settings project 的 name 是 *platform/packages/apps/Settings*。那么不管在哪个目录下，执行 `repo sync platform/packages/apps/Settings` 命令都会同步 Settings 目录的代码。

# repo sync 的 -d 选项说明
使用 repo sync 来同步代码时，如果本地修改了代码还没有commit，会提示无法sync：
> error: android/frameworks/base/: contains uncommitted changes

如果想要强制同步远端服务器代码，可以加上 -d 选项，`repo sync -d` 会将 HEAD 强制指向 repo manifest的库，而忽略本地的改动。查看 repo help sync 对 -d 选项的说明如下：
> **-d, --detach**          detach projects back to manifest revision

注意：加上 -d 选项只表示忽略本地改动，可以强制同步服务器代码，但是本地改动的文件还是保持改动不变，不会强制覆盖掉本地修改。而且执行之后，本地的分支指向会发生变化，不再指向原来的分支。具体举例如下。

下面是执行 repo sync -d 之前的分支信息：
```git
$ git branch
* curent_branch_xxx
```
下面是执行 repo sync -d 之后的分支信息：
```git
$ git branch
* (detached from 715faf5)
  curent_branch_xxx
```
用 git status 查看，可以看到本地还是有修改过且还没有commit的文件，同步服务器代码后，并不会强制覆盖本地文件为服务器的代码：
```git
$ git status
HEAD detached at 715faf5
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
        modified:   vendor/chioverride/default/g_pipelines.h
        modified:   vendor/topology/g_usecase.xml
```
即，如果想要丢弃本地修改、让本地代码跟git仓库代码一致，`repo sync -d` 命令达不到这个效果。

另外，repo sync 有一个 --force-sync 选项：
> **--force-sync**  
overwrite an existing git directory if it needs to point to a different object directory. WARNING: this may cause loss of data

从说明来看，像是可以强制同步，可能会丢失本地改动。但是实际测试发现，这个选项并不会强制覆盖本地的改动。如果本地文件被改动，加上这个选项还是会sync报错：
```bash
$ repo sync --force-sync .
Fetching project tools/
error: tools/: contains uncommitted changes
```
同时提供 -d 和 --force-sync 两个选项，还是不会强制覆盖本地修改。  
目前没有找到 repo sync 命令可以强制覆盖本地修改的选项.

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
> repo forall -c "branch_name=$(git rev-parse --abbrev-ref HEAD) && \
  git rev-parse --abbrev-ref ${branch_name}@{upstream} && git reset --hard && git pull --stat --no-tags" |& cat

对这个命令的各个部分说明如下。
- `repo forall -c`：对所有git仓库都执行后面的跟着的命令，这里使用双引号把多个git命令都括起来，作为一个整体传给repo，可以做一些复杂的操作。
- `branch_name=$(git rev-parse --abbrev-ref HEAD)`：这是bash shell的语法, `$(cmd)` 会执行 cmd 命令，得到它的输出，这里把输出结果赋值给 *branch_name* 变量。`git rev-parse --abbrev-ref HEAD` 命令输出且只输出本地分支名。
- `&&`：bash shell的与操作，前一个命令执行结果为true，才会执行下一个命令。
- `git rev-parse --abbrev-ref ${branch_name}@{upstream}`：`${branch_name}` 表示获取 *branch_name* 变量的值。*branch_name* 变量在前面被赋值为当前本地分支名。整个git命令获取本地分支在远端服务区分支名。如果获取不到，执行结果是false。
- `git reset --hard`：基于 `&&` 操作符的特性，如果上一个git命令获取不到本地分支在远端服务器的分支名，就不会往下执行。所以执行到这里时，说明本地分支名跟远端服务器分支名相同，我们假设本地开发分支没有关连到远端服务器分支，那么当前分支就不是开发分支，用 git reset 丢弃本地修改。
- `git pull --stat --no-tags`：同步服务器代码，--stat 表示要打印发生改动的文件信息，--no-tags 表示不获取远端服务器仓库的tag信息。如果需要pull远端服务器的tag，可以不加 --no-tags 选项。
- `|& cat`：把整个repo命令的标准输出、错误输出都重定向到 cat 命令。如果不重定向，repo 会用 less 命令来显示输出内容，需要手动按q退出才会继续输出，重定向到 cat 后，直接打印全部输出内容，不需要再手动进行其他操作。
