# 巧用 webpack 做页面静态资源依赖分析

## 前言：

所谓【静态资源依赖分析】，指的是可以通过分析页面资源后，可以以 json 数据或者图表的方式拿到页面资源间的依赖关系。

比如 college-index（酷家乐大学首页）的入口文件 entry.js 引用了 banner.js、 同时 banner.js 又引用了 utils.js， 那么我们希望经过分析后能拿到一份这样的数据：

```j
[
    {
        "type": "entry",
        "path": "/xx/xx/college-index/entry.js",
        "deps": [
            {
                "type": "module",
                "path": "/xx/xx/college-index/banner.js",
                "deps": [
                    {
                        "type": "module",
                        "path": "/xx/xx/college-index/utils.js",
                        "deps": []
                    }
                ]
            }
        ]
    }
]
// type 分表表示它是一个 entry 还是一个 module
```

拿到资源依赖文件之后可以做什么呢？笔者这里有几个利用场景可供参考：

- 对一个多页面 repo 而言，每次要发布的时候，我希望通过 git diff 拿到本次改动的文件，再通过依赖分析拿到此次需要构建的资源，这样就可以做到单页面发布了。
- 我可以拿到当前的资源依赖，为我剔除 repo 中没有用到的资源
- 我希望在 vscode 扩展中实时预览前端 repo 中的资源依赖情况

用处还可能有很多，关键是如何快速拿到这份依赖分析数据？

## 一个思路

这里给出一个笔者曾经考虑过的思路：通过遍历页面入口，然后进行关键字匹配，比如对（ 【import xx from xxx】、【require】) 等关键字做处理，拿到被依赖的模块路径，然后继续对模块路径做递归解析，最终汇总拿到依赖树。

这个思路乍看是可行的，而且使用一些措施会使得分析流程更加高效，比如再对关键词作匹配的时候可以借助 acorn 这类的 JavaScript 解析器，再通过对文件被解析后的 ast 作处理。

但是简单尝试后就放弃了这个思路，原因是对于现在的前端工程化项目而言，一个页面中的依赖不仅有 js 而且还会有各种各样的资源，比如 sass、less 之类的 css 预处理器、或者是别的资源等等，所以单单对 js 路径做处理是不够的。

## 借助 webpack 来实现

在上述思路不可行的情况下，我们将解决办法瞄向了借助 webpack 来实现，对 webpack 熟悉的开发者会知道 webpack 对于依赖对处理的过程，这里再简单的提一下：webpack 拿到入口文件 entry 后，会通过先获取资源的正确路径，再经过 loader 解析文件，最后通过遍历 ast 来拿到模块中引用的依赖 【dependences 】，再对 【dependences】 做递归处理，最终拿到依赖树。

这跟我们最初设想的思路基本一致，同时借助 loader 可以将不同的资源无法解析的问题也一并解决了。看到这里的人或许会有疑问：官方不是已经给出了 [webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer) 这类的工具了吗？ 而且每次构建后 stats 中都能拿到文件依赖，为啥不能直接使用呢。

不直接使用的原因很简单：首先构建一次实在太慢了，特别是有几十个页面存在的情况下，另一个原因是我只是想拿到资源依赖，我根本不想对整个前端 repo 进行一次构建，也不想生成任何 bundle。

有没有一种工具可以如同 webpack 一样，既可以使用 loader 找到文件依赖，又不需要生成和压缩 bundle 呢？

在我们改造 webpack 之前它本来是没有的，如何改造？ 一个 webpack plugin 即可。

## webpack 的构建流程

在介绍如何改造之前，有必要了解一下 webpack 的模块处理过程以及整体流程。

### 模块的处理过程：

webpack 拿到一个路径后，他会依次执行 reslove 模块路径 -> create 模块 -> build 模块 -> parse 模块等主要流程，每一步的流程的作用如下：

-【 reslove 模块 】：获取模块真实路径 -【 create 模块 】 ：创建一个模块的 context -【 build 模块 】 ：读取模块内容 -【 parse 模块 】 ：分析模块内容（主要是找到模块中的 require 关键词，将依赖添加到该模块的依赖数组中。

最后重复上述流程

每一个流程都非常复杂，以【 reslove 模块】 这个流程为例：
处理逻辑主要在 `webpack/lib/normalModuleFactory.js` 这个文件中， 整个 reslove 流程会依次执行 `beforeResolve`、 `factory`、 `resolver` 以及 `afterResolve` 这些步骤，每个步骤对应一个 hook ，每个 hook 执行的时候会拿到关于模块的描述信息，描述信息会随着 hook 的执行愈发饱满，最终获取到了完整的模块信息，为接下来的【 create 模块】以及 【 build 模块】做好准备，在下文【具体实现】中我们会插手这部分流程。

本文不再细谈模块的处理流程，网上有许多优秀的文章可以参考，请大家自由查阅。

### webpack 的整体流程：

在笔者看来，webpack 的整体流程分为 4 个步骤，图示如下：

![---------3-](//qhstaticssl.kujiale.com/newt/8/image/png/1558068562067/5704B8FA9497C10253B157DB886C9806.png)

实现依赖分析中我们只需要 webpack 的前 3 个流程就够了，下文会给出原因。

## 解决办法

前面我们提到 webpack 处理依赖的过程是递归分析入口的所有依赖， 由于页面中的依赖包含相对或者绝对路径引用的依赖，也包含借助 alias 引用的依赖，也包含 node_modules 中的依赖，既然我们只想拿到 repo 前的资源依赖，对于 node_modules 中的依赖可以直接屏蔽掉，这将使得模块的递归时间大大缩短，如下图所示：

由：

![-----](//qhstaticssl.kujiale.com/newt/8/image/png/1558066839561/75B19707687AB6B40BF772E3F6B7AEA5.png)

变成：
![-------1-](//qhstaticssl.kujiale.com/newt/8/image/png/1558066859326/59D383DF47B652737921810499E1A5A4.png)

在上面提到的 webpack 4 个主流程中，在 【step3】 结束后，webpack 能拿到所有 modules （一次构建行为产生的所有的模块），此时已经足够我们进行依赖分析了，我们直接终止 webpack 的后续流程，不再进行生成 chunk 以及对 chunk 做的合并优化等过程。这就达到了本文题目中目的，【用十分之一的构建时间做一场页面静态资源依赖分析】。

## 具体实现：

### 写一个 webpack plugin

plugin 名字就叫 FastDependenciesAnalyzerPlugin ，plugin 写法参照官方[文档](https://www.webpackjs.com/concepts/plugins/)。

```
class FastDependenciesAnalyzerPlugin {
  beforeResolve = (resolveData, callback) => {}

  afterResolve = (result, callback) => {}

  handleFinishModules = (modules, callback) => {}

  apply(compiler) {
    compiler.hooks.normalModuleFactory.tap(
      "FastDependenciesAnalyzerPlugin",
      nmf => {

        nmf.hooks.beforeResolve.tapAsync(
          "FastDependenciesAnalyzerPlugin",
          this.beforeResolve
        );

        nmf.hooks.afterResolve.tapAsync(
          "FastDependenciesAnalyzerPlugin",
          this.afterResolve
        );
      }
    );

    compiler.hooks.compilation.tap(
      "FastDependenciesAnalyzerPlugin",

      compilation => {
        compilation.hooks.finishModules.tapAsync(
          "FastDependenciesAnalyzerPlugin",
          this.handleFinishModules
        );
      }
    );

  }
}
```

在 `complier.hooks.normalModuleFactory` 这个 hook 的回调中继续监听 `normalModuleFactory` 的 `beforeResolve` hook 和 `beforeResolve` hook。

在 `complier.hooks.compilation` 这个 hooks 的回调中继续监听 `compilation` 的 `finishModules` hook。

### 插手 beforeResolve 流程

```
 beforeResolve(resolveData, callback) {
    const { context, contextInfo, request } = resolveData;
    const { issuer } = contextInfo;
  }
```

`context` 表示为 解析目录的绝对路径，一个页面的 `context` 都是一样的， `issuer` 翻译为发行人，在 webpack 中表示本模块被依赖的对象路径，也就指向这个模块的来源，`request` 表示当前模块的的请求路径。
比如：banner.js 的资源路径为：/xx/xxx/banner.js ，
文件内容是这样的：

```
// 1
import utils from './utils'

// 2
import utils from '@utils'

// 3
import utils from 'utils'
```

所以对于 utils 模块来说， issuer 的值为 /xx/xxx/banner.js， request 的值分别为 "./utils.js"、"@utils"、"utils" 。

此时，我们只能知道当前模块的来源路径 `issuer` 以及它被请求的路径 `request`，拿不到当前模块的真实路径，还无法将它放入我们的依赖树中，所以我们不会在 `beforeReslove` 中处理我们的依赖树，我们在这一步中只是想屏蔽掉一些我们不想被处理的模块就可以，比如假设 `utils` 这个 npm 包里有非常多的小模块，这些模块不会被放到依赖树中，所以对于这些模块我们选择直接跳过， 事实上在 webpack 源码中有这样一句：
![webpack normalModuleFactory 源码](//qhstaticssl.kujiale.com/newt/8/image/png/1558069285136/880FA45FCF60A44ED78595667B0D68DD.png)

对于在 `beforeResolve` 中没有返回值的模块会直接 callback ，在 webpack 源码中 `callback` 里如果没有参数，往往意味着流程的提前结束，在 `beforeResolve` 中 `return callback` 也就没有这个模块后续对 `reslove` 和 `build` 流程了，这就实现了模块跳过。

为此，我们可以实现一个 skip 函数，这里提供一个简单版本，传入参数为 `issuer` 和 `request`

```
// 事先获取到 package.json 里的 dependencies

const ignoreDependenciesArr = Object.keys(dependencies);

function skip(request, issuer) {
  return (
    ignoreDependenciesArr.some(item => request.includes(item)) ||
    issuer.includes("node_modules")
  );
}

```

通过比较 `request` 是否在 package.json 里定义的 dependencies 里，或者如果 `issuer` 本身包含 `node_modules`，就可以表示当前模块是可以跳过的。当然这个方法只是一个简单版本，实际上要考虑许多特殊情况，这里不会详细给出，感兴趣的读者可以自行实现。

### 借助 afterResolve

```
 afterResolve(result, callback) {
    const { resourceResolveData } = result;
    const {
        context:{
           issuer
         },
         path
    } = resourceResolveData;
  }

  // 这里添加依赖到依赖树
```

webpack 使用 [enhanced-resolve](https://www.npmjs.com/package/enhanced-resolve) 对一个请求路径作解析，只需要传入模块的 `contextInfo`, `context`,
`request`, 即可拿到当前模块的真实路径，当然前提是需要告知 `enhanced-resolve` ，这个模块所在 repo 的 package.json， webpack 配置中的 alias 信息等等，本文不会叙述 `enhanced-resolve` 的工作方法，感兴趣的读者可以自行查阅。

总之，在 `afterReslove` 中我们能够拿到 webpack 借用 `enhanced-resolve` 解析过后的模块路径了，还有依赖这个模块的父模块路径，将这两个路径添加到依赖树中，经过简单的递归操作就可以拿到完整的依赖树了，当然在依赖树中我们可以放置各种信息，比如是否是一个模块？是否是一个 js 文件、是否是一个 css 文件，这些都可以实现。

### 在 finishModules 里结束

webpack 官方在 4.30 提交了一次 [commit](https://github.com/webpack/webpack/commit/ea172ec5fdebb77647adde4c8406e9d0b2bbd48c)，在这次提交中将 `finishModules` 这个 `SyncHook` 转化成了 `AsyncSeriesHook` , 同时在 `finishModules` hook 中加入了一行代码，如下图：
![](//qhstaticssl.kujiale.com/newt/8/image/png/1558065748096/EBDFD812D1C534FD68E8BFCAB454CDFE.png)

这使得我们可以监听 `finishModules` 这个 hook ，然后在 err 方法里传入一个值，这就直接屏蔽掉了后续的文件合并以及优化流程了，当然这个操作比较 hack，也不是官方推荐用法，希望在 webpack 5.X 的更新中，官方可以提供更合适的 hook ，让我们方便跳过某些流程。

## 一些踩坑

1. 对于在 js 中引用的 css 或者 scss 文件，可以通过寻常的 reslove 流程拿到依赖，但是如果在 css 中使用了 @import 语法，由于 css-loader 会自行处理这些语法，所以它不会走 webpack 本身的 reslove 流程，详见这个 [issue](https://github.com/webpack-contrib/css-loader/issues/755)，这里得我们自己在 `beforeReslove` 中对这部分做额外对处理，比如通过字符串截取的方式去掉 `request` 中关于 `loder` 描述的部分，再通过主动调用 `enhance-resolve` 这个方法实现对 `@import` 传进来对模块做处理，最终拿到正确的路径。

2. 不同版本的 webpack 一些 hook 的用法和名称会不一样，开发者在处理内部流程的时候要注意。

## 总结

本文所阐述的原理并不深奥，主要是挖掘了一个 webpack 的一个用法，希望能启发想要利用 webpack 做更多工具的开发者多借助 webpack 内部原理，最后感谢大家的阅读，欢迎有兴趣的朋友在文章底下评论，给出建议和帮助。
