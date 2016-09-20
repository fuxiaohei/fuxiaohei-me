```toml
title = "Go 开发 HTTP"
slug = "go-and-http-server"
date = "2016-09-20 20:00:34"
update_date = "2016-09-20 20:00:35"
author = "fuxiaohei"
tags = ["Go","Golang","入门","HTTP"]
```

Go 是一门新语言。很多人都是用 Go 来开发 Web 服务。Web 开发很多同学急于求成，直接使用 [beego](#), [echo](#) 或 [iris](#) 等知名框架。对标准库 `net/http` 的了解甚少。这里我就主要聊一下标准库 `net/http` 开发 Web 服务时的使用细节。

### 创建 HTTP 服务

在 Go 中，创建 HTTP 服务很简单：

```go
package main

// in main.go

import (
    "fmt"
    "net/http"
)

func main(){
    if err := http.ListenAndServe(":12345",nil); err != nil{
        fmt.Println("start http server fail:",err)
    }
}

```

这样就会启动一个 HTTP 服务在端口 **12345**。浏览器输入 `http://localhost:12345/` 就可以访问。当然从代码看出，没有给这个 HTTP 服务添加实际的处理逻辑，所有的访问都是默认的 `404 Not Found`。

<!--more-->


##### 添加 http.Handler

添加 HTTP 的处理逻辑的方法简单直接：

```go

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello, Go HTTP Server"))
	})
    http.HandleFunc("/abc", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello, Go HTTP abc"))
	})
	if err := http.ListenAndServe(":123456", nil); err != nil {
		fmt.Println("start http server fail:", err)
	}
}

```

访问 `http://localhost:12345/` 就可以看到页面输出 `Hello, Go HTTP Server` 的内容。访问 `http://localhost:12345/abc` 就可以看到页面输出 `Hello, Go HTTP abc` 的内容。但是 Go 默认的路由匹配机制很弱。上面的代码除了 `/abc`，其他的请求都匹配到 `/` ，不足以使用，肯定需要自己写路由的过程。一个简单的方式就是写一个自己的 `http.Handler`。

```go

type MyHandler struct{} // 实现 http.Handler 接口的 ServeHTTP 方法

func (mh MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path == "/abc" {
		w.Write([]byte("abc"))
		return
	}
	if r.URL.Path == "/xyz" {
		w.Write([]byte("xyz"))
		return
	}
	w.Write([]byte("index"))
	// 这里你可以写自己的路由匹配规则
}

func main() {
	http.Handle("/", MyHandler{})
	if err := http.ListenAndServe(":12345", nil); err != nil {
		fmt.Println("start http server fail:", err)
	}
}
```

这样可以在自己的 `MyHandler` 写复杂的路由规则和处理逻辑。`http.ListenAndServe` 的第二个参数写的会更优雅：

```go
func main() {
	if err := http.ListenAndServe(":12345", MyHandler{}); err != nil {
		fmt.Println("start http server fail:", err)
	}
}
```

### Request

HTTP 过程的操作主要是针对客户端发来的请求数据在 `*http.Request`，和返回给客户端的 `http.ResponseWriter` 两部分。

请求数据 `*http.Request` 有两个部分：基本数据和传递参数。基本数据比如请求的方法、协议、URL、头信息等可以直接简单获取：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	fmt.Println("Method:", r.Method)
	fmt.Println("URL:", r.URL, "URL.Path", r.URL.Path) // 这里是 *net/url.URL 结构，对应的内容可以查API
	fmt.Println("RemoteAddress", r.RemoteAddr)
	fmt.Println("UserAgent", r.UserAgent())
	fmt.Println("Header.Accept", r.Header.Get("Accept"))
    fmt.Println("Cookies",r.Cookies())
    // 还有很多肯定会有的基本数据，可以查阅 API 找寻一下
}

http.HandleFunc("/", HttpHandle)
if err := http.ListenAndServe(":12345", nil); err != nil {
	fmt.Println("start http server fail:", err)
}
```

##### 表单数据

请求传递的参数，也就是 **表单数据**，保存在 `*http.Request.Form` 和 `*http.Request.PostForm` (POST 或 PUT 的数据)，类似 PHP 的 $_REQUEST 和 $_POST/$_PUT 两个部分。

例如 `GET /?abc=xyz`，获取这个数据并打印到返回内容：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	postValue := r.FormValue("abc")
	w.Write([]byte("GET: abc=" + postValue))
}
```

访问 `http://localhost:12345/?abc=123` 就可以看到页面内容 `GET: abc=123`。POST 的表单数据也是类似：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	v1 := r.FormValue("abc")
	v2 := r.PostFormValue("abc")
	v3 := r.Form.Get("abc")
	v4 := r.PostForm.Get("abc")
	fmt.Println(v1 == v2, v1 == v3, v1 == v4)
	w.Write([]byte("POST: abc=" + v1))
}
```

注意，这四个值 `v1,v2,v3,v4` 是相同的值。如果同一个表单域传递了多个值，需要直接操作 `r.Form` 或 `r.PostForm`，比如 `GET /?abc=123&abc=abc&abc=xyz`：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
    // 这里一定要记得 ParseForm，否则 r.Form 是空的
    // 调用 r.FormValue() 的时候会自动执行 r.ParseForm()
	r.ParseForm()
	values := r.Form["abc"]
	w.Write([]byte("GET abc=" + strings.Join(values, ","))) // 这里记得 import "strings"
}
```

访问 `http://localhost:12345/?abc=123&abc=abc&abc=xyz` 可以看到内容 `GET abc=123,abc,xyz`。

表单数据存储在 `r.Form`，是 `map[string][]string`，即支持一个表单域多个值的情况。**`r.FormValue()` 只获取第一个值**。

##### Body 消息体

无论表单数据，还是上传的二进制数据，都是保存在 HTTP 的 Body 中的。操作 `*http.Request.Body` 可以获取到内容。但是注意 `*http.Request.Body` 是 `io.ReadCloser`，只能一次性读取完整，**第二次就是空的**。

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	body, err := ioutil.ReadAll(r.Body)
	if err != nil {
		fmt.Println("read body fail:", err)
		w.WriteHeader(500)
		return
	}
    DoSomething(body) // 尽量使用已经读出的 Body 内容，不要再去读取 r.Body 
	w.Write([]byte("body:"))
	w.Write(body)
}
```

使用 `curl` 命令行发送 POST 数据到服务器，`curl -X POST --data "abcdefg" http://localhost:12345/`,可以看到返回内容 `body:abcdefg`。

根据 HTTP 协议，如果请求的 `Content-Type: application/x-www-form-urlencoded`，Body 中的数据就是类似 `abc=123&abc=abc&abc=xyz` 格式的数据，也就是常规的 **表单数据**。这些使用 `r.ParseForm()` 然后操作 `r.Form` 处理数据。如果是直接的文本数据，比如 `abcdefg` 甚至 **JSON** 数据，你才需要直接操作 Body 的。比如接收 JSON 数据：

```go

type User struct {
	Id   int    `json:"id"`
	Name string `json:"name"`
}

func HttpHandle(w http.ResponseWriter, r *http.Request) {
	// Body 里的内容是 JSON 数据：
	// {"id":123,"name":"xyz"}
	body, err := ioutil.ReadAll(r.Body)
	if err != nil {
		fmt.Println("read body fail:", err)
		w.WriteHeader(500)
		return
	}
	var u User
	if err = json.Unmarshal(body, &u); err != nil {
		fmt.Println("json Unmarshal fail:", err)
		w.WriteHeader(500)
		return
	}
	w.Write([]byte("user.id:" + strconv.Itoa(u.Id) + " "))
	w.Write([]byte("user.name:" + u.Name))
    // 返回内容是: user.id:123 user.name:xyz
}
```

如果需要对 Body 的数据做直接处理，JSON 数据例子是通用的模式。


##### 上传文件

上传的文件经过 Go 的解析保存在 `*http.Request.MultipartForm` 中,通过 `r.FormFile()` 去获取收到的文件信息和数据流，并处理：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	// 这里一定要记得 r.ParseMultipartForm(), 否则 r.MultipartForm 是空的
	// 调用 r.FormFile() 的时候会自动执行 r.ParseMultipartForm()
	r.ParseMultipartForm(32 << 20) 
    // 写明缓冲的大小。如果超过缓冲，文件内容会被放在临时目录中，而不是内存。过大可能较多占用内存，过小可能增加硬盘 I/O
	// FormFile() 时调用 ParseMultipartForm() 使用的大小是 32 << 20，32MB
	file, fileHeader, err := r.FormFile("file") // file 是上传表单域的名字
	if err != nil {
		fmt.Println("get upload file fail:", err)
		w.WriteHeader(500)
		return
	}
	defer file.Close() // 此时上传内容的 IO 已经打开，需要手动关闭！！

	// fileHeader 有一些文件的基本信息
	fmt.Println(fileHeader.Header.Get("Content-Type"))

	// 打开目标地址，把上传的内容存进去
	f, err := os.OpenFile("saveto.txt", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0666)
	if err != nil {
		fmt.Println("save upload file fail:", err)
		w.WriteHeader(500)
		return
	}

	defer f.Close()
	if _, err = io.Copy(f, file); err != nil {
		fmt.Println("save upload file fail:", err)
		w.WriteHeader(500)
		return
	}
	w.Write([]byte("upload file:" + fileHeader.Filename + " - saveto : saveto.txt"))
}
```

上传文件信息中，**文件大小** 信息是没有的。而文件大小是上传限制中必要的条件，所以需要一些方法来获取文件大小：

```go
// 使用接口检查是否有 Size() 方法
type fileSizer interface {
	Size() int64
}

// 从 multipart.File 获取文件大小
func getUploadFileSize(f multipart.File) (int64, error) {
    // 从内存读取出来
	// if return *http.sectionReader, it is alias to *io.SectionReader
	if s, ok := f.(fileSizer); ok {
		return s.Size(), nil
	}
    // 从临时文件读取出来
	// or *os.File
	if fp, ok := f.(*os.File); ok {
		fi, err := fp.Stat()
		if err != nil {
			return 0, err
		}
		return fi.Size(), nil
	}
	return 0, nil
}
```

**`r.FormFile()` 只返回第一个上传的文件**，如果同一个表单域上传多个文件，只能直接操作 `r.MultipartForm`：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	r.ParseMultipartForm(32 << 20)
	var (
		file multipart.File
		err  error
	)
	for _, fileHeader := range r.MultipartForm.File["file"] {
		if file, err = fileHeader.Open(); err != nil {
			fmt.Println("open upload file fail:", fileHeader.Filename, err)
			continue
		}
		SaveFile(file) // 仿照上面单个文件的操作，处理 file
		file.Close() // 操作结束一定要 Close，for 循环里不要用 defer file.Close()
		file = nil
		w.Write([]byte("save:" + fileHeader.Filename + " "))
	}
}
```

### ResponseWriter

`http.ResponseWriter` 是一个接口，你可以根据接口，添加一些自己需要的行为：

```go
type ResponseWriter interface {
	Header() Header // 添加返回头信息
	Write([]byte) (int, error) // 添加返回的内容
	WriteHeader(int) // 设置返回的状态码
}
```

`w.WriteHeader()` 是一次性的，不能重复设置状态码，否则会有提示信息：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(200) // 设置成功
	w.WriteHeader(404) // 提示：http: multiple response.WriteHeader calls 
	w.WriteHeader(503) // 提示：http: multiple response.WriteHeader calls 
}
```

而且需要在 `w.Write()` 之前设置 `w.WriteHeader()`，否则是 200。（要先发送状态码，再发送内容）

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello World"))
	w.WriteHeader(404) // 提示：http: multiple response.WriteHeader calls，因为 w.Write() 已发布 HTTP 200
}
```

`http.ResponseWriter` 接口过于简单，实际使用会自己实现 `ResponseWriter` 来使用，比如获取返回的内容：

```go
type MyResponseWriter struct {
	http.ResponseWriter
	bodyBytes *bytes.Buffer
}

// 覆写 http.ResponseWriter 的方法
func (mrw MyResponseWriter) Write(body []byte) (int, error) {
	mrw.bodyBytes.Write(body) // 记录下返回的内容
	return mrw.ResponseWriter.Write(body)
}

// Body 获取返回的内容，这个是自己添加的方法
func (mrw MyResponseWriter) Body() []byte {
	return mrw.bodyBytes.Bytes()
}

func HttpHandle(w http.ResponseWriter, r *http.Request) {
	m := MyResponseWriter{
		ResponseWriter: w,
		bodyBytes:      bytes.NewBuffer(nil),
	}
	m.Write([]byte("Hello World"))
	m.Write([]byte("abcxyz"))
	fmt.Println("body:", string(m.Body()))
}
```

##### 输出其他内容

`net/http` 提供一些便利的方法可以输出其他的内容，比如 cookie:

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	c := &http.Cookie{
		Name:     "abc",
		Value:    "xyz",
		Expires:  time.Now().Add(1000 * time.Second),
		MaxAge:   1000,
		HttpOnly: true,
	}
	http.SetCookie(w, c)
}
```

比如服务端返回下载文件：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	http.ServeFile(w, r, "download.txt")
}
```

或者是生成的数据流，比如验证码，当作文件返回：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	captchaImageBytes := createCaptcha() // 假设生成验证码的函数，返回 []byte
	buf := bytes.NewReader(captchaImageBytes)
	http.ServeContent(w, r, "captcha.png", time.Now(), buf)
}
```

还有一些状态码的直接操作：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	http.Redirect(w, r, "/abc", 302)
}

func HttpHandle2(w http.ResponseWriter, r *http.Request) {
	http.NotFound(w, r)
}
```

### Context

Go 1.7 添加了 `context` 包，用于传递数据和做超时、取消等处理。`*http.Request` 添加了 `r.Context()` 和 `r.WithContext()` 来操作请求过程需要的 `context.Context` 对象。

##### 传递数据

`context` 可以在 `http.HandleFunc` 之间传递数据：

```go
func handle1(w http.ResponseWriter, r *http.Request) {
	ctx := context.WithValue(r.Context(), "abc", "xyz123") // 写入 string 到 context
	handle2(w, r.WithContext(ctx))                         // 传递给下一个 handleFunc
}

func handle2(w http.ResponseWriter, r *http.Request) {
	str, ok := r.Context().Value("abc").(string) // 取出的 interface 需要推断到 string
	if !ok {
		str = "not string"
	}
	w.Write([]byte("context.abc = " + str))
}

func main() {
	http.HandleFunc("/", handle1)
	if err := http.ListenAndServe(":12345", nil); err != nil {
		fmt.Println("start http server fail:", err)
	}
}
```

##### 处理超时的请求

利用 `context.WithTimeout` 可以创建会超时结束的 context，用来处理业务超时的情况：

```go
func HttpHandle(w http.ResponseWriter, r *http.Request) {
	ctx, cancelFn := context.WithTimeout(r.Context(), 1*time.Second)

	// cancelFn 关掉 WithTimeout 里的计时器
	// 如果 ctx 超时，计时器会自动关闭，但是如果没有超时就执行到 <-resCh,就需要手动关掉
	defer cancelFn()

	// 把业务放到 goroutine 执行， resCh 获取结果
	resCh := make(chan string, 1)
	go func() {
        // 故意写业务超时
		time.Sleep(5 * time.Second)
		resCh <- r.FormValue("abc")
	}()

	// 看 ctx 超时还是 resCh 的结果先到达
	select {
	case <-ctx.Done():
		w.WriteHeader(http.StatusGatewayTimeout)
		w.Write([]byte("http handle is timeout:" + ctx.Err().Error()))
	case r := <-resCh:
		w.Write([]byte("get: abc = " + r))
	}
}

func main() {
	http.HandleFunc("/", HttpHandle)
	if err := http.ListenAndServe(":12345", nil); err != nil {
		fmt.Println("start http server fail:", err)
	}
}

```

### Hijack

// todo

### Template

// todo

### Server 的细节使用

// todo