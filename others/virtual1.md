## 长列表渲染之时间分片

> 这是一片质量比较高的文章，长列表渲染一般有两种方式，一种是分时渲染，另一种是[虚拟渲染](https://github.com/luoyang233/blog/blob/master/others/virtual2.md)，该文章介绍的是分时渲染的方式
>
> 引自[掘金-时间分片](https://juejin.cn/post/6844903938894872589)

## 前言

在实际工作中，我们很少会遇到一次性需要向页面中插入大量数据的情况，但是为了丰富我们的知识体系，我们有必要了解并清楚当遇到大量数据时，如何才能在不卡主页面的情况下渲染数据，以及其中背后的原理。

对于一次性插入大量数据的情况，一般有两种做法：

1. 时间分片
2. 虚拟列表
   

## 最粗暴的做法（一次性渲染）

我们先来看看最粗暴的做法，一次性将大量数据插入到页面中：

```html
<ul id="container"></ul>
复制代码
// 记录任务开始时间
let now = Date.now();
// 插入十万条数据
const total = 100000;
// 获取容器
let ul = document.getElementById('container');
// 将数据插入容器中
for (let i = 0; i < total; i++) {
    let li = document.createElement('li');
    li.innerText = ~~(Math.random() * total)
    ul.appendChild(li);
}

console.log('JS运行时间：',Date.now() - now);
setTimeout(()=>{
  console.log('总运行时间：',Date.now() - now);
},0)
// print: JS运行时间： 187
// print: 总运行时间： 2844
```

我们对十万条记录进行循环操作，JS的运行时间为`187ms`，还是蛮快的，但是最终渲染完成后的总时间确是`2844ms`。

简单说明一下，为何两次`console.log`的结果时间差异巨大，并且是如何简单来统计`JS运行时间`和`总渲染时间`：

- 在 JS 的`Event Loop`中，当JS引擎所管理的执行栈中的事件以及所有微任务事件全部执行完后，才会触发渲染线程对页面进行渲染
- 第一个`console.log`的触发时间是在页面进行渲染之前，此时得到的间隔时间为JS运行所需要的时间
- 第二个`console.log`是放到 setTimeout 中的，它的触发时间是在渲染完成，在下一次`Event Loop`中执行的

[关于Event Loop的详细内容请参见这篇文章-->](https://juejin.im/post/6844903919789801486)

依照两次`console.log`的结果，可以得出结论：

对于大量数据渲染的时候，JS运算并不是性能的瓶颈，性能的瓶颈主要在于渲染阶段

## 使用定时器

从上面的例子，我们已经知道，页面的卡顿是由于同时渲染大量DOM所引起的，所以我们考虑将渲染过程分批进行在这里，我们使用`setTimeout`来实现分批渲染

```html
<ul id="container"></ul>
复制代码
//需要插入的容器
let ul = document.getElementById('container');
// 插入十万条数据
let total = 100000;
// 一次插入 20 条
let once = 20;
//总页数
let page = total/once
//每条记录的索引
let index = 0;
//循环加载数据
function loop(curTotal,curIndex){
    if(curTotal <= 0){
        return false;
    }
    //每页多少条
    let pageCount = Math.min(curTotal , once);
    setTimeout(()=>{
        for(let i = 0; i < pageCount; i++){
            let li = document.createElement('li');
            li.innerText = curIndex + i + ' : ' + ~~(Math.random() * total)
            ul.appendChild(li)
        }
        loop(curTotal - pageCount,curIndex + pageCount)
    },0)
}
loop(total,index);
```

用一个gif图来看一下效果

<img src="https://github.com/luoyang233/blog/blob/master/images/virtual_1.gif" alt="oauth2_2" style="zoom:50%;" />

我们可以看到，页面加载的时间已经非常快了，每次刷新时可以很快的看到第一屏的所有数据，但是当我们快速滚动页面的时候，会发现页面出现闪屏或白屏的现象

### 为什么会出现闪屏现象呢

首先，理清一些概念。`FPS`表示的是每秒钟画面更新次数。我们平时所看到的连续画面都是由一幅幅静止画面组成的，每幅画面称为一`帧`，`FPS`是描述`帧`变化速度的物理量。

大多数电脑显示器的刷新频率是60Hz，大概相当于每秒钟重绘60次，`FPS`为60frame/s，为这个值的设定受屏幕分辨率、屏幕尺寸和显卡的影响。

因此，当你对着电脑屏幕什么也不做的情况下，大多显示器也会以每秒60次的频率正在不断的更新屏幕上的图像。

为什么你感觉不到这个变化？

那是因为人的眼睛有视觉停留效应，即前一副画面留在大脑的印象还没消失，紧接着后一副画面就跟上来了， 这中间只间隔了16.7ms(1000/60≈16.7)，所以会让你误以为屏幕上的图像是静止不动的。

而屏幕给你的这种感觉是对的，试想一下，如果刷新频率变成1次/秒，屏幕上的图像就会出现严重的闪烁， 这样就很容易引起眼睛疲劳、酸痛和头晕目眩等症状。

大多数浏览器都会对重绘操作加以限制，不超过显示器的重绘频率，因为即使超过那个频率用户体验也不会有提升。 因此，最平滑动画的最佳循环间隔是1000ms/60，约等于16.6ms。

直观感受，不同帧率的体验：

- 帧率能够达到 50 ～ 60 FPS 的动画将会相当流畅，让人倍感舒适；
- 帧率在 30 ～ 50 FPS 之间的动画，因各人敏感程度不同，舒适度因人而异；
- 帧率在 30 FPS 以下的动画，让人感觉到明显的卡顿和不适感；
- 帧率波动很大的动画，亦会使人感觉到卡顿。

### 简单聊一下 setTimeout 和闪屏现象

- `setTimeout`的执行时间并不是确定的。在JS中，`setTimeout`任务被放进事件队列中，只有主线程执行完才会去检查事件队列中的任务是否需要执行，因此`setTimeout`的实际执行时间可能会比其设定的时间晚一些。
- 刷新频率受屏幕分辨率和屏幕尺寸的影响，因此不同设备的刷新频率可能会不同，而`setTimeout`只能设置一个固定时间间隔，这个时间不一定和屏幕的刷新时间相同。

以上两种情况都会导致setTimeout的执行步调和屏幕的刷新步调不一致。

在`setTimeout`中对dom进行操作，必须要等到屏幕下次绘制时才能更新到屏幕上，如果两者步调不一致，就可能导致中间某一帧的操作被跨越过去，而直接更新下一帧的元素，从而导致丢帧现象。

## 使用 requestAnimationFrame

与`setTimeout`相比，`requestAnimationFrame`最大的优势是由系统来决定回调函数的执行时机。

如果屏幕刷新率是60Hz,那么回调函数就每16.7ms被执行一次，如果刷新率是75Hz，那么这个时间间隔就变成了1000/75=13.3ms，换句话说就是，`requestAnimationFrame`的步伐跟着系统的刷新步伐走。它能保证回调函数在屏幕每一次的刷新间隔中只被执行一次，这样就不会引起丢帧现象。

我们使用`requestAnimationFrame`来进行分批渲染：

```html
<ul id="container"></ul>
```

```html
//需要插入的容器
let ul = document.getElementById('container');
// 插入十万条数据
let total = 100000;
// 一次插入 20 条
let once = 20;
//总页数
let page = total/once
//每条记录的索引
let index = 0;
//循环加载数据
function loop(curTotal,curIndex){
    if(curTotal <= 0){
        return false;
    }
    //每页多少条
    let pageCount = Math.min(curTotal , once);
    window.requestAnimationFrame(function(){
        for(let i = 0; i < pageCount; i++){
            let li = document.createElement('li');
            li.innerText = curIndex + i + ' : ' + ~~(Math.random() * total)
            ul.appendChild(li)
        }
        loop(curTotal - pageCount,curIndex + pageCount)
    })
}
loop(total,index);
```

看下效果

<img src="https://github.com/luoyang233/blog/blob/master/images/virtual_2.gif" alt="oauth2_2" style="zoom:50%;" />

我们可以看到，页面加载的速度很快，并且滚动的时候，也很流畅没有出现闪烁丢帧的现象。

这就结束了么，还可以再优化么？

当然~~

## 使用 DocumentFragment

先解释一下什么是 DocumentFragment ，文献引用自[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/DocumentFragment)

> ```
> DocumentFragment`，文档片段接口，表示一个没有父级文件的最小文档对象。它被作为一个轻量版的`Document`使用，用于存储已排好版的或尚未打理好格式的XML片段。最大的区别是因为`DocumentFragment`不是真实DOM树的一部分，它的变化不会触发DOM树的（重新渲染) ，且不会导致性能等问题。
>  可以使用`document.createDocumentFragment`方法或者构造函数来创建一个空的`DocumentFragment
> ```

从MDN的说明中，我们得知`DocumentFragments`是DOM节点，但并不是DOM树的一部分，可以认为是存在内存中的，所以将子元素插入到文档片段时不会引起页面回流。

当`append`元素到`document`中时，被`append`进去的元素的样式表的计算是同步发生的，此时调用 getComputedStyle 可以得到样式的计算值。 而`append`元素到`documentFragment` 中时，是不会计算元素的样式表，所以`documentFragment` 性能更优。当然现在浏览器的优化已经做的很好了， 当`append`元素到`document`中后，没有访问 getComputedStyle 之类的方法时，现代浏览器也可以把样式表的计算推迟到脚本执行之后。

最后修改代码如下：

```html
<ul id="container"></ul>
复制代码
//需要插入的容器
let ul = document.getElementById('container');
// 插入十万条数据
let total = 100000;
// 一次插入 20 条
let once = 20;
//总页数
let page = total/once
//每条记录的索引
let index = 0;
//循环加载数据
function loop(curTotal,curIndex){
    if(curTotal <= 0){
        return false;
    }
    //每页多少条
    let pageCount = Math.min(curTotal , once);
    window.requestAnimationFrame(function(){
        let fragment = document.createDocumentFragment();
        for(let i = 0; i < pageCount; i++){
            let li = document.createElement('li');
            li.innerText = curIndex + i + ' : ' + ~~(Math.random() * total)
            fragment.appendChild(li)
        }
        ul.appendChild(fragment)
        loop(curTotal - pageCount,curIndex + pageCount)
    })
}
loop(total,index);

```