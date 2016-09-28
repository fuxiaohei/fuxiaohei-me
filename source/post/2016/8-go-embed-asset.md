```toml
title = "Go 内嵌静态资源"
slug = "go-binary-embed-asset"
date = "2016-10-01 19:00:34"
update_date = "2016-10-01 19:00:35"
author = "fuxiaohei"
tags = ["Go","Golang"]
```

使用 Go 开发应用的时候，有时会遇到需要读取静态资源的情况。比如开发 Web 应用，程序需要加载模板文件生成输出的 HTML。在程序部署的时候，除了发布应用可执行文件外，还需要发布依赖的静态资源文件。这给发布过程添加了一些麻烦。既然发布单独一个可执行文件是非常简单的操作，就有人会想办法把静态资源文件打包进 Go 的程序文件中。下面就来看一些解决方案：

### go-bindata

[go-bindata](https://github.com/jteeuwen/go-bindata) 是目前我的程序 `pugo` 在用的嵌入静态资源的工具。它可以把静态文件嵌入到一个 go 文件中，并提供一些操作方法。

安装 `go-bindata`：

```
go get -u github.com/jteeuwen/go-bindata/...
```

***注意 go get 地址最后的三个点 `...`***。这样会分析所有子目录并下载依赖编译子目录内容。`go-bindata` 的命令工具在子目录中。（还要记得把 `$GOPATH/bin` 加入系统 `PATH`）。


使用命令工具 `go-bindata` （ pugo 的例子）：

```
go-bindata -o=app/asset/asset.go -pkg=asset source/... theme/... doc/source/... doc/theme/... 
```

`-o` 输出文件到 `app/asset/asset.go`，包名 `-pkg=asset`，然后是需要打包的目录，三个点包括所有子目录。这样就可以把所有相关文件打包到 `asset.go` 且开头是 `package asset` 保持和目录一致。

pugo 里释放静态文件的代码：

```go
dirs := []string{"source", "theme", "doc"} // 设置需要释放的目录
isSuccess := true

var (
    err       error
    isExtract = true
)
for _, dir := range dirs {
    // 解压dir目录到当前目录
    if err = asset.RestoreAssets("./", dir); err != nil {
        isSuccess = false
        break
    }
}
if !isSuccess {
    for _, dir := range dirs {
        os.RemoveAll(filepath.Join("./", dir))
    }
}
```

`asset.go` 内的静态内容还是根据实际的目录位置索引。所以我们可以直接通过目录或者文件地址去操作。

##### 开发模式

