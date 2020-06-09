# 白话讲webpack（三）

[白话讲webpack（一）](https://github.com/luoyang233/blog/blob/master/engineering/webpack1.md)

[白话讲webpack（二）](https://github.com/luoyang233/blog/blob/master/engineering/webpack2.md)

> 该篇是介绍webpack在实战中常用的热替换机制（HMR）

## 为何需要HMR

### 目前为止的开发方式

通过前两篇的介绍，到目前为止，我们使用webpack的方式是

**源代码->手动打包->serve运行->浏览器查看->修改源代码->重新手动打包->浏览器刷新**

确实够麻烦的

但在我们实际开发过程中，往往都不是这样，一般都是在修改代码保存后，浏览器就自动刷新了，过程中也无需我们手动打包和刷新浏览器

## 简化过程

### 自动打包

很简单

```
npx webpack --watch
```

在启动打包时增加一个watch配置就行了，这个时候webpack在打包后会一直处于监听状态，一旦有文件的修改，将自动重新打包

这个时候我们的过程变成了

**源代码->运行打包->serve运行->浏览器查看->修改源代码->重新自动打包->浏览器刷新**

### 自动刷新

在自动打包完成后，我们依然需要自己手动去刷新浏览器，但在实际开发过程中肯定不能这样，所以这个时候就需要**Webpack Dev Server**

Webpack Dev Server是官方提供的，**集成了自动打包和自动刷新于一体**，但它和上面使用的watch模式又有一点点不同，想象一下，一个足够大的项目，在每次修改文件后，都得重新打包，那得是有多少的IO操作

所以Webpack Dev Server和watch不同之处在于，**Webpack Dev Server是将打包结果放在内存中**，大大提升了构建的效率

```javascript
//安装
npm install webpack-dev-server --save-dev
//运行
//--open参数，可在打包完成后自动打开浏览器
npx webpack-dev-server --open
```

直接通过`devServer`字段配置

```javascript
// ./webpack.config.js
const path = require('path')

module.exports = {
  // ...
  devServer: {
    contentBase: path.join(__dirname, 'dist'),
    compress: true,
    port: 9000
    // ...
    // 详细配置文档：https://webpack.js.org/configuration/dev-server/
  }
}

```

这个时候我们的过程变成了

**源代码->打包后自动运行浏览器->修改源代码->重新自动打包->浏览器自动刷新**

### HMR

> 到目前为止，我们已经实现了正常的开发流程了
>
> 吗？

还没有！

假如我们有一个页面，需要填写10个input后提交，我们好不容易疯狂的一阵输入，全部填完后，发现有段代码写错了，然后这个时候去改了它，好了，webpack-dev-server给我自动刷新了，然后就会惊讶的发现，之前填的数据全部都不在了，然后我又要一阵疯狂的填写

但在我们实际开发过程中，好像不是这样，例如我们在react脚手架项目中更改代码后，往往能够在不需要重新进入某个更改的页面，就直接能查看到更改后的结果，并且也保存了上一次的状态

这个过程正是叫做热替换，也就是HMR(Hot Module Replacement)

使用它也简单，它也已经集成在webpack之中

```
npx webpack-dev-server --hot
```

同时也可以通过配置的方式

⚠️注意它需要一个内部的**HotModuleReplacementPlugin**插件

```javascript
// ./webpack.config.js
const webpack = require('webpack')

module.exports = {
  // ...
  devServer: {
    // 开启 HMR 特性，如果资源不支持 HMR 会 fallback 到 live reloading
    //live reloading即是在不能热替换的情况下，完全刷新
    hot: true
    // 只使用 HMR，不会 fallback 到 live reloading
    // hotOnly: true
  },
  plugins: [
    // ...
    // HMR 特性所需要的插件
    new webpack.HotModuleReplacementPlugin()
  ]
}

```

这个时候我们的过程变成了

**源代码->打包后自动运行浏览器->修改源代码->重新自动打包->浏览器自动刷新并保存状态**

### 使用HMR所带来的问题	

> 到目前为止，我们已经疯狂的接近正常的开发流程了，甚至就是一个正常的开发流程了
>
> 吗？

还没完！

试着修改一下css文件会发现页面被成功热替换，但是在修改js文件之后，会发现页面依然是整体刷新，并没有保留上次的状态，换句话说在修改js文件后，执行的是live reloading而不是HMR

怎么回事呢？

webpack 中的 HMR 需要我们手动通过代码去处理，当模块更新过后该，如何把更新后的模块替换到页面中

因为js文件是没有任何规律的，我们可以导出一个string，在下一次修改之后我们也可以改变为导出一个dom节点，所以HMR并没有一个统一的处理规范，因为没法猜测接下来的修改操作

之所以css文件能够修改，因为在style-loader中就已经将热更新处理

再比如说react脚手架项目，jsx文件导出的必定是一个组件或函数，所以它是有规律可循的，所以内部能够实现热替换

所以js代码我们需要自己来手动处理

```javascript
// ./main.js

//。。。

module.hot.accept('./component', () => {
  // 当 ./component.js 更新，自动执行此函数
  console.log('component 更新了～～')
})
```

module是一个全局变量，hot是在开启HMR后被注入的一个变量，通过其中的accept方法接收处理函数

在main.js中引入了component.js，一旦component文件发生变化，将自动执行作为第二个参数的函数，浏览器将不再刷新

所以在回调函数中，一般进行的就是对被修改的dom进行直接操作，注意要移除之前的dom，和保留之前的状态，下面的代码只是一个简单的示范，大致原理就是这样

```javascript
module.hot.accept('./component', () => {
   // 临时记录更新前的状态
  const value = old_dom.innerHTML
  document.body.removeChild(old_dom) // 移除之前创建的元素
  const new_dom = createEditor() // 用新模块创建新元素
  // 还原状态
  new_dom.innerHTML = value
  document.body.appendChild(new_dom)
})
```

- 回调函数中一旦发生错误，将自动运行live reloading
- 但在配置中设置`hotOnly:true`后，将不再自动reloading，而是在控制台中打印错误

所以看到这里，应该就能明白，为什么HMR没有一个统一的处理规范，做不到开箱即用的原因，但实战当中，我们一般都是使用的成熟的集成框架，HMR已经在内部封装好了，所以我们只需要使用它就行了

⚠️**注意我们只需要将HMR配置在dev环境，同时别忘记了引入HotModuleReplacementPlugin插件**

[React HMR 方案](https://github.com/gaearon/react-hot-loader)

[Vue.js HMR 方案](https://github.com/gaearon/react-hot-loader)

### SourceMap

> 到目前为止，我们已经完全实现了实际开发中所用到的流程了
>
> 吗？

还是没有！

想一想，还缺了什么，如果在代码中我们写了一段可以编译通过的错误代码，在运行时发生错误怎么办

当然是将错误映射在浏览器中，指出错误发生的代码，这就是source map干的事情

#### source map 是什么

- 它不是webpack独有，而是webpack支持source map，就如同其他打包工具也支持source map一样
- 打包后的bundle.js是一个压缩文件，当错误发生时，指出错误的代码后，就算拿给你，你也不知道这是哪里出错了，因为甚至是变量的命名都有可能发生改变
- source map就类似于一个摩尔斯电码的转换图，...---...代表sos，只不过这里是代码上的转换图，例如压缩后的m代表变量myValue，它是以json格式编写，文件名以.map结尾的文件

使用方式很简单，但真正难的是，理解各种模式下的不同点

```javascript
// ./webpack.config.js
module.exports = {
  devtool: 'source-map' // source map 设置
}
```

webpack运行后会生成一个bundle.js和bundle.js.map文件

bundle.js文件的尾部会通过`//# sourceMappingURL=bundle.js.map`引入map文件，一旦报错，浏览器的控制台中就会打印出错误来源，跟进去，就会看到已经被反解析出来的源代码和错误行

关于各种模式这里不多做介绍，网上有很多，大概记住几个模式就可以了

- cheap-module-eval-source-map：eval代表通过eval的方式在浏览器中执行，module代表不通过loader的转换，直接展示原始代码，cheap代表错误只定位到行，不需要定位到列
-  nosources-source-map：只展示错误代码位置，不展示源代码，一般用于生产环境，方便定位到错误原因
- none：也就是不产生map文件，也是用在生产环境，防止源代码泄漏，生产环境更推荐none模式

好了，到现在终于达到开发环境的环境需求了

！

