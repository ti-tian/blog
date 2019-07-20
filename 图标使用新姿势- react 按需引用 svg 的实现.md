# 图标使用新姿势- react 按需引用 svg 的实现

## 前言

图标是前端在业务开发中不得不写的一个东西，以我司(酷家乐)的几个部门为例，每个组在写图标上都有不一样的方式：
​

- 组 1：单色图标用 iconfont 上提供的字体文件，彩色图标用 img 引入代替或者使用 iconfont 上提供的 symbol.js 。
- 组 2：引入 svg 文件，通过 [react-svg-loader](https://www.npmjs.com/package/react-svg-loader) 将其包裹成一个 react 组件使用。
- 组 3：引入 svg 文件，通过 [svg-sprite-loader](https://www.npmjs.com/package/svg-sprite-loader) 将所有 svg 图标处理成 svg 雪碧图的方式使用。
  ​
  这几种使用方式各有千秋，下面谈一谈他们的优缺点：）
  ​
  组 1 的使用方式【简单】，不需要手动引入每个 svg 文件，缺点是字体图标不如 svg 文件【可扩展性好】，同时为了引入一个图标引入一个完整的字体图标也会带来一定冗余。
  ​
  而其他两个组的问题在于【图标的引入】以及【管理】方面，需要手动引入 svg 文件，当然优点也非常可观。
  ​
  这里明确一个事实：svg 图标的综合表现是远大于字体图标的，从 antd 从 3.9.0 的更新就可以看出来。
  ​
  > 摘自官方文档
  > ![antd 3.9.0](//qhstaticssl.kujiale.com/newt/8/image/png/1559810086702/90DA8B81359CA46CC38D5527A5CB803D.png)
  > ​
  > antd 的图标使用体验一直很好，比如下面的代码就可以定义一个 `home` 图标
  > ​

```javascript
<Icon type='home' />
```

​
不需要事先引入任何资源 ，只需要指定 `type = "home"` 就可以使用。
但是 antd 没有解决一个问题，那就是如何做到图标的按需引用?
​

> 摘自官方文档
> ![](//qhstaticssl.kujiale.com/newt/8/image/png/1559810582423/1EE46DB441D6F6C785B859FF28EB0301.png)
> ​
> 即便是这里提到的 [webpack 插件](https://github.com/Beven91/webpack-ant-icon-loader) 也不过是图标改成了后置引入，并没有解决图标的按需引用问题。
> ​
> 当然 antd 不好优雅的这个问题是由它的使用方式决定的（合理猜测），作为一个流行的组件库，antd 在引入新的技术的同时又要照顾之前使用者的使用体验，不可避免的会出现一些瑕疵。这是可以理解的，不过换成我们普通业务开发而言，我们没有必要去追求太过完美的开发体验，做出略微的牺牲即可实现【既保持 antd Icon 一样的使用方式，又按需引用了 svg 文件】，怎么实现呢？
> ​

## 如何处理 svg

​
[svg-sprite-loader](https://www.npmjs.com/package/svg-sprite-loader) 是一个在 webpack 中应用比较广泛的 svg 处理库，它可以将代码里引入的 svg 文件合并到一起，然后以 svg symbol 的方式使用，关于它的使用方式网上有大量的文章，所以本文不会再描述它如何使用，请读者自行查阅，
​
值得一提的是，介绍此 loader 的的文章中，一般都会附带如何一次性引入项目中需要的所有 svg 的方法，那就是利用 webpack 的 [require.context](https://webpack.js.org/guides/dependency-management/#requirecontext) api，这个 api 可以获取一个特定的上下文，主要用来实现自动化导入模块，所以为了不再每个模块中一一写 `import 'xxx.svg` 这样的语句，使用这个 api 是有必要的。
​
借助 `require.context` 和 [svg-sprite-loader](https://www.npmjs.com/package/svg-sprite-loader) 能够使图标开发体验上升一个档次，也能配合 React 组件实现类似 antd Icon 的使用方式。
​
但是这种使用方式存在一个缺点，那就是【如何避免引入不必要的 svg】，要知道 `require.context` 可不会区分哪些 svg 是真正需要的，当然对于个人项目而言，我们可以给一个页面固定一个文件夹存在真正需要的 svg 文件，但是对于多页面的 repo 而言，我们无法也没必要给每一个页面都设置一个专门存放该页面需要的 svg 的文件夹。
​
作为一个挑剔的程序员，我需要一种更智能更自动化的方式去引入我真正需要的 svg 图标。
​

## 思路分析

​
现在要解决的问题是我需要在写下类似以下代码的时候:
​

```
<Icon type="close" />
```

​
有种工具能同时在文件中帮我 `import` 一个 `close.svg`。
​
比如下面的代码：

```
import Icon from './Icon.jsx';
​
ReactDOM.render(<Icon type="close"/>);
​
```

​
经过处理后变成这样：
​

```
import Icon from './Icon.jsx';
import './assets/close.svg'
​
ReactDOM.render(<Icon type="close"/>);
```

​
想一想，之前使用过什么工具？会自动帮我们引入我们所需要的代码呢？
​
答案是：[babel-plugin-transform-runtime](https://www.npmjs.com/package/@babel/plugin-transform-runtime)，一个自动帮前端工程师导入 polyfill 的 babel 插件，
​
以下是官网介绍
​

> Externalise references to helpers and builtins, automatically polyfilling your code without polluting globals
> ​
> 所以，参考 [babel-plugin-transform-runtime](https://www.npmjs.com/package/@babel/plugin-transform-runtime) 的原理和作用 ，我们想要自动导入一个 svg，也可以借用 babel-plugin 实现。
> ​
> ​

## 实现原理

​
熟悉 babel 的同学，应该知道 babel 插件作用原理，是通过对转化成 ast 的 js 代码做一些更改、替换之类的操作，不熟悉的同学可以点 [这里](https://github.com/jamiebuilds/babel-handbook) 了解一下 babel 插件是如何开发的。
​
以前文我们提到的这一句代码 `<Icon type="close"/>` 为例，它经过 babel 转化后的 ast 长这个样子
​
<img src="//qhstaticssl.kujiale.com/newt/8/image/png/1560769570817/157C173AF4A5CCC127ECC00233DCC0D1.png" width = "500" height = div />
​
转化成 `json` 会更清晰一些：
​

```
{
 "expression": {
    "type": "JSXElement",
    "start": 0,
    "end": 20,
    "openingElement": {
      "type": "JSXOpeningElement",
      "start": 0,
      "end": 20,
      "attributes": [
        {
          "type": "JSXAttribute",
          "start": 6,
          "end": 18,
          "name": {
            "type": "JSXIdentifier",
            "start": 6,
            "end": 10,
            "name": "type"
          },
          "value": {
            "type": "Literal",
            "start": 11,
            "end": 18,
            "value": "close",
            "raw": "\"close\""
          }
        }
      ],
      "name": {
        "type": "JSXIdentifier",
        "start": 1,
        "end": 5,
        "name": "Icon"
      },
      "selfClosing": true
    },
    "closingElement": null,
    "children": []
    }
  }
```

​
因为用的是 `Jsx` 语法，所以这个表达式的 `type` 是 `JSXElement` , 同时设置了了 `props.type` 的值为 `close` , 所以他会有个 `name` 为 `type` 而 `value` 为 `close` 的 `JSXAttribute` .
​
我们在 babel plugin 中可以拿到上述的分析结果，自然也知道了这条语句产生的作用是：
​

1. 我写下了一个 `type` 为 `close` 的 `Icon Component`，
2. 我希望它能够放一个 `close.svg` 在这里
   ​
   所以我们可以 `new` 一个 `Set()` 对象，将当前 `close` 这个关键词存放进去， 为什么用 `Set` ，因为 `Set` 中的对象是不想等的，免去重复添加关键词然后再去重的必要。
   ​
   代码演示：
   ​

```
function plugin({ types: t }) {
  return {
    visitor: {
      Program: {
        enter(path, state) {
          state.svgSet = new Set();
        }
      }
    }
  };
}
```

​
在初次访问整个语法树的时候，创建一个 Set 对象，注意 `svgSet` 一定要挂在 `state` 上。
​
然后借用 `babel plugin` 分析此文件内的所有 `JSXElement` ，直到整个文件的代码被处理完毕，这样我们就能拿到一个装满了所有关键词的 `Set` 对象。
​
代码片段：

```
function plugin({ types: t }) {
  return {
    visitor: {
      Program: {
       ...
      },
      JSXElement(path, state) {
        const {
          openingElement: {
            attributes
          }
        } = path.node;
        attributes
          .forEach(({ name, value }) => {
            // 判断 name.name 是否等于 "type" 或者是其他设置好的关键词
            state.svgSet.add(value.value);
          });
      }
    }
  };
}
```

​
最后，将 `Set` 里存放的 svg ，遍历之后，用 babel 工具库生成如下的语句：

```
import 'xxx.svg'
```

​
然后插入到此文件的最顶端，剩下的事情就交给 webpack 以及其他 loader 处理了。
​
我已经将上述代码封装了一个 [npm 包](https://www.npmjs.com/package/babel-plugin-jsx-svg-import)，欢迎大家下载和体验，当然目前还比较简陋，源码和详细文档也将在不久后发布。
​
还有 vue 版本的工具也在开发中。
​

## 后记

​
这篇文章实现的 babel 插件原理并不复杂，记录下来希望能够帮助到大家：遇到项目中的问题的时候可以参考社区的实现来解决。
​
代码参考：

- [babel-plugin-transform-runtime](https://github.com/babel/babel/tree/master/packages/babel-plugin-transform-runtime)
- [babel-plugin-import](https://github.com/ant-design/babel-plugin-import)
  ​
  工具使用
- [在线预览 ast](https://astexplorer.net/)
  ​
