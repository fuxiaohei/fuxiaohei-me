```toml

title = "Windows 下 gcc + golang 编译 git2go"
slug = "windows-git2go"
date = "2014-02-18 22:04:12"
author = "fuxiaohei"
tags = ["Go","git"]

```

最近研究用go语言操作git，除了直接走命令行用`os/exec`包，还可以使用`libgit2`的go绑定`git2go`操作。
但是`libgit2`是c语言库，go使用`cgo`连接c程序，需要`cgo`的支持。总之过程复杂，摔了一路。<!--more-->

#### 安装 gcc 和 pkg-config

首先是安装`gcc`和`pkg-config`（cgo依赖）。gcc编译器推荐用[TDM-GCC](http://tdm-gcc.tdragon.net/)来直接安装，方便快捷，注意不要用绿色版用安装版。`pkg-config`可以再gnome的[官方库](http://ftp.gnome.org/pub/gnome/binaries/win32/dependencies/)中找到。`pkg-config`安装需要同时下载：

```
http://ftp.gnome.org/pub/gnome/binaries/win32/dependencies/pkg-config_0.26-1_win32.zip
http://ftp.gnome.org/pub/gnome/binaries/win32/dependencies/gettext-runtime_0.18.1.1-2_win32.zip
http://ftp.gnome.org/pub/gnome/binaries/win32/glib/2.28/glib_2.28.8-1_win32.zip
```

把几个zip包中bin目录的所有exe和dll拷贝到gcc的bin目录。

#### gcc 编译 libgit2

[libgit2](http://libgit2.github.com/) 是Git核心开发包的纯c实现，可以很容易移植和嵌入到别的应用中。官网也提供的它和各种语言的绑定，比如Go语言的[git2go](https://github.com/libgit2/git2go)。

在Mac上golang编译git2go很容易：

```
brew install libgit2
go get github.com/libgit2/git2go
```

不过因为git2go和对应的libgit2进度不同，Windows编译的时候问题不断。

直接git clone最新的libgit2代码（错误的），使用cmake编译。具体方法在官方wiki [Building libgit2 on Windows](https://github.com/libgit2/libgit2/wiki/Building-libgit2-on-Windows) 已经写清楚，照着来就行。唯一注意，把编译参数中的 `BUILD_CLAR` 关闭，就可以不依赖python。还有，使用文档最后的参数：

```
cmake . -DCMAKE_INSTALL_PREFIX=C:\libgit2
```

编译到目录名没有空格的目录，否则git2go的wrapper.c会解析地址错误。建议用`cmake-gui`查看并设置编译参数。按照wiki编译：

```
cmake --build .. --target install
```

编译完成，将**C:/libgit2/lib/pkgconfig**添加到系统变量`PKG_CONFIG_PATH`，让`pkg-config`可以找到libgit2.pc文件。

之后就可以 go get啦。

###### 但是

后来在Github项目的[Pull#53](https://github.com/libgit2/git2go/pull/53)发现，最新的libgit2更新了API破坏了git2go。最终求助mac的同学，查到brew提供的编译好的是0.20版本，就去下载 [release 0.20.0](https://github.com/libgit2/libgit2/releases/tag/v0.20.0)。

#### golang 编译 git2go

git2go的编译唯一需要注意就是，将**C:/libgit2/lib/pkgconfig**添加到系统变量`PKG_CONFIG_PATH`，让cgo可以访问到.pc文件，读取库相关信息。剩下就是 go get 或者 go install。
