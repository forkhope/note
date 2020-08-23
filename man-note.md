# 介绍在 Linux 中使用 man 手册的一些使用笔记

# 安装完整的 man 手册
在 Debian 系统、Ubuntu 系统中，可以执行下面命令来安装完整的 man 手册。

这会安装系统函数、库函数等的说明文档。
```bash
sudo apt-get install manpages 
sudo apt-get install manpages-de 
sudo apt-get install manpages-de-dev 
sudo apt-get install manpages-dev
sudo apt-get install manpages-posix manpages-posix-dev
```

# man 手册去掉中文版本
当安装了中文版本的 Linux 系统时，查看 man 手册的帮助信息，默认是中文版本。

如果想要查看英文版本的 man 手册，可以执行下面命令从系统中去掉中文版本的 man 书册。
```
sudo apt-get purge manpages-zh
sudo mandb -c
```

当执行 `sudo apt-get purge manpages-zh` 命令后，再次查看 man 手册，就会看到英文版本的帮助信息。

但是在退出 man 手册后，会看到提示 “无法解析 ......” 的错误。

例如出现下面的提示：
```
man: 无法解析 /usr/share/man/zh_CN/man1/man.1.gz: 没有那个文件或目录
man: 无法解析 /usr/share/man/zh_CN/man7/man.7.gz: 没有那个文件或目录
```
即，提示无法解析中文版本的 man 手册文件。

解决这个现象的方法就是执行 `sudo mandb -c` 命令。

该命令会更新 man 手册的索引数据库，以便去掉中文版本的 man 手册文件后，不再解析它们。

重建 man 手册的数据库需要一些时间。稍微等待即可。

# 将 man 手册导出成 txt、pdf、html 格式

## 导出成 txt 格式
把 man 手册的帮助信息导出成 txt 格式，最简单的方法是重定向。

例如，`man grep > grep.txt` 命令导出 `grep` 命令的帮助信息到 *grep.txt* 文件。

但是这个写法导出来的 txt 格式文件可能会有一些乱码问题。

如果遇到乱码的情况，可以使用下面的命令来导出 txt 格式：
```
man grep | col -b > grep_man.txt
```
这里使用 `col -b` 命令去掉一些控制字符。

## 导出成 pdf 格式
可以使用下面命令把 man 手册的帮助信息导出成 pdf 格式：
```
man -t grep | ps2pdf - grep_man.pdf
```
这里查看 `grep` 命令的帮助信息，可以换成其他感兴趣的命令。

Debian 系统、Ubuntu 系统上，默认应该就会安装 `ps2pdf` 命令。

如果还没有安装这个命令，自行安装即可。

查看 man ps2pdf 的说明信息，`ps2pdf` 还有其他命令可以生成新版本的 pdf 格式。

例如下面贴出的 `ps2pdf14` 命令：
> ps2pdf14 - Convert PostScript to PDF 1.4 (Acrobat 5-and-later compatible) using ghostscript

如果对 pdf 版本有要求，可以查看 man ps2pdf 的说明，使用最新版本的命令。

## 导出成 html 格式
如果要把 man 手册的帮助信息导出成 html 格式，可以先用下面命令在浏览器中查看 man 手册的帮助信息：
```
man -Hfirefox grep
```

这里使用火狐（firefox）浏览器查看 `grep` 命令的帮助信息，可以换成其他感兴趣的命令。

也可以换成其他浏览器，例如 google-chrome 等。

在浏览器打开 man 的帮助信息后，使用浏览器自带的 “网页另存为” 功能导出这个 html 页面即可。

执行上面命令时，可能会遇到类似下面的报错：
```bash
$ man -Hfirefox grep
man: command exited with status 3: /usr/bin/zsoelim | /usr/lib/man-db/manconv -f UTF-8:ISO-8859-1 -t UTF-8//IGNORE | preconv -e UTF-8 | tbl | groff -mandoc -Thtml
```

此时，可以执行 `sudo apt-get install groff` 命令安装 groff 软件包。

如果系统已经安装了 groff 命令，还执行报错，可以先卸载，再重新安装最新版本的 groff 命令。
