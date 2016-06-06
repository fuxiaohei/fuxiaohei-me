```toml
title = "搭建 React , Babel 和 Webpack"
slug = "react-babel-webpack-start"
date = "2016-05-05 19:00:34"
update_date = "2016-05-06 00:01:01"
author = "fuxiaohei"
tags = ["react","babel","webpack"]
```

公司打算用 React 做一个微信项目。我以前玩过一阵子 React，就被安排给 PHP 和前端指导入门 React。当时玩 React 的时候还是 0.11.x 的时代，如今已经是 15.0.x (0.15.x)。Webpack 和 Babel 也改进许多，搭建和使用变得方便。

[React](https://facebook.github.io/react/) 已经成为前端很流行的工具，很多公司都在使用。根据官网的介绍，React是将界面组件化开发的工具，只是 MVC 中 V 的角色。通过 [Flux](https://facebook.github.io/flux/) 的概念，维持和传递数据状态给对应的组件，完成数据流动。

随着 ECMAScript 6 (ES6,ES2015) 标准的发布，越来越多开发者使用新的语法。由于浏览器的跟进速度略慢，ES6 的 JavaScript 代码还是需要翻译到 ES5 才能在浏览器正确的运行。[Babel](https://babeljs.io/) 就是翻译 ES6 到 ES5 代码的工具。同时还可以处理 React 的 JSX 格式到一般 JS 代码。

前端项目除了 JavaScript 还有 CSS 等内容。[Webpack](https://webpack.github.io/) 是一套各种前端工具协调工作的总工具。整合 Babel 和很多别的工具以实现。

<!--more-->

### 准备

前端工具都依赖 Node 和 npm 仓库，所以安装 Node 是必须的步骤。直接从官网下载安装就行，不再赘述。

首先创建一个 npm 项目：

	npm init

根据提示填好内容后，开始安装 React 库。React 已经被分为 `react` 和 `react-dom` 两个部分，因为工作需要，我这里固定使用 0.14.8 版本。

	npm install --save-dev react@0.14.8
	npm install --save-dev react-dom@0.14.8

安装 `webpack` 和 `webpack-dev-server` 辅助开发编译：

	npm install --save-dev webpack
	npm install -g webpack-dev-server

安装 `babel` 的相关库编译 JavaScript，其中 `babel-preset-es2015` 处理 ES6，`babel-preset-react` 处理 JSX：

	npm install --save-dev babel-loader
	npm install --save-dev babel-core
	npm install --save-dev babel-preset-es2015
	npm install --save-dev babel-preset-react

### 写组件

首先写一个 `React` 组件 `hello.jsx`，用 ES 6 的语法，比较贴近别的高级语言的样子：

```javascript
import React from "react"

class Hello extends React.Component{
     constructor(props){
        super(props)
    }
    render(){
        return <h1 id="title">Hello</h1>
    }
}

export default Hello
```
使用 ES6 的 `extends` 关键字继承 `React.Component` 组件的操作，一定记住写构造函数 `constructor(props)` 里的 `super(props)`。`export default` 将组件从模块暴露出去用于 `import`。

然后写一个入口文件 `app.jsx` 把组件渲染给页面 `#root` 元素：

```javascript
import React from "react"
import ReactDOM from "react-dom"
import Hello from "./hello.jsx"

class App extends React.Component{
    constructor(props){
        super(props)
    }
    render(){
        return <div id="app">
            <Hello/>
        </div>
    }
}

ReactDOM.render(<App/>,document.getElementById("root"))
```

然后写个 HTML 文件 `index.html` 展示内容：

```html
<!doctype html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Hello React</title>
</head>
<body>
    <div id="root">
        this is root container, the content should not be replaced by react components
    </div>
    <script src="bundle.js"></script>
</body>
</html>
```

这里我们标记把所有的 JS 内容编译到 `bundle.js` 来使用。

### 编译

写好了组件就需要 `webpack` 来编译和处理 ES6 和 JSX 语法。写 `webpack.config.js` 去定义 `webpack` 的编译内容：

```javascript
var path = require('path');
var webpack = require('webpack');
 
module.exports = {
  entry: './app.jsx', // 入口文件，单入口 app.jsx 文件
  output: { path: __dirname, filename: 'bundle.js' }, // 编译到的文件
  module: {
    loaders: [ // 使用特定的加载器 loader 处理特定的文件
      {
        test: /.jsx?$/, // 文件过滤规则
        loader: 'babel-loader',
        exclude: /node_modules/,
        query: {
          presets: ['es2015', 'react'] // es2015 处理 ES6 语法，react 处理 jsx 语法
        }
      }
    ]
  },
};
```

来让我们运行一下 `webpack`：

```bash
$ webpack
Hash: 06cff3023dd7ad8e3b4b
Version: webpack 1.13.0
Time: 1111ms
    Asset    Size  Chunks             Chunk Names
bundle.js  694 kB       0  [emitted]  main
    + 169 hidden modules
```

要实时看到效果，就用 `webpack-dev-server` 来运行：

```bash
$ webpack-dev-server --process --colors
http://localhost:8080/webpack-dev-server/
webpack result is served from /
content is served from /srv/react
404s will fallback to /index.html
Hash: 494285dc137bde62f483
Version: webpack 1.13.0
......
```

访问 `localhost:8080` 就可以看到 `Hello` 显示在页面上。`webpack-dev-server` 支持热更新和增量编译，可以立即看到新的结果。但是默认不会输出编译好的内容，可以到 `localhost:8080/webpack-dev-server` 查看编译情况。


### 后记

公司准备用 React 做项目。我从头到尾说了一下 React, Flux 以及相关工具的说明和使用。而且写了一个简单的例子介绍 SPA 开发需要的库的使用方法。

虽然 `redux` 是目前最炙手可热的 Flux 实现，但是对于公司同事学习成本较高。他们不是太理解 redux 的理念。我推荐他们看简单的 Flux 实现 [alt.js](htto://alt.js.org)。Action-Store 的模型类似 Controller-Model 的模型概念，对后端程序员更容易学习。另外 `react-router` 已经稳定，成为页面路由的必然选择。

之后就看这个项目如何定义和使用组件来实现 SPA 的业务了。