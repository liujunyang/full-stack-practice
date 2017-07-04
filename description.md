---

[TOC]

---
## 全栈开发的不同之处
开发环境也可以不用mock，直接提供后端server服务，这也是全栈开发和之前前端开发的不同之处。
可以把node后端当成中间层，请求旧后端并包装后返回。

## 本项目框架简介

项目使用webpack 1.15.0、vue 2.3.2、Node.js 6.9.2进行的开发。

使用模板[vue-element-admin-boilerplate](https://github.com/lynzz/element-admin)中关于开发的思想[^fn1]进行布局，然后又丰富了它，包装成分为开发、生产模式的最外层的server.js。

使用vue-element-admin-boilerplate模板默认生成的开发服务器只是相当于当前项目中client文件夹内部`client/build/dev-server.js`。

配置文件：.editorconfig、.eslintrc.js
项目介绍：readme.md。

---

## 效果
![效果图](https://raw.githubusercontent.com/lynzz/element-admin/master/screenshot/search.jpg)
---

## 思路拆解

### 文件夹结构
```
| - client            前端
| - dist              打包目录，也是服务器的静态资源路径
| - node_modules      
| - server            后端服务器
| - static            client静态资源
| - package.json      工程文件
| - server.js         开发入口服务器
```


#### layout各部分功能分配
核心的部分有3个，最外层的server.js，client文件夹，server文件夹。
最外层的server.js是开发使用的。
client文件夹相当于原来的前端。
server文件夹相当于原来的后端。
node_modules前后端公用。
总体上：把前后端结合到一个项目中进行开发。

最外层server.js使用了express框架。

### 开发环境vs生产环境

#### 不同之处
使用process.env.NODE_ENV（production为生产模式）并同时定义了一个变量useWebpackDev（false为生产模式）作为标志来判断是server.js是处于开发环境还是生产发布环境。
只有在npm run build的时候会将process.env.NODE_ENV设置为production（在client/build/build.js第4行）`env.NODE_ENV = 'production'`，useWebpackDev是在server_config中设置的，而server_config又是依靠NODE_ENV来确定是引用important.js(生成环境配置)还是development.js，在important.js中useWebpackDev为false（或undefined），在development.js中useWebpackDev为true。
使用NODE_ENV之外再useWebpackDev可以给定义当前是什么环境多一些扩展性，因为：`let env = process.env.NODE_ENV || 'development';`，所以可以不用是非此即彼，即只能在development和production之间选择。
important.js其实内容是类似development.js，只不过是存在雨服务器上，避免保密数据泄露在git里。

#### 相同之处
都是使用server.js提供服务，在生产环境也是当前的完整目录结构，依然是使用当前server.js提供服务。

开发环境和生产环境其实都是一样执行 `node server.js`，只不过其实分别用了`nodemon`或`pm2`。
开发环境使用webpack大礼包：webpack、webpack-dev-middleware、webpack-hot-middleware。
生产环境（已经build过）直接对根目录输出index.html（剩下的交给vue-router）及static服务。最后都是监听端口开启服务。

#### 关于`npm run build`
`npm run build`命令主要是使用webpack打包以及一些复制粘贴工作（docker部署是另外一码事）。

### 开发环境
自己造server并同时使用webpack的思路：
```
1.利用express自己造了一个server服务
1.5 可能希望在这里提供一些路由服务（参考下面对路由劫持的分析）
2.app.use(require('connect-history-api-fallback')())包装req.url
3.var compiler = webpack(webpackConfig)定义了读取哪些怎么读取资源
4.dev-middleware读取资源
5.hot-middleware提供热刷新
5.5 可能希望在这里提供一些路由服务（参考下面对路由劫持的分析）
```

#### 关于webpack-dev-server【我们项目没有使用，而是自己开发了一个server】
*参考文献[^fn3]*
webpack-dev-server模块内部在node_modules中包含webpack-dev-midlleware(把webpack生成的compiler变成express中间件的容器)。
```
The webpack-dev-server is a little Node.js Express server, which uses the webpack-dev-middleware to serve a webpack bundle. It also has a little runtime which is connected to the server via Sock.js.
```
webpack-dev-server提供了项目目录文件的static服务,在模块源文件第203行：
```
app.get("*", express.static(contentBase), serveIndex(contentBase));
```
比如经过测试发现，使用dev-server的rainvue项目在浏览器能访问到根目录的任何文件。

#### 关于express及其中间件
我们自己的server使用的是express。

关于[middleware中间件](https://expressjs.com/en/guide/using-middleware.html)看注解[^fn2]。可以连续写多个函数，即多个连续执行的中间件。`app.use([path,] callback [, callback...])`

根部如果一样，就执行中间件，比如定义了path为/abc  则/abc/ddd也执行该中间件
`app.use('/', function(){})`和`app.use(function(){})`没有区别，`'/' (root path)` 是默认值。

```
Mounts the specified middleware function or functions at the specified path: the middleware function is executed when the base of the requested path matches path.
```
*express的错误处理中间件，必须包含4个参数，这是区别于普通中间件的标志。*
```
app.use(function (err, req, res, next) {
  console.error(err.stack)
  res.status(500).send('Something broke!')
})
```

express.static是express内置中间件，提供静态资源服务。*可以同时有多个static。不存在代码覆盖关系，匹配到哪一个就直接返回了，后面的也就不执行了。即中间件的本性：谁在前面谁生效。*
```
//You can have more than one static directory per app:
app.use(express.static('public'))
app.use(express.static('uploads'))
app.use(express.static('files'))
```
```
// 本项目属于重复配置了static，可以后期清理下代码。
// server/index.js中 配置static服务路径是在生产环境中的路径
'/cms/getup/static' '/Users/liujunyang/henry/work/yktcms/server/dist/static'
// server.js中进入useWebpackDev的if后，也配置了static。
'/cms/getup/static' './static'
// else中也配置了static。
'/cms/getup/static' './dist/static'
```

#### 关于[connect-history-api-fallback](https://github.com/bripkens/connect-history-api-fallback)
功能：在单页应用中，处理当刷新页面或直接在地址栏访问非根页面的时候，返回404的bug。匹配非文件（路径中不带.）的get请求。
connect-history-api-fallback会 **包装** req.url。请求路径如果符合匹配规则（如/或路径如/dsfdsfd） 就会重写为/index.html（位于模块文件第71行），否则（如/test.do）就保持原样，最终都是进入next()，(模块文件的50-75行)，继续执行后面的中间件。
```
...
if (parsedUrl.pathname.indexOf('.') !== -1 &&
    options.disableDotRule !== true) {
  logger(
    'Not rewriting',
    req.method,
    req.url,
    'because the path includes a dot (.) character.'
  );
  return next();
}

rewriteTarget = options.index || '/index.html';
logger('Rewriting', req.method, req.url, 'to', rewriteTarget);
req.url = rewriteTarget;
next();
```

#### 关于[webpack-dev-middleware](https://webpack.github.io/docs/webpack-dev-middleware.html)中间件容器
1.生成的资源不是存在于文件夹，而是在内存中。
```
The webpack-dev-middleware is a small middleware for a connect-based middleware stack. It uses webpack to compile assets in-memory and serve them. When a compilation is running every request to the served webpack assets is blocked until we have a stable bundle.
*译：把资源编译进内存，在编译过程中，所有请求都不能进行，直到获得稳定的bundle。*
```

2.html-webpack-plugin只是提供渲染生成文件，至于生成到哪是根据compiler，比如webpack-dev-middleware 提供的服务是在内存中，则yktcms开发中，无论是生成index.html还是再生成别的文件，都是生成到内存中，在编辑器的文件目录中是看不到的。

3.webpack-dev-middleware提供的服务是利用compiler的配置编译在内存中的文件。
我们使用了html-webpack-plugin，dev模式开发时index.html不用server.js从views文件夹中进行render。所以即使把views中的index.html删除也不会影响dev模式开发。
```
new HtmlWebpackPlugin({
  filename: 'index.html',
  template: './client/index.html',
  inject: true
})
```
server在生产环境提供的views中的index.html是编译时通过`npm run build`生成的，并不是手动创建的。

4.webpack-dev-middleware 中有send，如果执行send返回了请求，就不会走express之后的中间件了。
```javascript
function processRequest() {
	...

	// server content
	var content = context.fs.readFileSync(filename);
	content = shared.handleRangeHeaders(content, req, res);
	res.setHeader("Access-Control-Allow-Origin", "*"); // To support XHR, etc.
	res.setHeader("Content-Type", mime.lookup(filename) + "; charset=UTF-8");
	res.setHeader("Content-Length", content.length);
	if(context.options.headers) {
		for(var name in context.options.headers) {
			res.setHeader(name, context.options.headers[name]);
		}
	}
	// Express automatically sets the statusCode to 200, but not all servers do (Koa).
	res.statusCode = res.statusCode || 200;
	if(res.send) res.send(content);
	else res.end(content);
}
```

#### 关于[webpack-hot-middleware](https://github.com/glenjamin/webpack-hot-middleware)
为webpack提供热刷新功能。
```
Webpack hot reloading using only webpack-dev-middleware. This allows you to add hot reloading into an existing server without webpack-dev-server.
```
使用方法的伪代码：
```
Add the following plugins to the plugins array
Add 'webpack-hot-middleware/client' into the entry array.
app.use(require("webpack-hot-middleware")(compiler));
```

#### 关于webpack的[entry](https://webpack.github.io/docs/configuration.html#entry)
*这里说这个主要是介绍webpack-hot-middleware如何把 webpack-hot-middleware/client 加入entry。*
entry单项如果是array，打包在一起输出最后一个。
webpack.dev.conf.js中的第9行：
```
baseWebpackConfig.entry[name] =['./client/build/dev-client'].concat(baseWebpackConfig.entry[name])
```
```
client/build/dev-client.js文件中的（require中省略了.js后缀）
'webpack-hot-middleware/client?reload=true’;其实是
'webpack-hot-middleware/client.js?reload=true';
```

#### 关于[walk](https://git.daplie.com/Daplie/node-walk)
node-walk不是中间件，是一个遍历文件夹的nodejs库，只是遍历，遍历过程中想 要做什么处理的话给传入相应的函数即可。
比如我们是利用walk模块遍历mock文件夹的文件，让指定的路由（`url`）能返回访问指定的文件（`require(mod)`）
```
...
var walk = require('walk')
var walker = walk.walk('./client/mock', {followLinks: false})
...
...
walker.on('file', function (root, fileStat, next) {
  if (!/\.js$/.test(fileStat.name)) next()

  var filepath = path.join(root, fileStat.name)

  var url = filepath.replace('client/mock', '/mock').replace('.js', '')
  var mod = filepath.replace('client/mock', './client/mock')

  app.all(url, require(mod))

  next()
})
```
```
// 匹配的本地mock的url，带.do后缀是为了不匹配connect-history-api-fallback以及vue-router的路由规则
...
if (useMock) {
  api = {
  ...
    article: '/mock/article/article.do',
    article_create: '/mock/article/create.do',
    ...
  }
}
...
```
注意：walk一定不要忘记最后的next()。

#### 一个请求进来之后的中间件处理顺序
**可能到某一步就直接结束了，像过多层筛子，拦住了就不往下走了**
1.如果符合放在前面的路由就直接结束。
2.connect-history-api-fallback。包装req.url。
3.webpack-dev-middleware的处理。
```
如果在内存中（即compiler生成的文件，如app.js，如index.html或htmlplugin生成的一系列html）输出req.url代表的文件，如/Users/liujunyang/henry/work/yktcms/dist/index.html。结束。

如果进入index.html,使用vue-router对当前路径再次进行匹配，劫持了所有的router，结束。不符合vue-router规则的才继续往下走next。
```
```
如果不在内存中，直接走next，也就不存在下面讨论的vue-router的过滤规则的事。
```

4.其他对文件的处理（能走到这的一般也就是文件了），比如上面介绍的walk的处理。

注意：在开发时，即useWebpackDev为true时，每个请求（不论文件还是接口）都会经过层层（不必然是所有的）中间件处理，所以即使是切换vue里面的路由其实也会把index.html本身再传输一遍。
而到线上之后，在输出index.html之后，凡是vue-router能匹配的请求都交给vue-router了。

#### vue-router过滤规则
1.vue-router.js的674-763行的代码是关于路径的正则匹配：凡是路径中带点.的路由*【除xxx.html外，对xxx.html的处理和不带点的路径是一样的】*，均不被vue-router匹配劫持。被匹配的就都会被vue-router处理（即劫持）。

2.在chrome的console中测试如下：
```
var PATH_REGEXP = new RegExp([
  // Match escaped characters that would otherwise appear in future matches.
  // This allows the user to escape special characters that won't transform.
  '(\\\\.)',
  // Match Express-style parameters and un-named parameters with a prefix
  // and optional suffixes. Matches appear as:
  //
  // "/:test(\\d+)?" => ["/", "test", "\d+", undefined, "?", undefined]
  // "/route(\\d+)"  => [undefined, undefined, undefined, "\d+", undefined, undefined]
  // "/*"            => ["/", undefined, undefined, undefined, undefined, "*"]
  '([\\/.])?(?:(?:\\:(\\w+)(?:\\(((?:\\\\.|[^\\\\()])+)\\))?|\\(((?:\\\\.|[^\\\\()])+)\\))([+*?])?|(\\*))'
].join('|'), 'g');

PATH_REGEXP
/(\\.)|([\/.])?(?:(?:\:(\w+)(?:\(((?:\\.|[^\\()])+)\))?|\(((?:\\.|[^\\()])+)\))([+*?])?|(\*))/g

PATH_REGEXP.test('http://localhost:20003/cms/getup/article-list2.dd')
true

PATH_REGEXP.exec('http://localhost:20003/cms/getup/article-list2.dd')
null

PATH_REGEXP.exec('http://localhost:20003/cms/getup/article-list2.')
null

PATH_REGEXP.test('http://localhost:20003/cms/getup/article-list2dd')
false

PATH_REGEXP.exec('http://localhost:20003/cms/getup/article-list2dd')
[":20003", undefined, undefined, "20003", undefined, undefined, undefined, undefined]

PATH_REGEXP.exec('http://localhost:20003/cms/getup/article-list2.html')
[":20003", undefined, undefined, "20003", undefined, undefined, undefined, undefined]
```

3.比如：在地址栏手动访问index.html，在webpack-dev-middleware输出index.html后，由于index.html页面中有app.js，app.js中安排了vue-router，那么接下来进入vue-router处理，由于`路径 index.html`能被vue-router的规则匹配，根据app.js的中路由的处理，fallback进入了404页面。
```
// fallback处理
routers.push({
  path: '*',
  name: 'notfound',
  component: require('./views/notfound')
})
```
但是如果有处理，就会进入相应的路由：
```
// 对路径 index.html匹配组件进行处理
{
    path: '/index.html',
    name: 'perm',
    component: require('./views/perm')
}
```

#### 测试在地址栏输入一个路径后，我们的服务器是如何处理的
假设作为webpack entry的那个包含有vue-router的js文件为：main.js。
在html-webpack-plugin的规则中，inject为true时main.js注入html文件，否则html文件没有main.js文件。
1.第一个测试：
在htmlplugin中添加一个test.html，设置inject为false。在地址栏输入相应的`pathto/test.html`，回车。
```
流程是这样的：
因为test.html文件是htmlplugin中的，所以被编译进了内存中，webpack-dev-middleware输出test.html文件并经server返回给浏览器。
因为没有main.js，所以没有vue-router再对'pathto/test.html'进行处理，结束。
webpack-dev-middleware中处理的url为：'/Users/liujunyang/henry/work/yktcms/dist/test.html'
```

2.第二个测试：
在htmlplugin中添加一个test2.html，设置inject为true.在地址栏输入相应的`pathto/test2.html`，回车。
```
流程是这样的：
因为test2.html文件是htmlplugin中的，所以被编译进了内存中，webpack-dev-middleware输出test2.html文件并经server返回给浏览器。
因为有main.js，所以加载并执行main.js。
main.js中有vue-router，'pathto/test2.html'能被vue-router匹配，但是根据main.js中的路由配置，并没有对'pathto/test2.html'进行处理，就fallback进入了404页面(这个流程和上面vue-router过滤规则中第3条对index.html的介绍是一样的)。
```

3.第三个测试：
在地址栏输入`pathto/test.do`，回车（测试文件类型的路径）。
```
流程是这样的：
因为test.do不是htmlplugin中的，不在内存中，不会提前被webpack-dev-middleware输出。
中间件直接next（也就没有输出index.html，更是根本没vue-router什么事）。
然后根据server有没有提供路由服务或static服务，找到了就输出，没有找到的话就告知“Cannot GET /dd.do”
```

### 生产环境

####概括
通过打包时`npm run build`之后生成的文件直接提供index.html服务，提供static服务。
```
没有connect-history-api-fallback
没有webpack-dev-middleware
没有webpack-hot-middleware
没有开发环境需要的mock（开发环境也可以不用mock，直接提供后端server服务，这也是全栈开发和之前前端开发的不同之处）。
```

##部署

### cms部署到雷
```sh
git pull
npm install
npm run build
pm2 restart cms
pm2 list
```

### docker部署到雨
TODO 详解docker。

##简单细节——解剖

### 概要
把webpack的配置文件放进了client端中，有历史原因（借鉴了`vue-element-admin-boilerplate`模板），其实是不应该的，应该放最外面。
涉及到的库、模式、工具等在下面的解剖中再逐一慢慢涉及。如redis、sequelize等。
下面的解剖都是简单解剖，详细细节再新开笔记进行单独介绍。本笔记着眼于整个项目的思路。

### 应用如何启动
如何启动开发环境 `npm run dev`

### 最外层server.js的解剖
基本上就是最外层布置了一个server。
根据环境判断条件决定进入哪个server。
开发环境需要devMiddleware。
生产环境由于是webpack打包好的，直接提供打包好的文件服务。
```javascript
...
// server毛坯
const app = require('./server/index')
...
// 什么环境都需要的路由处理（放在了devMiddleware处理的前面）
app.use(client_config.ROOT_ROUTER + '/api', cms_router);
app.use(client_config.ROOT_ROUTER + '/preview', cms_router);
app.use(client_config.ROOT_ROUTER + '/front', front_router); 
...
if (process.env.NODE_ENV !== 'production' && useWebpackDev) {
...
// 开发环境
  var compiler = webpack(webpackConfig)
...
  app.use(require('connect-history-api-fallback')())
...
  app.use(devMiddleware)
...
  app.use(hotMiddleware)
...
} else {
...
// 生产环境
  app.use(client_config.ROOT_ROUTER, function (req, res, next) {   // 渲染cms页面
    res.render('index.html');
  });
...
}
...
app.listen(server_config.PORT)
```

###client端的解剖
就是一个[vue2.0](https://cn.vuejs.org/)项目。
#### layout
```
| - client          前端
  | - build           开发、生产配置文件
  | - config          配置文件
  | - mock            模拟数据
  | - src             前端vue开发
  | - index.html      vue项目的入口html文件
```

#### html结构
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, minimum-scale=1, user-scalable=no, minimal-ui">
    <meta name="format-detection" content="telephone=no" />
    <meta name="keywords" content="">
    <meta name="Description" content="">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black">
    <link type="image/x-icon" rel="shortcut icon" href="">
    <title></title>
  </head>
  <body>
    <div id="app"></div>
    <script src="/cms/getup/static/js/tinymce/tinymce.min.js"></script>
    <script src="/cms/getup/static/js/qrcode.min.js"></script>
    <!-- built files will be auto injected -->
  </body>
</html>
```

#### 入口js
client/src/main.js
```javascript
import Vue from 'vue'
...
import VueRouter from 'vue-router'
import config from './config'
import App from './App'
...
Vue.use(VueRouter)
...
const router = new VueRouter({
  mode: 'history',
  base: config.ROOT_ROUTER,
  routes
})
...
/* eslint-disable no-new */
new Vue({
  el: '#app',
  template: '<App/>',
  router,
  components: {App}
})
```

#### 根模板组件
client/src/App.vue
```
<template>
  <transition>
    <router-view></router-view>
  </transition>
</template>
```

#### 路由控制
client/src/router-config.js
```javascript
/* eslint-disable */
...
const routers = [
  { path: '/', component: require('./views/index') },
  {
    path: '/article',
    name: 'article',
    component: require('./views/article/index'),
    children: [
      {
        path: 'list',
        name: 'article-list',
        component: require('./views/article/list')
      },
      {
        path: 'create',
        name: 'article-editor',
        component: require('./views/article/editor')
      },
    ]
  },
  ...
]

routers.push({
  path: '*',
  name: 'notfound',
  component: require('./views/notfound')
})

export default routers

```

#### request处理函数
path: `/client/src/api/request.js`
使用[axios](https://github.com/mzabriskie/axios)和[bluebird](https://github.com/petkaantonov/bluebird)包装了ajax请求。

使用方法：
```
// '/client/src/api/article.js'
import request from './request'
import Api from './api'

export function createActivity (params) {
  return request.post(Api.activity_create, params)
}
...
```

###server端的解剖
可以算是一个node项目。
#### layout
```
| - server            后端服务器
  | - config            配置
  | - controllers       处理业务逻辑
  | - helpers           工具方法
  | - models            数据库结构
  | - views             后端render的页面
  | - index.js          服务入口文件
  | - routes-front.js   路由：非cms
  | - routes_cmsapi.js  路由：cms

```

#### server端index.js
毛坯server。
使用express。

```javascript
'use strict';
const express = require('express');
...
// 造server
const app = express();
// 模板引擎使用ejs
app.set('views', path.join(__dirname, 'views'));
app.engine('html', require('ejs').renderFile);
app.set('view engine', 'ejs');
// 定义静态资源static服务
app.use(client_config.ROOT_ROUTER + '/static', express.static(path.join(__dirname, 'dist/static')));

// 配置统一的header处理
app.use((req, res, next) => {
	res.header("Access-Control-Allow-Origin", '*');
	res.header("Access-Control-Allow-Headers", "User-Agent, Referer, Content-Type, Range, Content-Length, Authorization, Accept,X-Requested-With");
	res.header("Access-Control-Allow-Methods", "PUT,POST,GET,DELETE,OPTIONS");
	res.header("Access-Control-Max-Age", 24 * 60 * 60);//预检有效期
	res.header("Access-Control-Expose-Headers", "Content-Range");
	res.header("Access-Control-Allow-Credentials", true);
	if (req.method == "OPTIONS") return res.sendStatus(200);
...
    // 给每一个请求的res对象定义了一个格式化函数备用
	res.fmt = function (data) {		//格式化返回结果
		...
		return res.json(resData);
	};
	// 进入后续中间件处理
	next();
});
...
// 输出app在最外层server.js中继续进行包装。
module.exports = app;
...
```

#### 流程
举2类例子：接口；渲染页面。
1.接口类以请求文章列表为例。
如某个ajax请求为`pathto/api/article/list`。
根据server.js第35行`app.use(client_config.ROOT_ROUTER + '/api', cms_router); `，被cms_router路由处理，其中cms_router为*`const cms_router = require('./server/routes_cmsapi');`*，
cms_router路由中第74行`router.get('/article/list', auth_validate, CmsArtileApiCtrl.articleList);` 是对文章列表的处理。
其中 `CmsArtileApiCtrl`是`const CmsArtileApiCtrl = require('./controllers/cms/article');`，是对`ArticleModel`的操作的包装，所以放在controllers中，从数据库获取完数据后对接口进行返回：
```
...
}).then(function (result) {
	return res.fmt({
		data: {
			listData: result,
			count: count,
			pageNo: pageNo,
			pageSize: pageSize
		}
	});
})
...
```

而`ArticleModel`是`const ArticleModel = require('../../models').ArticleModel;`，是用`sequelize`定义的数据库的用于文章的表。

2.渲染页面类以某篇文章页面为例。
如在浏览器地址栏请求为`pathto/front/view/article/34`回车。
根据server.js第37行`app.use(client_config.ROOT_ROUTER + '/front', front_router); `，被front_router路由处理，其中front_router为*`const front_router = require('./server/routes-front');`*，

front_router路由中第23行`router.get('/view/article/:id', FrontArtileApiCtrl.frontViewArticle);`是对文章的处理。
其中`FrontArtileApiCtrl`是`const FrontArtileApiCtrl = require('./controllers/front/article');`，是另外一个对`ArticleModel`的操作的包装，所以放在controllers中，是在构造函数的方法`frontViewArticle`中把对应id的数据从数据库获取完数据后套ejs模板并进行redis缓存后返回浏览器（如果redis中存在的话则直接返回已有的文章）。

而`ArticleModel`跟上面第一类例子相同是`const ArticleModel = require('../../models').ArticleModel;`，是同一个用`sequelize`定义的数据库的用于文章的表。

#### 路由配置 cms_router
对不同的接口请求配置不同的路由处理
```
...
const express = require('express');
...
const router = express.Router();
...
const CmsArtileApiCtrl = require('./controllers/cms/article');
...
//获取文章
router.get('/article', auth_validate, CmsArtileApiCtrl.article); 
//获取文章列表
router.get('/article/list', auth_validate, CmsArtileApiCtrl.articleList); 	
//创建文章
router.post('/article/create', auth_validate, CmsArtileApiCtrl.articleCreate); 	
//删除文章
router.post('/article/delete', auth_validate, CmsArtileApiCtrl.articleDelete); 
//编辑文章
router.post('/article/edit', auth_validate, CmsArtileApiCtrl.articleEdit); 	    	
...

module.exports = router;
```
#### 包装数据库操作 CmsArtileApiCtrl
CmsArtileApiCtrl是一个构造函数，它的方法其实是作为router的中间件处理函数，进行的是数据库操作。
```
...
// 引用数据表实例
const ArticleModel = require('../../models').ArticleModel;
...
const redisArticleUpdate = require('../../helpers/ArtAndActCacheUpdate').redisArticleUpdate;
// 定义一个构造函数，它的方法其实是作为router的中间件处理函数
function Article() {
}
// 查询文章列表并返回 注意req, res, next
Article.prototype.articleList = function (req, res, next) {
	let where = {status: {'$ne': -1}};
    ...
	let count = 0;
	Promise.all([
	    // 进行真正的数据库操作
		ArticleModel.findAll(pagination_query),
		ArticleModel.count(count_query)
	]).then(function (result) {
		...
	}).then(function (result) {
		return res.fmt({
			data: {
				listData: result,
				count: count,
				pageNo: pageNo,
				pageSize: pageSize
			}
		});
	})
}
// 查询某篇文章并返回
Article.prototype.article = function (req, res, next) {
	let where = {id: req.query.id || 0};

	let query = {
		where: where
	}

	ArticleModel.findOne(query).then(function (result) {
		...
	}).then(function (result) {
		return res.fmt({
			status: 0,
			data: result
		});
	})
}

Article.prototype.articleCreate = function (req, res, next) {
	...
}
Article.prototype.articleEdit = function (req, res, next) {
	...
}
Article.prototype.articleDelete = function (req, res, next) {
	...
}
...
// 返回构造函数的实例
module.exports = new Article();

```

#### 数据表 ArticleModel
依然是以本笔记中的文章的模型为例。
下面是使用[sequelize](http://docs.sequelizejs.com/)建立的数据表的定义。定义了title, content等字段。
path: `/server/models/article.js`
```
// 小写 sequelize 的为我们创建的数据库实例，大写的 Sequelize 为使用的sequelize库模块
const Sequelize = require('../helpers/mysql').Sequelize;
const sequelize = require('../helpers/mysql').sequelize;
// 文章表
var CmsArticle = sequelize.define('cms_article', {
	title: {
		type: Sequelize.TEXT,
		allowNull: false,
		comment: '标题'
	},
	content: {
		type: Sequelize.TEXT,
		allowNull: false,
		comment: '内容'
	},
	...
}, {
	'createdAt': false,
	'updatedAt': false
});

// CmsArticle.sync({force: true})
module.exports = CmsArticle;
```

path: `/server/helpers/mysql.js`
```
'use strict';

const Sequelize = require('sequelize');
const mysql_config = require('../config/env').MYSQL[0];

const sequelize = new Sequelize(mysql_config.DATABASE, mysql_config.USER, mysql_config.PASSWORD, {
	host: mysql_config.HOST,
	dialect: 'mysql', //数据库类型
	pool: {
		max: 5,
		min: 0,
		idle: 10000
	},
	logging: false
});
// 小写 sequelize 的为我们创建的数据库实例，大写的 Sequelize 为使用的sequelize库模块
exports.sequelize = sequelize;
exports.Sequelize = Sequelize;
```

#### ejs模板
path: `/server/views/article.html`
```
<!DOCTYPE html>
<html>
  <head>
   ...
    <title><%= navigator_title %></title>
    
    <style>
      <% include ./common/css-article.html %>
      .wrapper {background: #fff;}
      ...
    </style>
    ...
  </head>
  <body>
    <div class="wrapper">
      ...
        <div><%- content %></div>
    ...
  </body>
</html>

```
#### redis处理
使用的[ioredis](https://github.com/luin/ioredis)
path: `server/helpers/redis.js`

###完整layout
```
| - client          前端
  | - build           配置文件
    | - build.js              build时node调用的js
    | - config.js             一些公用配置
    | - webpack.base.conf.js  webpack基本配置
    | - webpack.dev.conf.js   webpack开发配置
    | - webpack.prod.conf.js  webpack生产配置
  | - config
    | - dev.env.js            开发环境NODE_ENV
    | - index.js              config模块的对外输出总接口
    | - prod.env.js           生产环境NODE_ENV
  | - mock            模拟数据
    | - article               文章相关
      | - list.do.js            文章列表的假数据
  | - src
    | - api                   定义请求接口
      | - api.js                定义接口的url（开发、生产）
      | - article.js            定义文章相关如何调用接口
      | - request.js            包装请求函数
    | - assets                静态资源
      | - loading.gif          
    | - components            存放定义的vue组件
      | - Editor.vue            富文本编辑组件
      | - Layout.vue            布局组件
      | - Search.vue            搜索框组件
      | - Editor.vue            富文本编辑组件
    | - helpers               存放公用js函数模块
    | - style                 存放公用scss模块（如mixin）
      | - _mixin.scss
    | - views                 存放路由页面
      | - article               文章相关页面
        | - editor.vue            编辑文章
        | - index.vue             router-view模板
        | - list.vue              文章列表
        | - preview.vue           预览文章
      | - index.vue             根页面
      | - login.vue             登录页面
      | - noauth.vue            无权限提示页面
      | - notfound.vue          404页面
    | - App.vue               根路由模板
    | - auth.js               权限管理模块
    | - config.js             开发配置
    | - main.js               vue入口js
    | - nav-config.js         导航栏路由配置
    | - router-config.js      路由配置
  | - index.html        vue项目的入口html文件
| - deploy            部署相关
| - dist              打包目录，也是服务器的静态资源路径
| - logs              自动生成的日志
| - node_modules      
| - scripts           node脚本
  | - job.js            生成字体、负责推送
| - server            后端服务器
  | - config            配置
    | - env               配置
      | - development.js    开发配置
      | - index.js          配置入口
      | - production.js     生产配置
    | - code.js           定义错误码
  | - controllers       处理业务逻辑
    | - cms               cms类
      | - article.js        文章处理接口，利用model操作数据库
      | - auth.js           权限过滤
    | - front             front类
      | - article.js         
  | - helpers           工具方法
    | - logger.js         日志处理
    | - mysql.js          mysql客户端
    | - permCache.js      权限相关
    | - redis.js          redis
    | - redisCacheUpdate.js redis
    | - request.js        包装请求
    | - upload.js         七牛上传
    | - userinfo.js       用户信息
    | - webfont.js        webfont
  | - models            数据库结构
    | - article.js        文章表
    | - index.js          入口
    | - profile.js        用户数据
  | - views             后端render的页面
    | - article.html      文章页ejs模板
    | - list.html         列表页ejs模板
  | - index.js          服务入口文件
  | - routes-front.js   路由：非cms
  | - routes_cmsapi.js  路由：cms
| - static            client静态资源
| - .bablerc          bablerc
| - .docerignore      docerignore
| - .editorconfig     editorconfig
| - .eslintignore     eslintignore
| - .eslintrc.js      eslintrc
| - .gitignore        gitignore
| - package.json      工程文件
| - readme.md         readme
| - server.js         开发服务器
```


[^fn1]: 并没有使用webpack的dev-server，而是自己提供了一套server开发环境，另外提供了一套后端服务的server，同时提供了一套前端的vue开发成果

[^fn2]: If the current middleware function does not end the request-response cycle, it must call next() to pass control to the next middleware function. Otherwise, the request will be left hanging.
To skip the rest of the middleware functions from a router middleware stack, call next('route') to pass control to the next route. NOTE: next('route') will work only in middleware functions that were loaded by using the app.METHOD() or router.METHOD() functions.

[^fn3]: https://juejin.im/entry/574f95922e958a00683e402d
http://acgtofe.com/posts/2016/02/full-live-reload-for-express-with-webpack
https://github.com/bripkens/connect-history-api-fallback
https://github.com/nuysoft/Mock/wiki
http://fontawesome.io/icons/
