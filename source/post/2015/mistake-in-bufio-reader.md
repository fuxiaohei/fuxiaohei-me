```toml
title = "我煞笔的被 bufio.Reader 小坑"
slug = "mistake-in-bufio-reader"
date = "2015-12-09 20:58:59"
author = "fuxiaohei"
tags = ["Go"]
```

最近在用 Go 做一个小型的 `gateway` 服务。PHP 请求 Go 的 tcp server，然后 Go 根据命令参数开启多个 goroutine 去调度 php-fpm 执行不同的脚本并组合结果返回。 想来只是利用 goroutine 的便利并发执行逻辑，如此简单直接。

不过在测试的时候 PHP 发送 socket 的 json 数据发生了明确的截断，后来发现是我煞笔的被 `bufio.Reader` 坑了，真是无言以对。

### 问题重现

因为是长连接，PHP 每次发一段 json ，都会加上换行符`\n`分割。我就很理所当然的用起了`*bufio.Reader.ReadLine()`。就像下面的代码：

```go
func handleConn(conn net.Conn) {
    reader := bufio.NewReader(conn)
    for {
        // 读取一行数据，交给后台处理
        line,_,err := reader.ReadLine()
        if len(line) > 0{
            fmt.Printf("ReadData|%d \n",len(line))
            executeBytes(line)
        }
        if err != nil{
            break
        }
    }
    conn.Close()
}
```

可是当 PHPer 测试发送大json数据的时候，发现了明确的截断：

    ReadData|4096|{"ename.com":........,"yunduo.com":{"check":[1
    // 又一次
    ReadData|4096|{"ename.com":........,"yunduo.com":{"getWhois"

想让 json 还没读完就被截断。我记得默认的 bufio.Reader 的大小是 4096，所以 bufio.Reader 的行为是读满了buffer就return出来啊。。我勒个去。我总不能写个很大的size吧。

```go
    reader := bufio.NewReaderSize(conn,409600)
```

PHP 发送的测试数据最大可能得到100k左右，平均只有<10k。我服务端每次都开一个100k+的`bufio.Reader`太浪费啦。最后变成，开小了截断，开大了浪费，我煞笔了。

<!--more-->

### 解决方式

既然`bufio.Reader.ReadLine()` 玩不下去了，我就只能回到最原生的用法，例如改造后的代码：

```go
func handleConn(conn net.Conn) {
	buf := make([]byte, 4096)
	var jsonBuf bytes.Buffer
	for {
		n, err := conn.Read(buf)

		if n > 0 {
			if buf[n-1] == 10 { // 10就是\n的ASCII
				jsonBuf.Write(buf[:n-1]) // 去掉最后的换行符
				executeBytes(jsonBuf.Bytes())
				jsonBuf.Reset() // 重置后用于下一次解析
			} else {
				jsonBuf.Write(buf[:n])
			}
		}

		if err != nil {
			break
		}
	}
	conn.Close()
}
```

这样就是每次读出4096的字节，读到\n就截断，和之前放入buffer的数据取出来一起交给后续运算。这样不用在意传来的数据大小，只要\n分隔符正确就没有问题。

经过测试也没有发现什么业务问题，pprof也没有看出明显的性能瓶颈。那就这么用着，我松一口气交差。

### 又煞笔啦

`bufio.Reader.ReadLine()`应该没有那么傻吧。如果buffer满了就return，我怎么知道只是满了还是读到\n的返回，万一内容刚好buffer长度呢！下班回家翻源码，我又煞笔了：

```go
// ReadLine is a low-level line-reading primitive. Most callers should use
// ReadBytes('\n') or ReadString('\n') instead or use a Scanner.
//
// ReadLine tries to return a single line, not including the end-of-line bytes.
// If the line was too long for the buffer then isPrefix is set and the
// beginning of the line is returned. The rest of the line will be returned
// from future calls. isPrefix will be false when returning the last fragment
// of the line. The returned buffer is only valid until the next call to
// ReadLine. ReadLine either returns a non-nil line or it returns an error,
// never both.
//
// ......
func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error) {
	line, err = b.ReadSlice('\n')
	if err == ErrBufferFull {
		// Handle the case where "\r\n" straddles the buffer.
		if len(line) > 0 && line[len(line)-1] == '\r' {
			// Put the '\r' back on buf and drop it from line.
			// Let the next call to ReadLine check for "\r\n".
			if b.r == 0 {
				// should be unreachable
				panic("bufio: tried to rewind past start of buffer")
			}
			b.r--
			line = line[:len(line)-1]
		}
		return line, true, nil
	}
	......
}

```

我擦。如果读出的满了buffer，当不是\n结尾，`isPrefix`是true。我从来没注意过中间这个返回值的意思。真是被自己坑了。其实代码可以这么写：

```go
func handleConn(conn net.Conn) {
    reader := bufio.NewReader(conn)
    var jsonBuf bytes.Buffer
    for {
        // 读取一行数据，交给后台处理
        line,isPrefix,err := reader.ReadLine()
        if len(line) > 0{
            jsonBuf.Write(line)
            if !isPrefix{
                executeBytes(jsonBuf.Bytes())
                jsonBuf.Reset()
            }
        }
        if err != nil{
            break
        }
    }
    conn.Close()
}
```

### RTFD

结论：**Read The Fucking Documentation**

