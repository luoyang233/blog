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

### loader

> 首先我们要明白，单是webpack本身，只能打包js代码，那么其他的各种资源，包括图片格式，或者.less/.jsx之类的文件类型，总不可能出一门技术标准，webpack就更新一次吧，这就引出了我们**webpack的核心 -- loader**

loader是啥？

把它理解为一个转换小工具吧，那么webpack其实不能算是打包工具，他算是一个工具盒，拧梅花螺丝需要梅花螺丝刀吧，拧平口螺丝需要平口螺丝刀，这里的loader其实就如同前面的螺丝刀，webpack就是装loader的工具盒

- 例如打包css文件，我们需要某种打包css的loader，比如`css-loader`

  所以我们需要把它引入项目中

  ```
  npm install css-loader --save-dev
  ```

- css文件打包完成后，我们想让他直接放入html的style中，以减少css文件的请求，这个时候就需要`style-loader`

  ```
  npm install style-loader --save-dev
  ```

接下来就是使用它了

```javascript
const path = require('path')
module.exports = {
    entry: './src/index.js',
    output: {
        filename: 'bundle.js',
        path: path.resolve(__dirname, 'dist')
    },
    mode: 'development',
    module: {
        rules: [
            {
                test: /\.css$/,
                use: [
                    { loader: 'style-loader' },
                    {
                        loader: 'css-loader',
                        options: {
                            modules: true
                        }
                    }
                ]
            }
        ]
    }
}
```
- 通过module来引入loader配置
- rules接受一个数组，里面是各种loader匹配各种文件的使用规则
- test接受一个正则表达式，即表明这个loader只在匹配到.css文件的时候用
- use则是表明使用哪一种loader，**顺序为从后到前**，这里的配置为：匹配到.css文件先使用css-loader，接着css-loader转换后的文件再交给style-loader继续转换
- use的顺序一定不能乱，比如这里的顺序颠倒，那就是先执行style-loader，后执行css-loader，先没学会走路，就直接跑了，肯定会报错
- options则是针对css-loader的配置

### 深入一下loader的原理

> loader的原理其实很简单，loader链要求最终返回一段js代码即可

比如我们在文件中引入了一个README.md文件，这个时候打包，webpack肯定不认识

```javascript
//index.js
import readme from './README.md'
	foo=()=>{
  	console.log(readme)
	}

//README.md
#hello 
world

//npx webpack 控制台报错
ERROR in ./src/README.md 1:0
Module parse failed: Unexpected character '#' (1:0)
You may need an appropriate loader to handle this file type, currently no loaders are configured to process this file. See https://webpack.js.org/concepts#loaders
> # hello
| world
 @ ./src/index.js 2:0-32 7:14-20
```

这个时候就需要有一个能转换.md文件的loader，假如还没人写这个loader呢？

那就需要自己动手了，写个简单的demo

```javascript
//my-md-loader.js

//注意这里一定要用CommonJS规范，因为loader是在node环境中运行
module.exports=source=>{
	//这里的source其实就是README中的所有内容
  //'# hello\n\world'
  return 'export default 'hello,world!''
}
```

- 这里的source包含了.md文件中的所有内容
- loader返回**字符串**，但字符串的内容是一段**js代码**，返回的就是经过loader处理后交出去的
- 现在可以把README.md的文件内容直接看作`export default 'hello,world!'`
- 注意这里不一定非要导出，比如这里可以直接返回`return ''hello world''`,注意这里是两个引号，交出去的相当于就是`'hello world'`，但是现在，在代码中就无法直接使用了，对引入它的index.js来说，该README.md就是一行写有'hello world'的js文件，其他啥都没有，连导出都没有，当然不能用
- 接上一条，这个时候就需要更多的loader了，那么下一个loader接受到的就是`'hello world'`，然后再进行各种骚操作。。。

最后配置一下

```javascript
const path = require('path')
module.exports = {
    entry: './src/index.js',
    output: {
        filename: 'bundle.js',
        path: path.resolve(__dirname, 'dist')
    },
    mode: 'development',
    module: {
        rules: [
            {
                test: /\.css$/,
                use: [
                    { loader: 'style-loader' },
                    {
                        loader: 'css-loader',
                        options: {
                            modules: true
                        }
                    }
                ]
            },
          {
            test: /\.md/,
            use:['./my-md-loader','othder1-loader','othder2-loader']
          }
        ]
    }
}
```



说完了～

