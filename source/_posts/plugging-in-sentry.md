---
title: 前端项目接入 sentry 监控系统
---

**写在前面**

业务代码忙碌期暂时告一段落，有空给几个线上的前端项目接了一下异常监控系统。踩了一些坑，做个简单的记录。

## 前端监控系统

一般的前端项目由于直接跑在浏览器端，直面用户，保证项目的稳定性十分重要。稳定性的一个重要评估点即程序要能正确捕获异常、处理异常。前端异常的产生常常是由各种因素导致的，例如服务端错误、网络异常、浏览器兼容性问题、前端程序自身错误等。与服务端项目不同，前端项目运行在各个用户的浏览器中，因此如何记录、追踪异常就显得尤为重要。

## sentry 和 raven

这样普适性的需求，一般都是有造好的轮子的。站在巨人的肩膀上，可以帮助我们更好更快地做很多事情。[sentry](https://sentry.io/welcome/) 就是这样的一个轮子。它的优势在于：

* [开源](https://github.com/getsentry/sentry)
* 部署自由。可以部署在自己的服务器，也可以付费部署在官方的服务器。
* 文档、社区资源相对丰富。

sentry 支持许多语言的异常监控，对于 javascript 而言，可以通过官方的 [raven.js](https://github.com/getsentry/raven-js) 接入 sentry 系统。raven.js 负责在前端搜集和上报错误，sentry 负责接收和处理异常。

## 前端接入 raven

### raven 初始化

文档参考：https://docs.sentry.io/clients/javascript

我们的项目基于 Vue 2.0，使用 webpack 构建。引入方式如下。

```javascript
import Vue from 'vue'
import Raven from 'raven-js'
import RavenVue from 'raven-js/plugins/vue'

const SENTRY_DSN_PUBLIC = ''
Raven.config(SENTRY_DSN_PUBLIC, {
	release: process.env.RELEASE,
	ignoreUrls: [/localhost/i, /127\.0\.0\.1/i, /webpack-internal/i]
})
.addPlugin(RavenVue, Vue)
.install()
```

解释几个地方。
* SENTRY_DSN_PUBLIC 的格式如：``https://[HASH_CODE]@[DOMAIN]//[PROJECT_SERIAL_NUMBER]``。使用时需要把 [PROJECT_SERIAL_NUMBER] 前面的 ``//`` 变成 ``/``。不然会出现请求无法发出的问题。
* ignoreUrls，并不是指所接入项目的域名，而是指 err.stack 中对象的 url 属性值。所以如果需要避免本地开发时产生的错误，并不只是排除本地的域名就可以了。

```javascript
  /*
  * https://github.com/getsentry/raven-js/blob/66b314849c79c780a2447b8a30d7c5a2090705f7/src/raven.js#L588
  */
  
  // null exception name so `Error` isn't prefixed to msg
  ex.name = null;
  var stack = TraceKit.computeStackTrace(ex);
  // stack[0] is `throw new Error(msg)` call itself, we are interested in the frame that was just before that, stack[1]
  var initialCall = isArray(stack.stack) && stack.stack[1];
  
  // if stack[1] is `Raven.captureException`, it means that someone passed a string to it and we redirected that call
  // to be handled by `captureMessage`, thus `initialCall` is the 3rd one, not 2nd
  // initialCall => captureException(string) => captureMessage(string)
  if (initialCall && initialCall.func === 'Raven.captureException') {
  initialCall = stack.stack[2];
  }
  
  var fileurl = (initialCall && initialCall.url) || '';
  
  if (
      !!this._globalOptions.ignoreUrls.test &&
      this._globalOptions.ignoreUrls.test(fileurl)
  ) {
  return;
  }
```

文档中写的就比较晦涩，而且 ignoreUrls 的说明还不完整，需要参考 whitelistUrls。大致意思是，ignoreUrls 和 whitelistUrls 会匹配 js 文件的地址，只有 inline 的 js 文件才会匹配站点的 url。在 webpack-dev-server 的本地调试环境下，js 文件都被写入了内存中，js 文件的地址是 webpack-internal 也不足为奇。

> ignoreUrls
>
> The inverse of whitelistUrls and similar to ignoreErrors, but will ignore errors from whole urls matching a regex pattern or an exact string.{ ignoreUrls: [/graph\.facebook\.com/, 'http://example.com/script2.js'] }
>
> whitelistUrls
>
> The inverse of `ignoreUrls`. Only report errors from whole urls matching a regex pattern or an exact string. `whitelistUrls`should match the url of your actual JavaScript files. It should match the url of your site if and only if you are inlining code inside `` tags. Not setting this value is equivalent to a catch-all and will not filter out any values.

### release 和 sourcemap

在生产环境中的代码，一般都是经过 uglify 的。sentry 在捕获到错误之后，可以根据错误产生的 sourcemap 生成源代码，更方便查找和解决异常。最简单的方式，就是将源文件和其 sourcemap 文件一同发布，只要浏览器能正常解析 sourcemap 文件，sentry 也同样可以。但这样的做法会让外界能轻易获取你的 sourcemap 文件，这样也就能轻易获取你的源代码。对于商业的项目而言，这样的做法并不可取。sentry 也考虑到了这点，所以它支持将 sourcemap 上传到 sentry 的部署服务器并解析，从而不让 sourcemap 文件暴露在外。不过，sentry 必须要有 release 号，才能支持 sourcemap 上传。因此，还必须配置 release 号，才能跑通整个流程。

文档参考：https://docs.sentry.io/clients/javascript/sourcemaps/, https://docs.sentry.io/learn/releases/

* 在编译时生成 sourcemap

```javascript
  /*
  * 在编译时，配置 webpack 的插件
  */
  plugins: [
      new webpack.SourceMapDevToolPlugin({
          filename: 'map/[filebase].map', // 给 sourcemap 文件命名。可以在其中设置路径，这样的路径是 output.path 的相对路径。
          exclude: [/^(.*?)vendor(.*?)\.js$/,/^(.*?)manifest(.*?)\.js$/] // 避免生成一些文件的 sourcemap
  	}),
   
  	new webpack.optimize.UglifyJsPlugin({
      	compress: {
          	warnings: false,
  	    },
      	sourceMap: true // 开启 source map
  	})
  ]  
```

  需要说明的是，sentry 似乎不支持 hidden-source-map 的方式，因此还是需要在 js 文件后进行显示声明 ``//# sourceMappingURL=<URL_TO_SOURCEMAP>``

* 通过 webpack-sentry 插件上传 sourcemap 至 sentry 服务器

```javascript
  /*
  * 在编译时，配置 webpack 的插件
  */
  new SentryCliPlugin({
      release: release, // release 的版本号
      include: <PATH_TO_INCLUDE>, // 上传的目录，一般同 output.path
      configFile: <PATH_TO_SENTRYCONFIG>// sentry的配置文件
  })
```
  sentry 配置文件如下，参考：https://docs.sentry.io/learn/cli/configuration/
```
  [defaults]
  url=<DOMAIN_OF_YOUR_SENTRY_SITE>
  org=<ORGANIZATION_SLUG>
  project=<PROJECT_NAME>
  
  [auth]
  token=<AUTH_TOKEN>
```
  其中，organization_slug 即团队的简称，project 即项目的简称，token 可以在个人设置的 API 选项中找到。

* Release 号的自动化
  一般我们都会希望，随着项目一个版本的发布，程序能自动生成一个能唯一标识的 release 号。这里，我们用git 的 commit 号来进行标识。这种方式比较适合迭代较快的项目。

```javascript
  /*
  * 生成 release 号
  */
  const release = [childProcess.execSync('git rev-parse HEAD').toString().trim().substr(0, 9)]
  const RELEASE = `"${release}"` // 这里用双引号将 release 号包裹起来，即为一个带双引号的字符串
  
  // 在 SentryCliPlugin 中，获取 release 号，详见往上两块的代码
  
  // 在编译时，注入到 process.env 中
  let env = {}
  env.RELEASE = RELEASE
  new webpack.DefinePlugin({
      'process.env': env
  }),
  
  // 在 raven 中注入版本号
  Raven.config(SENTRY_DSN_PUBLIC, {
  	release: process.env.RELEASE, // 这里会将 RELEASE 转义，去除双引号
  	ignoreUrls: [/localhost/i, /127\.0\.0\.1/i, /webpack-internal/i]
  })
```

* 在发布前，删除 sourcemap 文件。这样 sourcemap 就不会暴露了。

```javascript
  var shell = require('shelljs')
  var webpack = require('webpack')
  var webpackConfig = require('./webpackconfig')
  
  webpack(webpackConfig, (err,stats) => {
      if(err){
  	  // 这里如果有要终止程序的需求，需要调用 process.exit()
        // 详见 https://github.com/getsentry/sentry-webpack-plugin/issues/54
      }
      shell.rm('-rf', <PATTH_TO_SOURCEMAP_DIR>)
  })
  
```

## 服务端转发异常消息

理想的异常监控模式是，当 sentry 收到一条异常消息后，能通过某种方式通知到开发者。除了传统的邮件模式之外，sentry 还提供了对各种系统的集成 (https://docs.sentry.io/integrations) 。

因为团队主要采用钉钉进行通信，我们希望借助钉钉 webhook 机器人的功能进行 sentry 的消息转发。由于 sentry 原生的 webhook 并不直接支持钉钉的 webhook API，所以需要搭建一个消息转发服务桥接 sentry 和钉钉的 webhook。这里主要看一下 senty webhook 消息的结构。我们在钉钉中所展示消息基本也就是如下字段。

```javascript
/*
* https://github.com/getsentry/sentry-webhooks
*/

{
  "id": "27379932",
  "project": "project-slug", // project 简称
  "level": "error", // 异常等级
  "url": "https://app.getsentry.com/getsentry/project-slug/group/27379932/", // sentry 中错误的 URL
  "message": "This is an example Python exception", // 异常消息
  "event": {
    "extra": {},
    "sentry.interfaces.User": { // 用户信息，可以通过 Raven.setUserContext 进行自定义
    },
    "sentry.interfaces.Http": {
      "fragment" : "" // 错误发生的 URL，便于定位错误产生的页面
    }
  }
}
```

sentry 的 webhook 可以在 ``项目设置 > 所有集成`` 中进行开启，并配置 webhook callbackURL。另外，你也不需要担心 sentry 是否会频繁报警，可以在 ``项目设置 > 警报`` 中设置其发出消息的频率、规则等，非常强大。更多参考：https://docs.sentry.io/learn/notifications/#alerts

## 部署 Sentry 至私有服务器

将 sentry 部署到私有服务器上的最大好处就是免费。sentry 可以通过 docker 和 python 两种方式部署到服务器上。当然，最好给你的 sentry 服务配置一个 https 的域名，因为**好像**直接通过 ip 的方式并不能成功的上报异常。详细文档请查阅：https://docs.sentry.io/server/installation 。这里不再赘述。

## 小结

前端监控系统接入以来，发现了许多不易发现或不小心产生的异常，如 ``Unhandled Promise Rejection``，``Network Error: HTTP No Response``，``Undefined is not a/an Function/Object ``。有针对性地解决这些问题，对于整个前端业务的稳定、用户体验的提升、项目经验的积累都十分有益。