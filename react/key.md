# 别用index作为key

之前在学习react的时候看过关于key的一些注意事项，官网的意思就是在万不得已的时候不要用索引来当作key，有时候图省事不以为然，直到今天踩了个坑才意识到别用index当key！别用index当key！真别用index当key！

## 来源

旁边的小伙伴遇到一个bug，这个bug是这么一个情况，这里有一个上传文件到服务器的功能，一开始是这样的

![key1.png](https://github.com/luoyang233/blog/blob/master/images/key1.png)

然后上传了一张图片，中间出现了奇怪的闪烁，有那么一丢丢的时间，前面71开头的图片变成了word文档的图标？？然后新上传的图片变成了前一张图片的缩略图？？最后一个又成空白了？？？

![key2.png](https://github.com/luoyang233/blog/blob/master/images/key2.png)

后来变成了他该有的样子样子

![key3.png](https://github.com/luoyang233/blog/blob/master/images/key3.png)

虽然中间过程很短，但既然被测试发现了，那它就是bug了（看不见就算了）

## 问题分析

这个bug大概就是上传后刷新的过程中，前面的图标被后面一个图标所代替，然后经历了极短的时间又恢复到他该有的样子

### 看下代码

结构大概是这么一个样子

``` jsx
foo=(value,index)=>{
  return <div key={index}>
    <img src={'...'}/>
  </div>
}

render(){
  <Parents>
     xxx.map((value,index)=>foo(value,iundex))
  </Parents>
}
```

+ **会不会是代码在其他的函数中被刷新了，多次render导致？**

  没有！
  屏蔽了其他代码在render一次的情况下问题依然存在
  这就有点迷了。。

+ **一切解释不过的都归咎到浏览器的问题**

  一定是浏览器缓存导致的
  那我换safari吧！
  还是这样？？
  这就更迷了。。

+ **会不会是diff的问题**

  因为返回的整个结构未变化，diff过程中不会认为我后面加入的那个还是前面那个东西吧？然后就返回了一坨和之前一样的dom，图片资源是在浏览器渲染完成后去获取的，在获取完成后更替了之前的图片。

  这样就说得通了，在图片加载比较慢的情况下会持续显示前一张图片，直到图片下载完成

  于是我给新加入的图片套了一层div来试一下

  还真tm的没问题了！

  在套多套了一层div之后，就没出现这个问题了，因为这次返回的结构和上次不一样了

## kill it

+ **怎么肥事呢？**

  为什么diff会认为我新加入的就是之前的？（说得不是很直观，反正就是那个意思）

  再次细细一看代码

  ```jsp
  key={index}
  ```

  好了，你就是凶手

+ **为什么key=index会导致这个问题？**

  因为react没有感情！

  他咋知道你给他的是新加入的东西啊？都长得一样，你又没给它说

  不是加了个key=index的吗

  这里就要细细的想了，直接用index有什么问题

  分析一下过程

  最先的图片我们假设他为p0,p1,p2吧，他们的key分别对应0，1，2，这个时候我加入了一张图片，于是图片顺序变成了p3,p0,p1,p2，这个时候的key呢

  > 0，1，2，3

  现在懂了吧？对diff来说p2才是新加入的，而本身新加入的p3这个时候key为0，对diff来说其实就是原来的那个东西，也就是p0，有点绕，说白点就是，diff把p3认成了p0

  所以对virtual dom来说的变更的其实只是属性src，顺便最后再添加了个东西

  这里还有个react官网上的[例子](https://react.docschina.org/redirect-to-codepen/reconciliation/index-used-as-key)能直接体验

+ **为什么key一定要保持唯一且不变**

  好了，现在我们换成uid来作为key，还是p0,p1,p2，这个时候key变成了abc1,def2,ghi3，然后新入了一张图片p3，图片顺序变成了p3,p0,p1,p2,这个时候的key呢

  > jkl3,abc0,def1,ghi2

  这个时候没有感情的diff不就明白了

+ **什么情况下可以用index**

  1.顺序稳定

  2.最好什么时候都不用

