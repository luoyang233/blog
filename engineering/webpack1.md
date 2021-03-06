# 白话讲webpack（一）

[白话讲webpack（二）](https://github.com/luoyang233/blog/blob/master/engineering/webpack2.md)

[白话讲webpack（三）](https://github.com/luoyang233/blog/blob/master/engineering/webpack3.md)

[白话讲webpack（四）](https://github.com/luoyang233/blog/blob/master/engineering/webpack4.md)

> 之前一直苦于看不懂webpack的文档，暂时放弃了学习，估计很多和我一样的小白，都是还没开始就结束了，所以我决定结合自身，边学边记录，出一篇能让小白完全读懂的webpack资料。
>
> 文中部分见解引自于拉勾网中的 *Webpack原理与实践 - 汪磊*

## 带着问题找答案

> 这次之所以能够让我基本理解（离彻底还有几篇文章的距离）webpack，还是源自于工作中需要解决的实际问题，项目是用的antd pro框架搭建，在做移动端开发的时候习惯了ES2020可选链（不会自行百度）带来的各种方便，然而web端这边的umi默认配置又没给配提案中的新特性，这就很伤脑经了。
>
> 强行写了几天`a & a.b & a.b.c`后还是受不了了，决定去研究一下`babel`，不懂的暂时不管这是啥，但是就算不懂，大概也知道懂`babel`，那么一定就要懂`webpack`,然后我就带着如何让项目能够使用es2020可选链的问题开始了`webpack`的研究。

下面一段都是废话

----

无论是关于前端还是后端的学习方式，其实都是相通的，光看肯定是没得用滴，就算看个3.4遍看懂了，过一段时间还是忘了，在之前学习webpack的时候我也有一样的经历，所以一定不能看干，边看边动手是最好的学习方式，有些时候以为自己会了，过几天忘了约等于不会，所以还是不会，更好的学习方式就是**带着问题边看边动手**。

---

## webpack到底是个啥？

> 前端打包工具

上面都是看腻了的算半个专业词汇，对于刚接触的小白还是没那么友好，我最喜欢用具体的思维来说抽象的东西，那么webpack到底是个啥？在了解webpack到底是个啥之前，先了解一下为啥会出现webpack这个东西

### CommonJS和AMD规范

没吃过猪肉总看过猪跑吧，不知道CommonJS和AMD规范总听说过吧。一直以来这两个规范对小白来说都是含糊其辞。

#### CommonJS

不管对Node.js熟悉不熟悉，模块化的思维是每个前端必须所掌握的，CommonJS其实就是Node.js中遵循的模块规范，CommonJS中约定一个文件就是一个模块，每个模块都有单独的作用域，通过`module.exports`导出，通过`require`引入。

那么浏览器为啥不直接使用这个规范呢？因为CommonJS约定所有模块都是以同步的方式加载的，你等浏览器给你一通全部同步加载完？？你等得了用户可等不了

#### AMD

这个时候AMD就闪亮登场了，先来看看AMD的英文单词*Asynchronous Module Definition*异步模块定义，好了现在能区分CommonJS和AMD了吗，一个是用在Node.js里**同步**模块加载，一个是用在浏览器中**异步**模块加载。

关于AMD规范，这里简单说一下，不想了解的可以跳过了。
​AMD 规范中约定每个模块通过`define`来定义，`define(arg1,arg2)`中arg1是一个数组，arg2是一个函数，arg2函数接受的参数就和arg1数组中的项一一对应，如果需要向外导出，则直接在arg2函数中return。

上面说的是定义并导出，引用呢，和define类似，通过`require(arg1,arg2)`引入，arg1是一个数组，arg2是一个函数，数组中引入所需要的模块，函数的参数接受该模块

```javascript
//定义并导出模块
define(['jquery','./moduleXXX.js'],function($,muduleXXX){
  return {
    foo:function(){
      $.xxx.xxx
      muduleXXX()
    }
  }
})

//引用并使用模块
require(['./moduleXXX.js'],function(moduleXXX){
  moduleXXX.foo()
})
```

### 扯回来		

> 说了这么多好像和这篇文章要讲的东西越说越远，其实并没有！

AMD规范确实将模块化实现了，但随之也产生了问题，你搞那么多的模块我浏览器不得一个一个去请求？那确实！你等得了他噼里啪啦一通全部请求完，服务器可受不了，怎么办呢？所以这个时候终于扯回到主题上来了，**webpack**！

现在再想一想webpack能干嘛，你既然写代码的时候嫌全局化不方便，然后AMD到后来的ES6都给你模块化整得那么方便，然后你又嫌浏览器请求的时候又请求多了，是不是又想搞成一坨？webpack还真tm是干这个的！好了现在知道大概知道webpack是干啥的了吧

## webpack解决了什么问题

> 前面说了webpack大概是个啥，现在来说webpack它到底是个啥，能干些什么事

- 解决模块文件请求过多的问题，这个在上面中提到了，其实就是把代码揉成一坨（可以实现拆分，这里我们暂时不管）
- 再说ES6中的module规范，虽然现在大多数浏览器都支持了，但它刚出来的时候也不见得那IE啥的把它放在眼里吧，然后你代码写了一通`import xxx from 'xxx'` 然而浏览器它不认识，你能怎么着，这个时候webpack就派上用场了，把它转成AMD规范不就行了
- 不光是模块导出导出的问题，包括ES6的其他语法，以至于后来的ES2016、ES2017等等，都可以将它们转换成浏览器认识的语法（又回到最初的起点。。。）
- 我猜你在开发React或者Vue吧，那.jsx/.sass/.less啥的总见过吧，你认识浏览器可不认识，它就知道.css/.html/.js，这个时候也是webpack把它转换成了这些浏览器认识的文件

好了，我讲完了～