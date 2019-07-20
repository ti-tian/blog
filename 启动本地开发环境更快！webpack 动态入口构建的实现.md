# 启动本地开发环境更快！webpack 动态入口构建的实现

## Overview

### 酷家乐主站前端开发方式介绍

整个酷家乐主站有 100+ 页面，而且所有的页面都在同一个业务 repo 里，并且按照所属业务的不同划分了不同的页面目录，比如户型库、酷家乐大学等等，我们有着统一的 def-cli 命令行工具，它提供了工程开发各个生命周期（dev、build、deploy 等）的支持，比如在命令行中执行 kjl dev (启动本地开发) 就可以构建 repo 内所有的页面的资源（js、css），同时启动本地开发服务器代理线上请求。
​

### 名词解释

1. webpack：一种前端资源打包工具（滑稽
2. 动态入口打包：在酷家乐主站这种多页面开发的项目中，启动 kjl dev 时不构建任何资源，按照具体访问的前端页面再构建对应的资源，使得每次进入新的页面的时候，再将对应的 js 动态的添加到 webpack entry 中。
   ​

## 为什么要做动态入口打包

### 目前的痛点

对于目前 100+ 的页面来说，每次启动 kjl dev 的时候将所有页面资源全部构建已经不现实，全部构建不仅没有意义而且打包时间巨慢无比，还会存在内存不足的情况。
​

### 之前的解决方案

通过 kjl dev -p xxx 指定命名空间的方式，只对将要开发的页面打包，确实 3 使得 kjl dev 启动的时间大大减少，特别是借助 webpack 4 后，打包整个酷家乐大学的 22 个页面只需 15s，但是依旧不够快!
​
而且主站的业务开发每次可能涉及到多个页面的同时开发，所以使用 -p 指定单一的页面每次切换会比较繁琐。
​

### 动态入口打包的好处

使用动态入口打包后，启动开发服务器只需要 0.5s，需要等待的时间就变成了具体页面资源构建的时间，而且由于 cache-loader 的存在，实际上这些时间可以忽略不计。
​

## 原理介绍

### webpack 中的 entry 处理流程

要了解如何实现 webpack 动态入口打包，要先了解在 webpack 构建流程中关于处理 entry 的部分，所以下面先介绍一下这部分的原理：
​
在 webpack 主函数调用的时候，有 3 个步骤比较重要：

1.  通过 new Compiler 生成一个 compiler 实例，compiler 代表了完整的配置的 webpack 环境。
2.  然后通过 webpackOptionsDefaulter 这个类为传入的参数 options 赋值一些默认值，并且返回新的 options，新的 options 包含了所有 webpack 后续流程里需要的参数。
3.  将 compiler 和处理过后的 options 同时传入 WebpackOptionsApply 这个类中，注册 webpack 打包流程中要使用的一系列插件（plugin），webpack 的打包过程就是依赖类发布订阅的方式按照流程调用这些 plugin。
    ​
    此处要有流程图：
    ![image2018-7-3-11_56_39](/content/images/2018/07/image2018-7-3-11_56_39.png)
    要实现动态入口打包，我们必须要在 webpack 处理 entry 的时候动一些手脚，所以就要找到 webpack 打包流程中处理 entry 的插件，在 WebpackOptionsApply 这个类的内部可以找到实际处理 entry 的插件：EntryOptionPlugin，在 EntryOptionPlugin 的内部注册了一个事件 (compiler.hooks.entryOption) 后又立即触发了这个事件。
    ​
    ![image2018-7-3-11_31_16](/content/images/2018/07/image2018-7-3-11_31_16.png)
    ​
    EntryOptionPlugin 内部注册的 compiler.hooks.entryOption 事件非常简洁，它的作用是判断 webpack options 中输入的 entry 是什么类型，然后根据不同的类型使用不同的 entryPlugin 处理。
    ​
    ![------_15312136819106](/content/images/2018/07/------_15312136819106.png)
    ​
    以日常多页面开发为例，假设开发时候输入的 entry 为以下形式

```javascript
​
{ 'activity-design':
   [ 'webpack-hot-middleware/client?noInfo=true&reload=true&name=activity-design',
     'E:\kjl-site\pages-activity\design\entry.webpack.js' ],
  'college-homework':
   [ 'webpack-hot-middleware/client?noInfo=true&reload=true&name=activity-hao-index',
     'E:\kjl-site\pages-college\homework\entry.webpack.js' ],
  'college-video':
   [ 'webpack-hot-middleware/client?noInfo=true&reload=true&name=shop-join-brandcenter',
     'E:\kjl-site\pages-college\video\entry.webpack.js' ] }
```

​
显然是一个 object 类型，所以处理 entry 的方法就是：首先遍历每个 entry ，然后将 entry 和 name 给 itemToPlugin 方法，并且由于每个二级 entry 都是一个数组，所以最终处理 entry 的 plugin 是 MultiEntryPlugin。
​
最终处理 entry 的代码实际等同为:

```
​
const MultiEntryPlugin = require("./MultiEntryPlugin");
​
const entry = { 'activity-design':
   [ 'webpack-hot-middleware/client?noInfo=true&reload=true&name=activity-design',
     'E:\kjl-site\pages-activity\design\entry.webpack.js' ],
  'college-homework':
   [ 'webpack-hot-middleware/client?noInfo=true&reload=true&name=activity-hao-index',
     'E:\other\kjl-site\pages-college\homework\entry.webpack.js' ],
  'college-video':
   [ 'webpack-hot-middleware/client?noInfo=true&reload=true&name=shop-join-brandcenter',
     'E:\kjl-site\pages-college\video\entry.webpack.js' ] }
​
for (const name of Object.keys(entry)) {
    new MultiEntryPlugin(context, entry[name], name).apply(compiler);
}
​
```

再看 MultiEntryPlugin 内部的调用，抛开其他不谈，其中关键的一段代码是：
​
![image2018-7-3-12_16_45](/content/images/2018/07/image2018-7-3-12_16_45.png)
​
使用了 compiler.hooks.make.tapAsync，这个钩子函数表示为在 compiler 执行 make 事件的时候调用监听的回调函数，而在回调函数中会调用 compilation.addEntry 方法。
​
前面提到过 compiler，它代表了完整的配置的 webpack 环境，而这里的 compilation 又表示当前一次资源构建的环境，所以把 entry 交给了它，一旦 webpack 开始执行构建，就会调用 compilation.addEntry 方法，它去解析 entry 然后执行后续构建，本文就不再叙述后续的打包流程，有兴趣的同学可以参考网上的文章。
​
既然知道了 webpack 对于 entry 的处理过程，我们如何实现 webpack 的动态打包呢？这就必须借助 webpack-dev-middleware。
​

### webpack-dev-middleware

​
[webpack-dev-middleware](https://github.com/webpack/webpack-dev-middleware) 是一个非常广泛的用作静态资源服务器的中间件，它可以将 webpack 构建好的资源存存放到内存中，然后根据一次 http 请求的 req.url 找到被请求的资源名称，然后从内存中取得资源内容，将资源内容发送出去。
​
下面来介绍一下 webpack-dev-middleware 的调用方式和原理:
​

```
const webpackDevMiddleware = require("webpack-dev-middleware")
const webpackConfig = require('./webpack.config.js')
​
//  获取 express 实例
const app = express(),

// 获取 compiler 对象
const complier = webpack(webpackConfig)
​
// 传入 compiler 和 options
// 然后在 express 的实例 中 use 这个中间件。
app.use(webpackDevMiddleware(complier, {
    publicPath: webpackConfig.output.publicPath,
    ... // 其他配置参数
}))
```

webpack-dev-middleware 提供了两种 webpack 的运行方式，一种是默认的 watch 模式，这种模式会调用 compiler.watch 然后利用 webpack 的 watch 监听文件的改动、另外一种方式是每次中间件捕获到请求的时候再去执行 webpack 的构建，调用 compiler.run 方法。
​
其大致流程如下：
![image2018-7-3-14_36_5](/content/images/2018/07/image2018-7-3-14_36_5.png)
​
webpack-dev-middleware 的作用跟我们要实现的动态入口打包的功能非常相似，所以我们可以借助它来实现我们想要的功能。
​

### 最终方案

借助 webpack-dev-middleware 能解决根据不同请求获取不同的资源的问题，但是依然存在另外一个问题，那就是每次将生成 compiler 的时候，都已经在 webpack 的主函数流程中处理过 webpack options 中的 entry 了，这使得它们已经被通过事件监听的方式加入到了本次构建需要的 entry 中了，一旦开始执行构建（调用 compiler.run 方法），就会将这些 entry 全部构建。
​
这显然不是我们希望的在启动服务的时候不构建任何资源，只在请求到了具体资源的时候再构建。
​
所以我们必须屏蔽掉 EntryOptionPlugin 处理 entry 这一过程，而如何屏蔽这一过程， webpack 并没有向用户提供 api，只能通过某些取巧的方法去做。
​
这个方法就是利用 webpack 的 Tapable 的机制，前文提到过：在 WebpackOptionsApply 这个类的内部可以找到实际处理 entry 的插件：EntryOptionPlugin，在 EntryOptionPlugin 的内部注册了一个事件 (compiler.hooks.entryOption) 后又立即触发了这个事件。
​
而 webpack 使用一个插件的方式是通过 Tapable 事件机制，比如 EntryOptionPlugin 这个插件在 webpack 事件流中注册和触发的方式是 ：

```
// 注册事件
compiler.hooks.entryOption.tap("EntryOptionPlugin, ( context, entry)=>{
    //
})

// 触发事件
compiler.hooks.entryOption.call(options.context, options.entry);
```

​
掘金上有一篇关于 [Tapable](https://juejin.im/post/5abf33f16fb9a028e46ec352) 的文章，介绍了 webpack 的事件机制，有兴趣的同学可以去看看。
​
通过上面的代码可以看到 EntryOptionPlugin 这个事件注册在 compiler.hooks.entryOption 这个钩子上，而这个钩子在 Compiler 这个类的内部的形成方式是：

```
const {
    Tapable,
    SyncBailHook,
    ...
} = require("tapable");
​
class Compiler extends Tapable {
    constructor(context) {
        super();
        this.hooks = {
            // 省略
            entryOption: new SyncBailHook(["context", "entry"])
        }
    }
}
```

SyncBailHook 是 Tapable 钩子函数中的一种，它的使用方式是 **【只要监听函数中有一个函数的返回值不为 null，则跳过剩下所有的逻辑】**。同时 webpack 构建流程中会先调用开发者在 webpack options 中注入的外部插件（plugin）再去调用它本身内部的插件。
​
所以我们在 webpack 内部的这个的事件钩子函数调用之前，可以先在外部插件中触发了这个事件并且返回一个 true，这样内部 EntryOptionPlugin 中监听的事件就不会触发了，webpack 也就不会拿到 options 中输入的 entry。
​
接着重写 webpack-dev-middleware 中的部分逻辑，使得在每个请求的第一次进到 middleware 的时候，利用 EntryOptionPlugin 里的处理 entry 的方式将要构建的 entry 加入到 webpack 的本次构建中，而且在所有请求的第一个请求进来的时候，生成一个 webpack watching 对象，把后续的文件改动交给 watching 去监听，这就实现了动态打包！
​
整个过程如下：
​ 1.首先是在 webpack config 中的 plugins 中预先加入一个处理 compiler.hooks.entryOption 的插件

```
const webpackConfig = require('./webpack.config.js')
​
// 在 webpackConfig plugins 中加入一个 plugin
class DynamicDevPlugin {
    apply(compiler) {
        compiler.hooks.entryOption.tap('dynamicDevPlugin', () => {
            // 仅仅返回 true 就可以 目的是屏蔽掉 webpack 本身对于 entry 的处理
            return true;
        });
    }
};


webpackConfig.plugins.push(new DynamicDevPlugin());

const compiler = webpack(webpackConfig)
​
```

2. 重写 webpack-dev-middleware 中的部分逻辑

```
... // 省略一些变量的获取和声明
function middleware(req,res,next){

    // 通过 req.url 和 options.publicPath 获取资源名称 借助 webpack-dev-middleware 中的方法

    let filename = getFilenameFromUrl(
            publicPath,
            compiler,
            req.url
    );

    // 获取 webpackConfig 中的 entry
    const webpackEntry = compiler.options.entry

    // 使用 webpack 内部插件 entryOptionPlugin 中的逻辑

    const SingleEntryPlugin = require('webpack/lib/SingleEntryPlugin');
    const MultiEntryPlugin = require('webpack/lib/MultiEntryPlugin');

    // 表示是否需要重新构建
    let rebuild；

    // 将要构建的资源路径和名称加入到本次构建的 compilation 中

    const itemToPlugin = (context, item, name) => {
        if (Array.isArray(item)) {
            return new MultiEntryPlugin(context, item, name);
        }
        return new SingleEntryPlugin(context, item, name);
    };

    // 遍历 webpackEntry ，并且匹配 filename 与  entry 中对应的资源路径，然后利用 itemToPlugin 处理
    Object.keys(webpackEntry).forEach(item => {
         const entry = webpackEntry[item];
            if (filename.includes(item)) {
               if (!fs.existsSync(filename)) {
                   rebuild = true;
                   itemToPlugin(
                      compiler.options.context,
                      entry,
                      item
                 ).apply(compiler);
             }
         }
     });

    // 如果是第一次进来 生成 watching 对象表示 用 webpack watch 进行文件监听
    let watching;
    if (!watching) {
       watching = compiler.watch(options.watchOptions, err => {
         ...
       })
    }

    // 如果需要重新构建资源 就执行 watching.invalidate();
    if (rebuild) {
        watching.invalidate();
        // 在构建结束的钩子（compiler.done）中将构建好的资源内容发送出去
        ...
      } else {
        // 直接将构建好的资源内容发送出去
        ...
     }

     ...
}
```

## 总结

以上就是整个动态入口打包的原理，目前已经在酷家乐主站前端工程化套件中使用了这一功能，不过自我感觉处理 entry 的那部分操作不是很科学，但是又没有发现更好的方式，希望能听听大家的意见。
​
