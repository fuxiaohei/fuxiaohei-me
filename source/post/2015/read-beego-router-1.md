```toml
title = "阅读 Beego - 路由机制 (1)"
slug = "read-beego-router-1"
date = "2015-03-01 10:58:13"
author = "fuxiaohei"
tags = ["Go","Beego"]
```


[beego](https://github.com/astaxie/beego) 是国产著名的 Golang Web 框架，基于 MVC 模型，支持自动路由和Restful，并且有Config,Cache,Session,Orm等模块可以直接使用。MVC中最重要的就是路由。它实现从Web请求到对应控制器方法的映射。一般而言，路由包括三个部分：**添加规则**，**解析规则**和**匹配规则**。正好我分为三个部分，来分析beego的路由实现机制。

这一篇是添加路由规则的分析，从哪里入手呢？Beego有个开源论坛系统 [Wetalk](https://github.com/beego/wetalk)，也就是 [golanghome](http://golanghome.com) 的实现，从他的路由开始看起。

<!--more-->

Wetalk的路由设置很长，有很多的控制器需要注册。我从一段简单的看起，[wetalk.go#L79](https://github.com/beego/wetalk/blob/master/wetalk.go#L79)：

```go
posts := new(post.PostListRouter)
beego.Router("/", posts, "get:Home")
beego.Router("/:slug(recent|best|cold|favs|follow)", posts, "get:Navs")
beego.Router("/category/:slug", posts, "get:Category")
beego.Router("/topic/:slug", posts, "get:Topic;post:TopicSubmit")
```
### 一般路由

路由的注册方法是 `beego.Router(...)`，参数是URL规则，控制器对象和他内部对应的方法。具体这个方法怎么用，可以去参考官方文档。如何执行呢？继续看下去，[beego.go](https://github.com/astaxie/beego/blob/v1.4.3/beego.go#L61)：

```go
func Router(rootpath string, c ControllerInterface, mappingMethods ...string) *App {
	BeeApp.Handlers.Add(rootpath, c, mappingMethods...)
	return BeeApp
}
```

呵呵，方法落在`BeeApp.Hanlders`上。`BeeApp.Hanlders` 是 `*ControllerRegistor` 实例，对应的`Add`方法在 [router.go](https://github.com/astaxie/beego/blob/v1.4.3/router.go#L138)：

```go
func (p *ControllerRegistor) Add(pattern string, c ControllerInterface, mappingMethods ...string) {
	reflectVal := reflect.ValueOf(c)
	t := reflect.Indirect(reflectVal).Type()
	methods := make(map[string]string)
	if len(mappingMethods) > 0 {
		semi := strings.Split(mappingMethods[0], ";")
		for _, v := range semi {
			colon := strings.Split(v, ":")
			if len(colon) != 2 {
				panic("method mapping format is invalid")
			}
			comma := strings.Split(colon[0], ",")
			for _, m := range comma {
				if _, ok := HTTPMETHOD[strings.ToUpper(m)]; m == "*" || ok {
					if val := reflectVal.MethodByName(colon[1]); val.IsValid() {
						methods[strings.ToUpper(m)] = colon[1]
					} else {
						panic("'" + colon[1] + "' method doesn't exist in the controller " + t.Name())
					}
				} else {
					panic(v + " is an invalid method mapping. Method doesn't exist " + m)
				}
			}
		}
	}

	route := &controllerInfo{}
	route.pattern = pattern
	route.methods = methods
	route.routerType = routerTypeBeego
	route.controllerType = t
	if len(methods) == 0 {
		for _, m := range HTTPMETHOD {
			p.addToRouter(m, pattern, route)
		}
	} else {
		for k, _ := range methods {
			if k == "*" {
				for _, m := range HTTPMETHOD {
					p.addToRouter(m, pattern, route)
				}
			} else {
				p.addToRouter(k, pattern, route)
			}
		}
	}
}
```

比较长，一点一点看：

第一步，获取控制器的反射类型`reflect.Type`。

第二步，解析`mappingMethods`,即上面代码`beego.Router(...)`的第三个参数，比如`get:Topic;post:TopicSubmit`。从字面猜就是HTTP请求方式对应的控制器方法名称，像 GET -> PostListRouter.Topic()。分号分割多种HTTP请求，冒号分割HTTP请求和对应控制器方法。`HTTPMETHOD`限制支持的HTTP请求方式，不正常的panic。`*`意味着匹配所有`HTTPMETHOD`. 用反射获取一次对应方法，判断是否有效。

第三步，生成`controllerInfo{}`,并添加到路由中。`pattern`就是传入的URL规则，还没有解析。`methods`是解析好的路由参数中HTTP请求方式到控制器方法的映射。这里有个`routerTypeBeego`，标识`controllerInfo{}`是个一般的路由。还有`routerTypeRESTFul`和`routerTypeHandler`两种，会在下面说明。

接下来，就是看看`p.addToRouter(...)`是个啥啦！[router.go](https://github.com/astaxie/beego/blob/v1.4.3/router.go#L186):

```go
func (p *ControllerRegistor) addToRouter(method, pattern string, r *controllerInfo) {
	if !RouterCaseSensitive {
		pattern = strings.ToLower(pattern)
	}
	if t, ok := p.routers[method]; ok {
		t.AddRouter(pattern, r)
	} else {
		t := NewTree()
		t.AddRouter(pattern, r)
		p.routers[method] = t
	}
}
```

终于看到了`NewTree()`——路由树。所有路由规则的集合其实是一个`map[http_method]*Tree`。关于路由树的实现，我们下一篇文章再来详细阅读。


### RESTful 路由

beego 有个方法 `beego.RESTRouter(...)`。我本来以为这个方法是 RESTful 类型的路由，看源码发现，还是调用了 `beego.Router(...)`。找了一下，`routerTypeRESTFul`类型的路由，原来是在 [router.go](https://github.com/astaxie/beego/blob/v1.4.3/router.go#L318) 的 `beego.AddMethod(...)`:

```go
func (p *ControllerRegistor) AddMethod(method, pattern string, f FilterFunc) {
	if _, ok := HTTPMETHOD[strings.ToUpper(method)]; method != "*" && !ok {
		panic("not support http method: " + method)
	}
	route := &controllerInfo{}
	route.pattern = pattern
	route.routerType = routerTypeRESTFul
	route.runfunction = f
	methods := make(map[string]string)
	if method == "*" {
		for _, val := range HTTPMETHOD {
			methods[val] = val
		}
	} else {
		methods[strings.ToUpper(method)] = strings.ToUpper(method)
	}
	route.methods = methods
	for k, _ := range methods {
		if k == "*" {
			for _, m := range HTTPMETHOD {
				p.addToRouter(m, pattern, route)
			}
		} else {
			p.addToRouter(k, pattern, route)
		}
	}
}
```

也是生成一个`controllerInfo{}`提交给路由树。区别在`router.runfunction`,不是控制器的反射类型，是一个函数类型`FilterFunc`。那么这个RESTFul的路由在哪儿用的呢？

`beego.Get(...)` 就是 `beego.AddMethod("get",...)`。类似的`beego.Post(...)`,`beego.Put(...)`等等。换句话说这是一类路由，用来接收一个函数作为路由规则对应的方法，而不是一个控制器。

### HTTP Handler 路由

beego 还有`routerTypeHandler`类型的路由，添加的方法在`beego.Handler(...)` [router.go](https://github.com/astaxie/beego/blob/v1.4.3/router.go#L347) ：

```go
func (p *ControllerRegistor) Handler(pattern string, h http.Handler, options ...interface{}) {
	route := &controllerInfo{}
	route.pattern = pattern
	route.routerType = routerTypeHandler
	route.handler = h
	if len(options) > 0 {
		if _, ok := options[0].(bool); ok {
			pattern = path.Join(pattern, "?:all")
		}
	}
	for _, m := range HTTPMETHOD {
		p.addToRouter(m, pattern, route)
	}
}
```

生成`controllerInfo{}`的时候使用的是`http.Handler`，保存在`router.handler`字段。而且下面把这个路由的HTTP请求方式设置给所有支持的方式。


### 自动路由

自动路由是为了简化用控制器注册路由时，一个一个添加的麻烦。根据控制器的结构来自动添加规则，具体的地方在[router.go](https://github.com/astaxie/beego/blob/v1.4.3/router.go#L376):

```go
func (p *ControllerRegistor) AddAutoPrefix(prefix string, c ControllerInterface) {
	reflectVal := reflect.ValueOf(c)
	rt := reflectVal.Type()
	ct := reflect.Indirect(reflectVal).Type()
	controllerName := strings.TrimSuffix(ct.Name(), "Controller")
	for i := 0; i < rt.NumMethod(); i++ {
		if !utils.InSlice(rt.Method(i).Name, exceptMethod) {
			route := &controllerInfo{}
			route.routerType = routerTypeBeego
			route.methods = map[string]string{"*": rt.Method(i).Name}
			route.controllerType = ct
			pattern := path.Join(prefix, strings.ToLower(controllerName), strings.ToLower(rt.Method(i).Name), "*")
			patternInit := path.Join(prefix, controllerName, rt.Method(i).Name, "*")
			patternfix := path.Join(prefix, strings.ToLower(controllerName), strings.ToLower(rt.Method(i).Name))
			patternfixInit := path.Join(prefix, controllerName, rt.Method(i).Name)
			route.pattern = pattern
			for _, m := range HTTPMETHOD {
				p.addToRouter(m, pattern, route)
				p.addToRouter(m, patternInit, route)
				p.addToRouter(m, patternfix, route)
				p.addToRouter(m, patternfixInit, route)
			}
		}
	}
}
```

这里根据控制器的方法来拼接pattern。首先不处理控制器内置的方法`exceptMethod`，然后根据控制器的名称和方法，区分大小写地注册`prefix/controller/method/*`和`prefix/controller/method`两个规则到所有HTTP请求方式上。


### Final

综合来看，添加路由的过程，总结起来是添加`controllerInfo{}`到`*Tree`中。`controllerInfo{}`的结构是：

```go
type controllerInfo struct {
	pattern        string
	controllerType reflect.Type
	methods        map[string]string
	handler        http.Handler
	runfunction    FilterFunc
	routerType     int
}
```

可以看出，此时pattern —— 路由规则 —— 还没有经过解析。只是在methods的map中记录了HTTP请求方式和对应调用的方法，或者是在handler或runfunction直接保存了调用的方法函数。

那么下一篇，我们来阅读以下`controllerInfo{}`添加到`*Tree`中的过程。既然是路由树，那么层级规则，节点内容，就是重要的细节。

**Notice:** 本文基于 beego v1.4.3




