## 长列表渲染之虚拟列表

> 这是关于长列表渲染的第二种方式，[虚拟渲染](https://github.com/luoyang233/blog/blob/master/others/virtual2.md)，比上一篇中的[时间分片](https://github.com/luoyang233/blog/blob/master/others/virtual1.md)方式应用得更加广泛。
>
> 引自[掘金-虚拟渲染](https://juejin.cn/post/6844903982742110216)

## 前言

在工作中，有时会遇到需要一些不能使用分页方式来加载列表数据的业务情况，对于此，我们称这种列表叫做`长列表`。比如，在一些外汇交易系统中，前端会实时的展示用户的持仓情况(收益、亏损、手数等)，此时对于用户的持仓列表一般是不能分页的。

## 什么是虚拟列表

`虚拟列表`其实是按需显示的一种实现，即只对`可见区域`进行渲染，对`非可见区域`中的数据不渲染或部分渲染的技术，从而达到极高的渲染性能。

假设有1万条记录需要同时渲染，我们屏幕的`可见区域`的高度为`500px`,而列表项的高度为`50px`，则此时我们在屏幕中最多只能看到10个列表项，那么在首次渲染的时候，我们只需加载10条即可。

<img src="https://github.com/luoyang233/blog/blob/master/images/virtual_3.jpg" alt="oauth2_2" style="zoom:50%;" />

由于只是对`可视区域`内的列表项进行渲染，所以为了保持列表容器的高度并可正常的触发滚动，将Html结构设计成如下结构：

```html
<div class="infinite-list-container">
    <div class="infinite-list-phantom"></div>
    <div class="infinite-list">
      <!-- item-1 -->
      <!-- item-2 -->
      <!-- ...... -->
      <!-- item-n -->
    </div>
</div>
```

- `infinite-list-container` 为`可视区域`的容器
- `infinite-list-phantom` 为容器内的占位，高度为总列表高度，用于形成滚动条
- `infinite-list` 为列表项的`渲染区域`

>这里注释一下，`infinite-list-phantom`是作为一个`absolute`的存在，一是不参与dom的回流，二是通过`z-index:-1`来隐藏该标签，达到撑起外部container形成滚动条的作用

接着，监听`infinite-list-container`的`scroll`事件，获取滚动位置`scrollTop`

- 假定`可视区域`高度固定，称之为`screenHeight`
- 假定`列表每项`高度固定，称之为`itemSize`
- 假定`列表数据`称之为`listData`
- 假定`当前滚动位置`称之为`scrollTop`

则可推算出：

- 列表总高度`listHeight` = listData.length * itemSize
- 可显示的列表项数`visibleCount` = Math.ceil(screenHeight / itemSize)
- 数据的起始索引`startIndex` = Math.floor(scrollTop / itemSize)
- 数据的结束索引`endIndex` = startIndex + visibleCount
- 列表显示数据为`visibleData` = listData.slice(startIndex,endIndex)

当滚动后，由于`渲染区域`相对于`可视区域`已经发生了偏移，此时我需要获取一个偏移量`startOffset`，通过样式控制将`渲染区域`偏移至`可视区域`中。

- 偏移量`startOffset` = scrollTop - (scrollTop % itemSize);

最终的`简易代码`如下：

```html
<template>
  <div ref="list" class="infinite-list-container" @scroll="scrollEvent($event)">
    <div class="infinite-list-phantom" :style="{ height: listHeight + 'px' }"></div>
    <div class="infinite-list" :style="{ transform: getTransform }">
      <div ref="items"
        class="infinite-list-item"
        v-for="item in visibleData"
        :key="item.id"
        :style="{ height: itemSize + 'px',lineHeight: itemSize + 'px' }"
      >{{ item.value }}</div>
    </div>
  </div>
</template>
```

```javascript
export default {
  name:'VirtualList',
  props: {
    //所有列表数据
    listData:{
      type:Array,
      default:()=>[]
    },
    //每项高度
    itemSize: {
      type: Number,
      default:200
    }
  },
  computed:{
    //列表总高度
    listHeight(){
      return this.listData.length * this.itemSize;
    },
    //可显示的列表项数
    visibleCount(){
      return Math.ceil(this.screenHeight / this.itemSize)
    },
    //偏移量对应的style
    getTransform(){
      return `translate3d(0,${this.startOffset}px,0)`;
    },
    //获取真实显示列表数据
    visibleData(){
      return this.listData.slice(this.start, Math.min(this.end,this.listData.length));
    }
  },
  mounted() {
    this.screenHeight = this.$el.clientHeight;
    this.start = 0;
    this.end = this.start + this.visibleCount;
  },
  data() {
    return {
      //可视区域高度
      screenHeight:0,
      //偏移量
      startOffset:0,
      //起始索引
      start:0,
      //结束索引
      end:null,
    };
  },
  methods: {
    scrollEvent() {
      //当前滚动位置
      let scrollTop = this.$refs.list.scrollTop;
      //此时的开始索引
      this.start = Math.floor(scrollTop / this.itemSize);
      //此时的结束索引
      this.end = this.start + this.visibleCount;
      //此时的偏移量
      this.startOffset = scrollTop - (scrollTop % this.itemSize);
    }
  }
};

```

[点击查看在线DEMO及完整代码](https://codesandbox.io/s/virtuallist-1-rp8pi)

## 关于偏移量

> 在读这篇文章的时候，偏移量这个点让我想了很久，所以这里说明一下

整个布局其实就简单的三个部分

- 外部父级container
- 同级隐藏div撑起滚动条
- 同级可视框

整个布局思路也很简单：监听隐藏div的滚动，每次渲染的数据是通过滑动的高度去计算出来的，也就是只截取一部分

重点是偏移量这个点，看似一眼知，其实很容易被上面的图片所误导，并不容易get到它的点，为什么会有偏移量，因为随着隐藏div的向上滑动，那么整个container就会向上移动，而可视窗（也就是显示数据的div）始终固定在container的顶端，随着container向上移动，可视窗也随之移动，这个时候就需要向下偏移这个可视窗

### 为什么不是scrollTop

为什么偏移量是`scrollTop - (scrollTop % this.itemSize)`而不是`scrollTop`，按正常的思维来说，滑上去了多少就往下偏移多少，这个时候才正好让可视窗在正中间才对，下面有个例子可以亲身体会一下才更能找出其中的问题

[使用`scrollTop`的例子](https://codesandbox.io/s/8frm9?file=/src/components/VirtualList.vue)

我们这里的itemSize是100，假设现在向上滑动的高度是50，而这个时候按照`scrollTop`又向下偏移了50，会出现的现象就是，我们看到的视图没有变化，从感知角度视图并没有向上滑动。

假设我们这个时候向上滑到了100，这个时候数据本身发生了变化，start-end从0-5变成了1-6，但向上滑动0-99这个过程，对我们来说视图都没有发生改变，这时候出现的现象就是我们看到的数据突然发生了变化，视图依然保持不动。

所以偏移量需要用`scrollTop - (scrollTop % this.itemSize)`，算法本身不需要多说，在0-99这个过程中，它计算出的偏移量始终是0，所以我们才能体会到**滑动**这一感官过程，而到了100-199时，它的偏移量始终保持100，所以我们又能体会到在“数据1”这个item之间滑动的过程，这样往复。

