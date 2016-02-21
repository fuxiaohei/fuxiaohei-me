```toml
title = "Beego源码分析"
slug = "beego-source-study"
date = "2014-04-30 13:02:55"
author = "fuxiaohei"
tags = ["Go","beego"]
```

`beego` 是 [@astaxie](https://github.com/astaxie) 开发的重量级Go语言Web框架。它有标准的MVC模式，完善的功能模块，和优异的调试和开发模式等特点。并且`beego`在国内企业用户较多，社区发达和Q群，文档齐全，特别是 @astaxie 本人对bug和issue等回复和代码修复很快，非常敬业。`beego`框架本身模块众多，无法简单描述所有的功能。我简单阅读了源码，记录一下`beego`执行过程。官方文档已经图示了`beego`执行过程[图](http://beego.me/docs/mvc/)，而我会比较详细的解释`beego`的源码实现。
<!--more-->
**注意**，本文基于beego 1.1.4 (2014.04.15) 源码分析，且不是`beego`的使用教程。使用细节的问题在这里不会说明。

## 1. 启动应用

[beego官方首页](http://beego.me/)提供的示例非常简单：

```go
package main

import "github.com/astaxie/beego"

func main() {
    beego.Run()
}
```

那么，从`Run()`方法开始，在[beego.go#179](https://github.com/astaxie/beego/blob/master/beego.go#L179)：

```go
func Run() {
	initBeforeHttpRun()

	if EnableAdmin {
		go beeAdminApp.Run()
	}

	BeeApp.Run()
}
```

额呵呵呵，还在更里面，先看`initBeforeHttpRun()`，在[beego.go#L189](https://github.com/astaxie/beego/blob/master/beego.go#L189):

```go
func initBeforeHttpRun() {
	// if AppConfigPath not In the conf/app.conf reParse config
	if AppConfigPath != filepath.Join(AppPath, "conf", "app.conf") {
		err := ParseConfig()
		if err != nil && AppConfigPath != filepath.Join(workPath, "conf", "app.conf") {
			// configuration is critical to app, panic here if parse failed
			panic(err)
		}
	}

	// do hooks function
	for _, hk := range hooks {
		err := hk()
		if err != nil {
			panic(err)
		}
	}

	if SessionOn {
		var err error
		sessionConfig := AppConfig.String("sessionConfig")
		if sessionConfig == "" {
			sessionConfig = `{"cookieName":"` + SessionName + `",` +
				`"gclifetime":` + strconv.FormatInt(SessionGCMaxLifetime, 10) + `,` +
				`"providerConfig":"` + SessionSavePath + `",` +
				`"secure":` + strconv.FormatBool(HttpTLS) + `,` +
				`"sessionIDHashFunc":"` + SessionHashFunc + `",` +
				`"sessionIDHashKey":"` + SessionHashKey + `",` +
				`"enableSetCookie":` + strconv.FormatBool(SessionAutoSetCookie) + `,` +
				`"cookieLifeTime":` + strconv.Itoa(SessionCookieLifeTime) + `}`
		}
		GlobalSessions, err = session.NewManager(SessionProvider,
			sessionConfig)
		if err != nil {
			panic(err)
		}
		go GlobalSessions.GC()
	}

	err := BuildTemplate(ViewsPath)
	if err != nil {
		if RunMode == "dev" {
			Warn(err)
		}
	}

	middleware.VERSION = VERSION
	middleware.AppName = AppName
	middleware.RegisterErrorHandler()
}
```

从代码看到在`Run()`的第一步，初始化`AppConfig`，调用`hooks`，初始化`GlobalSessions`，编译模板`BuildTemplate()`，和加载中间件`middleware.RegisterErrorHandler()`，分别简单叙述。

### 1.1 加载配置

加载配置的代码是：

```go
if AppConfigPath != filepath.Join(AppPath, "conf", "app.conf") {
	err := ParseConfig()
	if err != nil && AppConfigPath != filepath.Join(workPath, "conf", "app.conf") {
		// configuration is critical to app, panic here if parse failed
		panic(err)
	}
}
```

判断配置文件是不是`AppPath/conf/app.conf`，如果不是就`ParseConfig()`。显然他之前就已经加载过一次了。找了一下，在[config.go#L152](https://github.com/astaxie/beego/blob/master/config.go#L152)，具体加载什么就不说明了。需要说明的是`AppPath`和`workPath`这俩变量。找到定义[config.go#72](https://github.com/astaxie/beego/blob/master/config.go#L72)：

```go
workPath, _ = os.Getwd()
workPath, _ = filepath.Abs(workPath)
// initialize default configurations
AppPath, _ = filepath.Abs(filepath.Dir(os.Args[0]))

AppConfigPath = filepath.Join(AppPath, "conf", "app.conf")

if workPath != AppPath {
	if utils.FileExists(AppConfigPath) {
		os.Chdir(AppPath)
	} else {
		AppConfigPath = filepath.Join(workPath, "conf", "app.conf")
	}
}
```

`workPath`是`os.Getwd()`，即当前的目录；`AppPath`是`os.Args[0]`，即二进制文件所在目录。有些情况下这两个是不同的。比如把命令加到`PATH`中，然后cd到别的目录执行。`beego`以二进制文件所在目录为优先。如果二进制文件所在目录没有发现`conf/app.conf`，再去`workPath`里找。


### 1.2 Hooks

`hooks`就是钩子，在加载配置后就执行，这是要做啥呢？在 [beego.go#L173](https://github.com/astaxie/beego/blob/master/beego.go#L173) 添加新的hook：

```go
// The hookfunc will run in beego.Run()
// such as sessionInit, middlerware start, buildtemplate, admin start
func AddAPPStartHook(hf hookfunc) {
	hooks = append(hooks, hf)
}
```

`hooks`的定义在[beego.go#L19](https://github.com/astaxie/beego/blob/master/beego.go#L19)：

```go
type hookfunc func() error //hook function to run
var hooks []hookfunc       //hook function slice to store the hookfunc
```

`hook`就是`func() error`类型的函数。那么为什么调用`hooks`可以实现代码注释中的如`middleware start, build template`呢？因为`beego`使用的是单实例的模式。

### 1.3 单实例

`beego`的核心结构是`beego.APP`，保存路由调度结构`*beego.ControllerRegistor`。从`beego.Run()`方法的代码`BeeApp.Run()`发现，`beego`有一个全局变量`BeeApp`是实际调用的`*beego.APP`实例。也就是说整个`beego`就是一个实例，不需要类似`NewApp()`这样的写法。

因此，很多结构都作为全局变量如`beego.BeeApp`暴露在外。详细的定义在 [config.go#L18](https://github.com/astaxie/beego/blob/master/config.go#L18)，特别注意一下`SessionProvider(string)`，马上就要提到。

### 1.4 会话 `GlobalSessions`

继续`beego.Run()`的阅读，`hooks`调用完毕后，初始化会话`GlobalSessions`：

```go
if SessionOn {
	var err error
	sessionConfig := AppConfig.String("sessionConfig")
	if sessionConfig == "" {
		sessionConfig = `{"cookieName":"` + SessionName + `",` +
			`"gclifetime":` + strconv.FormatInt(SessionGCMaxLifetime, 10) + `,` +
			`"providerConfig":"` + SessionSavePath + `",` +
			`"secure":` + strconv.FormatBool(HttpTLS) + `,` +
			`"sessionIDHashFunc":"` + SessionHashFunc + `",` +
			`"sessionIDHashKey":"` + SessionHashKey + `",` +
			`"enableSetCookie":` + strconv.FormatBool(SessionAutoSetCookie) + `,` +
			`"cookieLifeTime":` + strconv.Itoa(SessionCookieLifeTime) + `}`
	}
	GlobalSessions, err = session.NewManager(SessionProvider,
		sessionConfig)
	if err != nil {
		panic(err)
	}
	go GlobalSessions.GC()
}
```

`beego.SessionOn`定义是否启动Session功能，然后`sessionConfig`是Session的配置，如果配置为空，就使用拼接的默认配置。`sessionConfig`是json格式。

`session.NewManager()`返回`*session.Manager`，session的数据存储引擎是`beego.SessionProvider`定义，比如"file"，文件存储。

`go GlobalSessions.GC()`开启一个goroutine来处理session的回收。阅读一下`GC()`的代码，在 [session/session.go#L183](https://github.com/astaxie/beego/blob/master/session/session.go#L183)：

```go
func (manager *Manager) GC() {
	manager.provider.SessionGC()
	time.AfterFunc(time.Duration(manager.config.Gclifetime)*time.Second, func() { manager.GC() })
}
```

这是个**无限循环**。`time.AfterFunc()`在经过一段时间间隔`time.Duration(...)`之后，又调用自己，相当于又开始启动`time.AfterFunc()`等待下一次到期。`manager.provider.SessionGC()`是不同session存储引擎的回收方法（其实是`session.Provider`接口的）。

### 1.5 模板构建

继续`beego.Run()`，session初始化后，构建模板：

```go
err := BuildTemplate(ViewsPath)
```

`beego.ViewsPath`是模板的目录啦，不多说。仔细来看看`BuildTemplate()`函数，[template.goL#114](https://github.com/astaxie/beego/blob/master/template.go#L114)：

```go
// build all template files in a directory.
// it makes beego can render any template file in view directory.
func BuildTemplate(dir string) error {
	if _, err := os.Stat(dir); err != nil {
		if os.IsNotExist(err) {
			return nil
		} else {
			return errors.New("dir open err")
		}
	}
	self := &templatefile{
		root:  dir,
		files: make(map[string][]string),
	}
	err := filepath.Walk(dir, func(path string, f os.FileInfo, err error) error {
		return self.visit(path, f, err)
	})
	if err != nil {
		fmt.Printf("filepath.Walk() returned %v\n", err)
		return err
	}
	for _, v := range self.files {
		for _, file := range v {
			t, err := getTemplate(self.root, file, v...)
			if err != nil {
				Trace("parse template err:", file, err)
			} else {
				BeeTemplates[file] = t
			}
		}
	}
	return nil
}
```

比较复杂。一点点来看，`os.Stat(dir)`判断目录是否存在。`filepath.Walk()`走一边目录里的文件，记录在`self.files`里面。循环`self.files`中的`file`（map[dir][]file])，用`getTemplate`获取`*template.Template`实例，保存在`beego.BeeTemplates`（map[string]*template.Template）。

为什么要**预先编译**模板？想像一下，如果每次请求，都去寻找模板再编译一遍。这显然是个浪费的。而且如果模板复杂，嵌套众多，编译速度会是很大的问题。因此存下编译好的`*template.Template`是必然的选择。但是，编译后模板的修改不能立即响应了，怎么办呢？先继续看下去。

### 1.6 中间件

`middleware`包目前似乎只有错误处理的功能。
```go
middleware.RegisterErrorHandler()
```
只是注册默认的错误处理方法 `middleware.NotFound` 等几个。

### 1.7 beeAdminApp

```go
if EnableAdmin {
	go beeAdminApp.Run()
}
```

`beeAdminApp`也是一个`*beego.adminApp`，负责系统监控、性能检测、访问统计和健康检查等。具体的介绍和使用可以访问[文档](http://beego.me/docs/advantage/monitor.md)。

## 2. HTTP服务

写了这么多，终于要开始讲核心结构`beego.BeeApp`的启动：

```go
BeeApp.Run()
```

`Run()`的实现代码在[app.go#L29](https://github.com/astaxie/beego/blob/master/app.go#L29)。代码较长，看看最重要的一段：

```go
if UseFcgi {
	if HttpPort == 0 {
		l, err = net.Listen("unix", addr)
	} else {
		l, err = net.Listen("tcp", addr)
	}
	if err != nil {
		BeeLogger.Critical("Listen: ", err)
	}
	err = fcgi.Serve(l, app.Handlers)
} else {
	if EnableHotUpdate {
		server := &http.Server{
			Handler:      app.Handlers,
			ReadTimeout:  time.Duration(HttpServerTimeOut) * time.Second,
			WriteTimeout: time.Duration(HttpServerTimeOut) * time.Second,
		}
		laddr, err := net.ResolveTCPAddr("tcp", addr)
		if nil != err {
			BeeLogger.Critical("ResolveTCPAddr:", err)
		}
		l, err = GetInitListener(laddr)
		theStoppable = newStoppable(l)
		err = server.Serve(theStoppable)
		theStoppable.wg.Wait()
		CloseSelf()
	} else {
		s := &http.Server{
			Addr:         addr,
			Handler:      app.Handlers,
			ReadTimeout:  time.Duration(HttpServerTimeOut) * time.Second,
			WriteTimeout: time.Duration(HttpServerTimeOut) * time.Second,
		}
		if HttpTLS {
			err = s.ListenAndServeTLS(HttpCertFile, HttpKeyFile)
		} else {
			err = s.ListenAndServe()
		}
	}
}
```

`beego.UseFcgi`定义是否使用`fast-cgi`服务，而不是HTTP。另一部分是启动HTTP。里面有个重要功能`EnableHotUpdate`————**热更新**。对他的描述，可以看看官方[文档](http://beego.me/docs/advantage/reload.md)。

### 2.1 HTTP过程总览

上面的代码看得到`*http.Server.Handler`是`app.Handlers`，即`*beego.ControllerRegistor`，`ServeHTTP`就定义在代码[router.go#L431](https://github.com/astaxie/beego/blob/master/router.go#L431)。非常长，我们检出重要的部分来说说。

首先是要创建当前请求的上下文：

```go
// init context
context := &beecontext.Context{
	ResponseWriter: w,
	Request:        r,
	Input:          beecontext.NewInput(r),
	Output:         beecontext.NewOutput(),
}
context.Output.Context = context
context.Output.EnableGzip = EnableGzip
```

`context`的类型是`*context.Context`，把当前的`w(http.ResponseWriter)`和`r(*http.Request)`写在`context`的字段中。

然后，定义了过滤器`filter`的调用方法，把`context`传递给过滤器操作：

```go
do_filter := func(pos int) (started bool) {
	if p.enableFilter {
		if l, ok := p.filters[pos]; ok {
			for _, filterR := range l {
				if ok, p := filterR.ValidRouter(r.URL.Path); ok {
					context.Input.Params = p
					filterR.filterFunc(context)
					if w.started {
						return true
					}
				}
			}
		}
	}
	return false
}
```

然后，加载Session：

```go
if SessionOn {
	context.Input.CruSession = GlobalSessions.SessionStart(w, r)
	defer func() {
		context.Input.CruSession.SessionRelease(w)
	}()
}
```

`defer`中的`SessionRelease()`是将session持久化到存储引擎中，比如写入文件保存。

然后，判断请求方式是否支持：

```go
if !utils.InSlice(strings.ToLower(r.Method), HTTPMETHOD) {
	http.Error(w, "Method Not Allowed", 405)
	goto Admin
}
```

这里看一看到 `goto Admin`，就是执行`AdminApp`的监控操作，记录这次请求的相关信息。`Admin`定义在整个HTTP执行的最后：

```go
Admin:
	//admin module record QPS
	if EnableAdmin {
		timeend := time.Since(starttime)
		if FilterMonitorFunc(r.Method, requestPath, timeend) {
			if runrouter != nil {
				go toolbox.StatisticsMap.AddStatistics(r.Method, requestPath, runrouter.Name(), timeend)
			} else {
				go toolbox.StatisticsMap.AddStatistics(r.Method, requestPath, "", timeend)
			}
		}
	}
```

所以`goto Admin`直接就跳过中间过程，走到HTTP执行的最后了。显然，当请求方式不支持的时候，直接跳到HTTP执行最后。如果不启用`AdminApp`，那就是HTTP执行过程结束。

继续阅读，开始处理静态文件了：

```go
if serverStaticRouter(context) {
	goto Admin
}
```

然后处理POST请求的内容体：

```go
if context.Input.IsPost() {
	if CopyRequestBody && !context.Input.IsUpload() {
		context.Input.CopyBody()
	}
	context.Input.ParseFormOrMulitForm(MaxMemory)
}
```

执行两个前置的过滤器：

```go
if do_filter(BeforeRouter) {
	goto Admin
}

if do_filter(AfterStatic) {
	goto Admin
}
```

不过我觉得这俩顺序怪怪的，应该先`AfterStatic`后`BeforeRouter`。需要注意，过滤器如果返回`false`，整个执行就结束（跳到最后）。

继续阅读，然后判断有没有指定执行的控制器和方法：

```go
if context.Input.RunController != nil && context.Input.RunMethod != "" {
	findrouter = true
	runMethod = context.Input.RunMethod
	runrouter = context.Input.RunController
}
```

如果过滤器执行后，对`context`指定了执行的控制器和方法，就用指定的。

继续，路由的寻找开始，有三种路由：

```go
if !findrouter {
	for _, route := range p.fixrouters {
		n := len(requestPath)
		if requestPath == route.pattern {
			runMethod = p.getRunMethod(r.Method, context, route)
			if runMethod != "" {
				runrouter = route.controllerType
				findrouter = true
				break
			}
		}
		//......
	}
}
```

`p.fixrouters`就是不带正则的路由，比如`/user`。`route.controllerType`的类型是`reflect.Type`，后面会用来创建控制器实例。`p.getRunMethod()`获取实际请求方式。为了满足浏览器无法发送表单`PUT`和`DELETE`方法，可以用表单域`_method`值代替。（注明一下`p`就是`*beego.ControllerRegistor`。

接下来当然是正则的路由：

```go
if !findrouter {
	//find a matching Route
	for _, route := range p.routers {

		//check if Route pattern matches url
		if !route.regex.MatchString(requestPath) {
			continue
		}
        // ......
		runMethod = p.getRunMethod(r.Method, context, route)
		if runMethod != "" {
			runrouter = route.controllerType
			context.Input.Params = params
			findrouter = true
			break
		}
	}
}
```

正则路由比如`/user/:id:int`，这种带参数的。匹配后的参数会记录在`context.Input.Params`中。

还没找到，就看看是否需要自动路由：

```go
if !findrouter && p.enableAuto {
	// ......
	for cName, methodmap := range p.autoRouter {
		// ......
	}
}
```

把所有路由规则走完，还是没有找到匹配的规则：

```go
if !findrouter {
	middleware.Exception("404", rw, r, "")
	goto Admin
}
```

另一种情况就是找到路由规则咯，且看下文。

### 2.2 路由调用

上面的代码发现路由的调用依赖`runrouter`和`runmethod`变量。他们值觉得了到底调用什么控制器和方法。来看看具体实现：

```go
if findrouter {
	//execute middleware filters
	if do_filter(BeforeExec) {
		goto Admin
	}

	//Invoke the request handler
	vc := reflect.New(runrouter)
	execController, ok := vc.Interface().(ControllerInterface)
	if !ok {
		panic("controller is not ControllerInterface")
	}

	//call the controller init function
	execController.Init(context, runrouter.Name(), runMethod, vc.Interface())

	//if XSRF is Enable then check cookie where there has any cookie in the  request's cookie _csrf
	if EnableXSRF {
		execController.XsrfToken()
		if r.Method == "POST" || r.Method == "DELETE" || r.Method == "PUT" ||
			(r.Method == "POST" && (r.Form.Get("_method") == "delete" || r.Form.Get("_method") == "put")) {
			execController.CheckXsrfCookie()
		}
	}

	//call prepare function
	execController.Prepare()

	if !w.started {
		//exec main logic
		switch runMethod {
		case "Get":
			execController.Get()
		case "Post":
			execController.Post()
		case "Delete":
			execController.Delete()
		case "Put":
			execController.Put()
		case "Head":
			execController.Head()
		case "Patch":
			execController.Patch()
		case "Options":
			execController.Options()
		default:
			in := make([]reflect.Value, 0)
			method := vc.MethodByName(runMethod)
			method.Call(in)
		}

		//render template
		if !w.started && !context.Input.IsWebsocket() {
			if AutoRender {
				if err := execController.Render(); err != nil {
					panic(err)
				}

			}
		}
	}

	// finish all runrouter. release resource
	execController.Finish()

	//execute middleware filters
	if do_filter(AfterExec) {
		goto Admin
	}
}
```

研读一下，最开始的又是过滤器：

```go
if do_filter(BeforeExec) {
	goto Admin
}
```

`BeforeExec`执行控制器方法前的过滤。

然后，创建一个新的控制器实例：

```go
vc := reflect.New(runrouter)
execController, ok := vc.Interface().(ControllerInterface)
if !ok {
	panic("controller is not ControllerInterface")
}

//call the controller init function
execController.Init(context, runrouter.Name(), runMethod, vc.Interface())
```

`reflect.New()`创建新的实例，用`vc.Interface().(ControllerInterface)`取出，调用接口的`Init`方法，将请求的上下文等传递进去。
这里就说明为什么不能存下控制器实例给每次请求使用，因为每次请求的上下文是**不同的**。

```go
execController.Prepare()
```

控制器的准备工作，这里可以写用户登录验证等。

然后根据`runmethod`执行控制器对应的方法，非接口定义的方法，用`reflect.Call`调用。

```go
if !w.started && !context.Input.IsWebsocket() {
	if AutoRender {
		if err := execController.Render(); err != nil {
			panic(err)
		}
	}
}
```

如果自动渲染`AutoRender`，就调用`Render()`方法渲染页面。

```go
execController.Finish()

//execute middleware filters
if do_filter(AfterExec) {
	goto Admin
}
```

控制器最后一刀`Finish`搞定，然后过滤器`AfterExec`使用。

总结起来，`beego.ControllerInterface`接口方法的`Init`,`Prepare`,`Render`和`Finish`发挥很大作用。那就来研究一下。

## 3. 控制器和视图

### 3.1 控制器接口

控制器接口`beego.ControllerInterface`的定义在[controller.go#L47](https://github.com/astaxie/beego/blob/master/controller.go#L47)：

```go
type ControllerInterface interface {
	Init(ct *context.Context, controllerName, actionName string, app interface{})
	Prepare()
	Get()
	Post()
	Delete()
	Put()
	Head()
	Patch()
	Options()
	Finish()
	Render() error
	XsrfToken() string
	CheckXsrfCookie() bool
}
```

官方的实现`beego.Controller`定义在[controller.go#L29](https://github.com/astaxie/beego/blob/master/controller.go#L29)：

```go
type Controller struct {
	Ctx            *context.Context
	Data           map[interface{}]interface{}
	controllerName string
	actionName     string
	TplNames       string
	Layout         string
	LayoutSections map[string]string // the key is the section name and the value is the template name
	TplExt         string
	_xsrf_token    string
	gotofunc       string
	CruSession     session.SessionStore
	XSRFExpire     int
	AppController  interface{}
	EnableReander  bool
}
```

内容好多，没必要全部都看看，重点在`Init`,`Prepare`,`Render`和`Finish`这四个。

### 3.2 控制器的实现

`Init`方法：

```go
// Init generates default values of controller operations.
func (c *Controller) Init(ctx *context.Context, controllerName, actionName string, app interface{}) {
	c.Layout = ""
	c.TplNames = ""
	c.controllerName = controllerName
	c.actionName = actionName
	c.Ctx = ctx
	c.TplExt = "tpl"
	c.AppController = app
	c.EnableReander = true
	c.Data = ctx.Input.Data
}
```

没什么话说，一堆赋值。唯一要谈的是`c.EnableReander`，这种拼写错误实在是，掉阴沟里。实际的意思是`EnableRender`。

`Prepare`和`Finish`方法：

```go
// Prepare runs after Init before request function execution.
func (c *Controller) Prepare() {

}

// Finish runs after request function execution.
func (c *Controller) Finish() {

}
```

空的！原来我要自己填内容啊。

`Render`方法：

```go
// Render sends the response with rendered template bytes as text/html type.
func (c *Controller) Render() error {
	if !c.EnableReander {
		return nil
	}
	rb, err := c.RenderBytes()

	if err != nil {
		return err
	} else {
		c.Ctx.Output.Header("Content-Type", "text/html; charset=utf-8")
		c.Ctx.Output.Body(rb)
	}
	return nil
}
```

### 3.3 视图渲染

渲染的核心方法是`c.RenderBytes()`:

```go
// RenderBytes returns the bytes of rendered template string. Do not send out response.
func (c *Controller) RenderBytes() ([]byte, error) {
	//if the controller has set layout, then first get the tplname's content set the content to the layout
	if c.Layout != "" {
		if c.TplNames == "" {
			c.TplNames = strings.ToLower(c.controllerName) + "/" + strings.ToLower(c.actionName) + "." + c.TplExt
		}
		if RunMode == "dev" {
			BuildTemplate(ViewsPath)
		}
		newbytes := bytes.NewBufferString("")
		if _, ok := BeeTemplates[c.TplNames]; !ok {
			panic("can't find templatefile in the path:" + c.TplNames)
			return []byte{}, errors.New("can't find templatefile in the path:" + c.TplNames)
		}
		err := BeeTemplates[c.TplNames].ExecuteTemplate(newbytes, c.TplNames, c.Data)
		if err != nil {
			Trace("template Execute err:", err)
			return nil, err
		}
		tplcontent, _ := ioutil.ReadAll(newbytes)
		c.Data["LayoutContent"] = template.HTML(string(tplcontent))

		if c.LayoutSections != nil {
			for sectionName, sectionTpl := range c.LayoutSections {
				if sectionTpl == "" {
					c.Data[sectionName] = ""
					continue
				}

				sectionBytes := bytes.NewBufferString("")
				err = BeeTemplates[sectionTpl].ExecuteTemplate(sectionBytes, sectionTpl, c.Data)
				if err != nil {
					Trace("template Execute err:", err)
					return nil, err
				}
				sectionContent, _ := ioutil.ReadAll(sectionBytes)
				c.Data[sectionName] = template.HTML(string(sectionContent))
			}
		}

		ibytes := bytes.NewBufferString("")
		err = BeeTemplates[c.Layout].ExecuteTemplate(ibytes, c.Layout, c.Data)
		if err != nil {
			Trace("template Execute err:", err)
			return nil, err
		}
		icontent, _ := ioutil.ReadAll(ibytes)
		return icontent, nil
	} else {
		//......
	}
	return []byte{}, nil
}
```

看起来很复杂，主要是两种情况，有没有Layout。如果有Layout：

```go
err := BeeTemplates[c.TplNames].ExecuteTemplate(newbytes, c.TplNames, c.Data)
// ......
tplcontent, _ := ioutil.ReadAll(newbytes)
c.Data["LayoutContent"] = template.HTML(string(tplcontent))
```

渲染模板文件，就是布局的主内容。

```go
for sectionName, sectionTpl := range c.LayoutSections {
	if sectionTpl == "" {
		c.Data[sectionName] = ""
		continue
	}

	sectionBytes := bytes.NewBufferString("")
	err = BeeTemplates[sectionTpl].ExecuteTemplate(sectionBytes, sectionTpl, c.Data)
	// ......
	sectionContent, _ := ioutil.ReadAll(sectionBytes)
	c.Data[sectionName] = template.HTML(string(sectionContent))
}
```

渲染布局里的别的区块`c.LayoutSections`。

```go
ibytes := bytes.NewBufferString("")
err = BeeTemplates[c.Layout].ExecuteTemplate(ibytes, c.Layout, c.Data)
// ......
icontent, _ := ioutil.ReadAll(ibytes)
return icontent, nil
```

最后是渲染布局文件，`c.Data`里带有所有布局的主内容和区块，可以直接赋值在布局里。

渲染过程有趣的代码：

```go
if RunMode == "dev" {
	BuildTemplate(ViewsPath)
}
```

开发状态下，每次渲染都会重新`BuildTemplate()`。这样就可以理解，最初渲染模板并存下`*template.Template`，生产模式下，是不会响应即时的模版修改。

## 总结

本文对`beego`的执行过程进行了分析。一个Web应用，运行的过程就是路由分发，路由执行和结果渲染三个主要过程。本文没有非常详细的解释`beego`源码的细节分析，但是还是有几个重要问题进行的说明：

* 路由规则的分类，固定的，还是正则，还是自动的。不同的路由处理方式不同，需要良好设计
* 控制器的操作其实就是上下文的处理，使用控制器类，还是函数，需要根据应用考量。
* 视图的效率控制需要严格把关，而且如何简单的设计就能满足复杂模板的使用，需要仔细考量。

`beego`本身复杂，他的很多实现其实并不是很简洁直观。当然随着功能越来越强大，`beego`会越来越好的。
