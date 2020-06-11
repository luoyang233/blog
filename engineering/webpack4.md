# 白话讲webpack（四）

[白话讲webpack（一）](https://github.com/luoyang233/blog/blob/master/engineering/webpack1.md)

[白话讲webpack（二）](https://github.com/luoyang233/blog/blob/master/engineering/webpack2.md)

[白话讲webpack（三）](https://github.com/luoyang233/blog/blob/master/engineering/webpack3.md)

> 这是最后一篇关于webpack的文章，这一篇主要是对webpack打包后的优化项作介绍

## tree shaking

tree shaking的意思就是'摇树'，就是把树上的枯枝烂叶摇下来，代码也是一样，在我们的日常编写中，总会不小心的出现一些忘记删除又未被引用的代码，然后这部分代码如果不做处理（当然webpack的prod模式下会自动处理），就会被打包下来，显得很多余

比如有以下一段代码

```javascript
// ./src/components.js
export const Button = () => {
  return document.createElement('button')
  console.log('dead-code')
}

export const Link = () => {
  return document.createElement('a')
}

export const Heading = level => {
  return document.createElement('h' + level)
}

// ./src/main.js
import { Button } from './components'
document.body.appendChild(Button())

```

很明显，上面的Link和Heading函数都是多余的，包括Button函数中的 `console.log('dead-code')`也是永远都不会执行

在`mode:'production'`模式下，webpack会自动处理，我们是抱着学习的目的去了解，所以首先把它关闭`mode:none`

<img src="https://github.com/luoyang233/blog/blob/master/images/webpack4_1.png" alt="webpack4_1" style="zoom:50%;" />

可以在结果中看到，webpack将他们全部都打包入其中，甚至是注释也没放过，虽然都将它们导出了，但是却没有使用它，造成了空间和性能上的浪费

现在来开启tree shaking，这里说一下开启tree shaking主要的两个配置项

- `usedExports:true`先不管是啥，把它理解为标记树叶
- `minimize:true`同样不管是啥，把它理解为把标记的树叶摇下来

首先只使用usedExport开启标记树叶

```javascript
// ./webpack.config.js
module.exports = {
  // ... 其他配置项
  optimization: {
    // 模块只导出被使用的成员
    usedExports: true
  }
}

```

<img src="https://github.com/luoyang233/blog/blob/master/images/webpack4_2.png" alt="webpack4_2" style="zoom:50%;" />

对比前面的结果可以发现，虽然现在还是会有多余的代码，但是现在只会导出它该导出的，这些多余的代码没地方使用，已经被vscode标记为了灰色

现在这个过程就好比把树上的多余的叶子标记了出来，接下来就是摇落它

```javascript
// ./webpack.config.js
module.exports = {
  // ... 其他配置项
  optimization: {
    // 模块只导出被使用的成员
    usedExports: true,
    // 压缩输出结果
    minimize: true
  }
}
```

结果就不截图了，代码又变成了人类看不懂的样子。。。

总之就是，Link 和 Heading 这些未引用代码都被自动移除了

所以再来看一看前面的配置

- `usedExports:true`标记未引用代码，未引用代码将不会被导出
- `minimize:true`摇下被标记的代码，压缩打包结果

## Code Splitting

> 有个成语叫做物极必反

所以当我们的应用变得越来越庞大，越来越复杂的时候，如果所有的代码还是被打包进同一个bundle，那这个文件将变得特别的大，反而会影响文件的加载速度

而且在首屏渲染中，一般的都不会用到所有的东西，所以按需加载尤为重要

刚接触时可能会有这样的问题，最开始的前端就是散的，后来有个打包工具又要合并，现在又要拆分，不是矛盾了吗？其实并不矛盾，可以理解成这样

<center>项目中有1000文件 -> webpack打包成1个 -> 拆分成10个</center>

> P.S.真正的打包当中不会是先打包成一个又来拆分，会是直接就拆分成n个，这里只是为了方便理解为什么要拆分的思想

接下来看一看分块打包如何配置

- 首先是`entry`，可以分成多个入口
- 再是`output`，根据入口名导出文件
- `optimization.splitChunks.chunk`配置是为了提取模块间的公用块，比如都用到了某个很大的模块，如果不提去公共部分，就会造成空间和性能的浪费
- HtmlWebpackPlugin插件中的配置不需要过多的关心，里面的chunk，只是为了配合分块打包的一项配置，具体在使用这个插件的时候再去了解就行了

```javascript
// ./webpack.config.js
const HtmlWebpackPlugin = require('html-webpack-plugin')
module.exports = {
  entry: {
    index: './src/index.js',
    album: './src/album.js'
  },
  output: {
    filename: '[name].bundle.js' // [name] 是入口名称
  },
  optimization: {
    splitChunks: {
      // 自动提取所有公共模块到单独 bundle
      chunks: 'all'
    }
  }
  // ... 其他配置
  plugins: [
    new HtmlWebpackPlugin({
      title: 'Multi Entry',
      template: './src/index.html',
      filename: 'index.html',
      chunks: ['index'] // 指定使用 index.bundle.js
    }),
    new HtmlWebpackPlugin({
      title: 'Multi Entry',
      template: './src/album.html',
      filename: 'album.html',
      chunks: ['album'] // 指定使用 album.bundle.js
    })
  ]
}

```

从上面的配置来看，其实分块打包还是很简单，其实主要用到的就是一个分入口导入，提去公共部分只是一个优化项而已

实际的SPA开发过程当中，入口一般还是只有一个，主要是通过ES6中的`const mod = import('./src/xxx')`这种方式来达到按需加载的目的，然后webpack内部会自动处理分包和按需加载，无需额外配置

`import()`动态导入的方式，可以直接参考阮一峰写的[ES6教程](https://es6.ruanyifeng.com/)，这里就不在关公面前耍大刀了

完了～