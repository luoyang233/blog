# 白话讲webpack（二）

[白话讲webpack（一）](https://github.com/luoyang233/blog/blob/master/engineering/webpack1.md)

> 接上一篇，对webpack有了一个大体的认识，但真正落到实处的东西，可不是几句口水话就能说清楚的。这篇文章就是对webpack配置文件做简单的介绍

## 准备工作

> 边看边动手才记得住

- 命令行初始化一个项目`npm init --yes`
- 项目中添加一个webpack配置文件`webpack.config.js`
- `npm i webpack webpack-cli -D`这里的cli也就是命令行工具，运行webpack时需要
- 建议文件src/index.js

## 配置文件

> 什么是配置文件？前文说了webpack是一个把各种东西打包的工具，那写好了一个项目之后，从哪里开始打包，打包后的文件放在哪里去，哪些文件需要打包，打包需要把他包成什么样子，这些问题都需要告诉webpack，不然他咋知道你要怎么搞。

webpack打包总共有3种方式，其他两种可以自行百度，一般的都没怎么用，这里只对配置的方式做介绍。

### entry

首先来看一段最基本的`webpack.config.js`的配置文件

```javascript
module.exports = {
    entry: './src/index.js'
}
```

- webpack是运行在node.js环境中，所以最前面是通过commonJS规范导出该配置文件

- `entry`其实很容易猜到，顾名思义，就是入口，从哪个文件开始顺藤摸瓜打包下去

- 入口可能还以数组[]的形式，也就是多个入口，这里才第二篇讲这么多干嘛，跳过！

  

实际上这个时候通过`npx webpack`可以直接运行了，会发现多了一个dist/main.js文件，这里的main.js就是打包后的结果

再或者，甚至可以把配置文件删了，运行出来也是同样的结果，因为webpack4本身拥有默认配置

### output

```javascript
const path = require('path')

module.exports = {
    entry: './src/index.js',
  	output: {
      filename:'bundle.js',
      path:path.resolve(__dirname,'dist')
    }
}
```

- path为node.js中的代码，`path.resolve(__dirname,'dist')`这段代码的意思就是，输出一个绝对路径，path.resolve用法自行百度

- output是一个对象，至少必须拥有filename和path两个属性，也是顾名思义，filename就是打包后的文件名，path就是打包后的文件存放的路径

### mode

```javascript
const path = require('path')

module.exports = {
    entry: './src/index.js',
  	output: {
      filename:'bundle.js',
      path:path.resolve(__dirname,'dist')
    },
  mode:'development'
}
```

- mode总共有三种模式`none`,`production`,`development`
- mode其实也就是控制打包结果，最好的理解方式就是直接去看打包后的bundle.js
- 当mode为`development`时也就是开发环境，打包结果当然不用那么精细，你给我快点打包完，我好去调试就行了
- 当mode为`production`时也就是生产环境，要求就是压缩压缩再压缩，这个时候再去看看bundle.js，就会发现这不是人类能读懂的了



这里讲讲打包原理，不感兴趣可以跳过

---

查看打包后的bundle.js文件会发现，实际上它是这么一个函数

```javascript
(function(modules){})([fn1,fn2,fn3,...])
```

也就是一个自执行的函数，参数为一个数组，里面的每一项对应一个模块

然后展开前面的自执行函数

```javascript
(function(modules){
	var cache=[] //已加载过的模块
	function require(id){}
  //...中间省略一些其他东西
	return require(0)
})([fn0,fn1,fn2,...])
```

这里展开后重点就这么几个

- `cache`用于存储已经加载过的模块
- 中间有个`require`函数接受一个id参数，这里的id其实就是对应的数组项，也就是第几个函数，即第几个模块
- 最后将0传入`require`函数并调用，这里可以直接当成将第一个函数（模块）传入require函数中调用，这里其实也是一开始定义的入口，通过这里顺藤摸瓜，摸下去

我们再来把`require`函数展开

```javascript
function(modules){
	var cache=[] //已加载过的模块
   // ========================================
	function require(id){
    if(cache[id]) {
      return cache[id].export
    }
    const module={
      export:{},
      id,
      loaded:false
    }
    modules[id].call(module.export,module.module.export,require)
    module.loaded=true
    return module.export
	}
  // ========================================
  //...中间省略一些其他东西
  return require(0)
})([fn0,fn1,fn2,...])
```

展开的代码也就是中间的那一部分，我们先说下大致过程

- 首先是判断是否被加载过，加载了就直接返回
- 接着定义这个module，先不管是干啥的
- 调用传入的函数（就这样想吧）
- 返回module.export

暂时不用细想，先来看看前面的fn0,fn1,fn2长什么样子

```javascript
function fn0(modules,export,require){
  //对应原本代码中的var a =import './xx.js'
  var obj = require(1)
  obj.foo()
}

function fn1(modules,export,require){
  //对应原本代码中的export foo = console.log(666)
  export.foo = console.log(666)
}
```

- 这里的`var a = require(1)`实际上对应原本代码中的`var a =import './xx.js' `
- 这里的`export.foo = console.log(666)`实际上对应原本代码中的`export foo = console.log(666) `

再来看看整段代码

```javascript
function(modules){
	var cache=[] //已加载过的模块
   // ========================================
	function require(id){
     if(cache[id]) {
       return cache[id].export
     }
		 const module={
       export:{},
       id,
       loaded:false
     }
  modules[id].call(module.export,module.module.export,require)
	module.loaded=true
  return module.export
	}
  // ========================================
  //...中间省略一些其他东西
	return require(0)
})([function fn0(modules,export,require){
  //对应原本代码中的var a =import './xx.js'
  var a = require(1)
},function fn1(modules,export,require){
  //对应原本代码中的export foo = console.log(666)
  export.foo = console.log(666)
},fn2,...])
```

先别晕，有一点点上头，但想通了会发现这段代码特别妙

- 首先是调用了fn0，将`require`函数作为参数传给了fn0
- 再fn0的内部又调用了`require`函数，这个时候require(1)开始跑了
- 跟require(0)的步骤一样，到了调用fn1这里，注意这里**module.export作为了参数传给fn1**
- 在fn1的回调过程中修改了module.export，最后将module.export返回
- 这里又回到了刚才的fn0中，`var a = require(1)`a就顺利拿到了fn1中的导出内容，也就是被修改后的export

绕明白了吗？总结一下，**实际上这里就是将export交给回调函数修改它的值，最后再将它返回，返回到原来调用它的函数**，这就是导出导入一环扣一环的被传递的原因。

妙吗？或许这就是高级coder和程序猿的区别吧！

---

未完。。

