# 实现React瀑布流组件

> “选片”是一个公司的项目，总的来说是在一系列图片中挑选自己喜欢的图片，这里用到了瀑布流，同时也引出我想实现一个瀑布流组件的欲望

## 实现瀑布流

> 项目中使用了外部库[autoresponsive-react](https://github.com/xudafeng/autoresponsive-react)，是一个响应式瀑布流布局组件，可以说弹性是非常的高，[这里](https://xudafeng.github.io/autoresponsive-react/)可以体验一下。但这里不对这个库做研究，只实现一个简单的瀑布流，基本能满足常规使用

### 什么是瀑布流？

如图，这是一个竖版瀑布流，同时也是大家口中的“瀑布流”，算是默认的，还有一种是横版，随便百度一下，切换到图片，那个就是横版，原理都差不多，甚至横版通过flex布局，实现起来更为简单，所以不做讨论

![image-20210707173901428](https://github.com/luoyang233/blog/blob/master/images/waterfall1.png)

### 可能你把它想复杂了

再来看看上面的图片，现在做了一些辅助线

![image-20210707175600375](https://github.com/luoyang233/blog/blob/master/images/waterfall2.png)

#### 思路1：利用计算每次插入最短的一列

获取最短一列，然后插入，这是必不可少的步骤，否则依次插入会出现某一列很长或很短的情况

[codesandbox地址](https://codesandbox.io/s/pubuliuyuanli-qrwmf)

```jsx
import "./styles.css";

export default function App() {
  const source = [100, 200, 300, 100, 200, 400, 200, 100].map((_, i) => ({
    height: _,
    index: i
  }));
  const columns = Array(3)
    .fill("")
    .map(() => ({
      height: 0,
      items: []
    }));

  const getMinHeightColumnIndex = (items) => {
    const mHeight = Math.min(...items.map((t) => t.height));
    const index = items.findIndex((t) => t.height === mHeight);
    return index;
  };

  source.forEach((item) => {
    const index = getMinHeightColumnIndex(columns);
    columns[index].items.push(item);
    columns[index].height += item.height;
  });

  return (
    <div style={{ display: "flex" }}>
      {columns.map((clounm, i) => (
        <div key={i}>
          {clounm.items.map((item, j) => (
            <div
              key={j}
              style={{
                height: item.height,
                width: 200,
                backgroundColor: "red",
                margin: 2
              }}
            >
              {item.index}
            </div>
          ))}
        </div>
      ))}
    </div>
  );
}

```

实际效果如下

![image-20210708115424390](https://github.com/luoyang233/blog/blob/master/images/waterfall3.png)



#### 思路2：CSS3 column 属性

- column-count：指定列数
- column-gap: 设置列之间的间距

利用的是css来实现，但图片会从上到下依次排列，个人认为，在瀑布流场景下并不适用，不符合常规的浏览逻辑，所以[从这里](https://www.runoob.com/try/try.php?filename=trycss3_columns)copy了一段基础代码，了解一下这个属性就好

```html
<style> 
.newspaper
{
	columns:100px 3;
}
</style>

<div class="newspaper">
Lorem ipsum dolor sit amet, consectetuer adipiscing elit, sed diam nonummy nibh euismod tincidunt ut laoreet dolore magna aliquam erat volutpat. Ut wisi enim ad minim veniam, quis nostrud exerci tation ullamcorper suscipit lobortis nisl ut aliquip ex ea commodo consequat.
</div>
```

## 实现瀑布流React组件

> 最初我想的是实现一个最基本的，传入一个url数组prop

于是有了[这个版本](https://codesandbox.io/s/reactpubuliuzujian-q1mvv)，代码如下

```jsx
import "./styles.css";
import { Random } from "mockjs";
import { useEffect, useMemo, useReducer } from "react";

export default function App() {
  const source = Array(10)
    .fill("")
    .map(() =>
      Random.image(
        `${Math.floor(Math.random() * 500)}x${Math.floor(Math.random() * 500)}`,
        "#FF6600"
      )
    );
  return <Waterfall source={source} col={3} width={200} margin={10}/>;
}

const Waterfall = (props) => {
  const { col, source, width } = props;
  const [_, forceUpdate] = useReducer((s) => s + 1, 0);
  const cols = useMemo(
    () =>
      Array(col)
        .fill("")
        .map(() => ({
          height: 0,
          urls: []
        })),
    [col, source]
  );

  const getMinHeightColIndex = (cols) => {
    const minHeight = Math.min(...cols.map((c) => c.height));
    const index = cols.findIndex((c) => c.height === minHeight);
    return index;
  };

  useEffect(() => {
    const img = new Image();
    const layoutImg = (i) => {
      img.src = source[i];
      img.onload = () => {
        const index = getMinHeightColIndex(cols);
        cols[index].urls.push(img.src);
        cols[index].height += img.height * (width / img.width);
        forceUpdate();

        if (i < source.length - 1) {
          layoutImg(i + 1);
        }
      };
    };
    layoutImg(0);
  }, [source, cols, width]);

  return (
    <div style={{ display: "flex" }}>
      {cols.map((col, i) => (
        <div key={i} style={{ marginLeft: 10, width }}>
          {col.urls.map((url) => (
            <img
              src={url}
              key={url}
              style={{ width, objectFit: "contain" }}
              alt="1"
            />
          ))}
        </div>
      ))}
    </div>
  );
};

```

但后来发现一个问题，图片的请求也呈“瀑布流”式`data-fetching`，就导致了一个问题：整个请求过程会很漫长，并且请求一张呈现一张

但如果并发请求，`complete`的图片直接参与计算的话，就无法确保图片是按照顺序布局，所以**下一张图片必须等到上一张图片加载并计算完成**后才能参与计算

所以又了策略**加载并发进行，但计算等待执行**，需要把请求方式稍加改造一下，于是就有了[组件2.0版本](https://codesandbox.io/s/reactpubuliuzujian20-ys1mj?file=/src/App.js)，代码如下

```jsx
import "./styles.css";
import { Random } from "mockjs";
import { useEffect, useMemo, useReducer, useState } from "react";

export default function App() {
  const [col, setCol] = useState(2);
  const [_, change] = useReducer((s) => s + 1, 0);
  const source = useMemo(
    () =>
      Array(10)
        .fill("")
        .map(() =>
          Random.image(
            `${Math.floor(Math.random() * 500)}x${Math.floor(
              Math.random() * 500
            )}`,
            "#FF6600"
          )
        ),
    [_]
  );
  return (
    <>
      <button onClick={() => setCol(col - 1)}>col-1</button>
      <button onClick={() => setCol(col + 1)}>col+1</button>
      <button onClick={change}>change</button>
      <Waterfall source={source} col={col} width={200} margin={10} />
    </>
  );
}

const Waterfall = (props) => {
  const { col, source, width, margin } = props;
  const [_, forceUpdate] = useReducer((s) => s + 1, 0);
  const cols = useMemo(
    () =>
      Array(col)
        .fill("")
        .map(() => ({
          height: 0,
          urls: []
        })),
    [col, source]
  );

  useEffect(() => {
    const getMinHeightColIndex = () => {
      const minHeight = Math.min(...cols.map((c) => c.height));
      const index = cols.findIndex((c) => c.height === minHeight);
      return index;
    };

    const layoutImg = (img) => {
      const index = getMinHeightColIndex();
      cols[index].urls.push(img.src);
      cols[index].height += img.height * (width / img.width);
      forceUpdate();
    };

    let prev;
    source.forEach((url, i) => {
      const img = new Image();
      img.src = url;
      const curr = new Promise((resolve) => {
        let _p = prev;
        img.onload = () => {
          if (i > 0) {
            _p.then(() => {
              layoutImg(img);
              resolve();
            });
          } else {
            layoutImg(img);
            resolve();
          }
        };
      });
      prev = curr;
    });
  }, [source, cols, width]);

  return (
    <div style={{ display: "flex", marginLeft: -margin }}>
      {cols.map((col, i) => (
        <div key={i} style={{ marginLeft: margin, width }}>
          {col.urls.map((url) => (
            <img
              src={url}
              key={url}
              style={{ width, objectFit: "contain" }}
              alt="1"
            />
          ))}
        </div>
      ))}
    </div>
  );
};

```

抛开图片异常情况没有做处理，这已经能满足一个最基本的瀑布流展示的需求，但是实际使用过程中，更多的是用户传入`props.children`，map出每个块，不一定是照片，这对组件来说其实反而更简单，组件内部不需要再管理请求，只需要用户传入每个块的`height`，组件只负责计算，其他的交由用户管理即可

于是有了[组件3.0版本](https://codesandbox.io/s/reactpubuliuzujian20-forked-tj997?file=/src/App.js)

```jsx
import "./styles.css";
import { Random } from "mockjs";
import React, { useMemo, useReducer, useState } from "react";

export default function App() {
  const [col, setCol] = useState(3);
  const [_, change] = useReducer((s) => s + 1, 0);
  const heightArr = useMemo(
    () =>
      Array(10)
        .fill("")
        .map(() => Math.random() * 500),
    [_]
  );
  const source = useMemo(
    () =>
      Array(10)
        .fill("")
        .map((_, i) =>
          Random.image(
            `${Math.floor(Math.random() * 500)}x${heightArr[i]}`,
            "#FF6600"
          )
        ),
    [_]
  );
  return (
    <>
      <button onClick={() => setCol(col - 1)}>col-1</button>
      <button onClick={() => setCol(col + 1)}>col+1</button>
      <button onClick={change}>change</button>
      <Waterfall col={col} width={200} marginH={10} marginV={10}>
        {source.map((url, i) => (
          // <div key={i} style={{ height: heightArr[i], background: "red" }}>
          //   {i}
          // </div>
          <img src={url} key={url} style={{ height: heightArr[i] }} alt="1" />
        ))}
      </Waterfall>
    </>
  );
}

const Waterfall = (props) => {
  const { col, width, marginH, marginV, children } = props;
  const cols = useMemo(
    () =>
      Array(col)
        .fill("")
        .map(() => ({
          height: 0,
          items: []
        })),
    [col, children]
  );
  const getMinHeightCol = () => {
    const minHeight = Math.min(...cols.map((c) => c.height));
    const col = cols.find((c) => c.height === minHeight);
    return col;
  };

  React.Children.forEach(children, (child) => {
    const col = getMinHeightCol();
    col.height += child.props.style.height;
    const isImg = child.type === "img";
    col.items.push(
      React.cloneElement(child, {
        style: {
          ...child.props.style,
          objectFit: "contain",
          width,
          marginTop: marginV,
          height: isImg ? "auto" : child.props.style.height
        }
      })
    );
  });

  return (
    <div style={{ display: "flex", marginLeft: -marginH }}>
      {cols.map((col, i) => (
        <div
          key={i}
          style={{ marginLeft: marginH, marginTop: -marginV, width }}
        >
          {col.items}
        </div>
      ))}
    </div>
  );
};

```

转念一想，其实不管是直接传入url数组，还是map自定义children，在不同场合可能都有需求，所以决定组合一下，[瀑布流4.0](https://codesandbox.io/s/reactpubuliuzujianzhongjiban-o89x0)

```tsx
import React, { FC } from "react";
import WaterfallComp from "./Waterfall.comp";
import WaterfallImg from "./Waterfall.img";
import WaterfallUrl from "./Waterfall.url";

enum Mode {
  Url = "url",
  Img = "img",
  Component = "component"
}

interface Props {
  mode: Mode;
  col: number;
  width: number;
  marginH: number;
  marginV: number;
  source: string[];
}

const WaterfallStrategy = {
  [Mode.Url]: WaterfallUrl,
  [Mode.Img]: WaterfallImg,
  [Mode.Component]: WaterfallComp
};

const Waterfall: FC<Props> = (props) => {
  const {
    mode = Mode.Component,
    marginH = 10,
    marginV = 10,
    children,
    ...others
  } = props;
  const comp = WaterfallStrategy[mode];
  return React.createElement(comp, { marginH, marginV, ...others }, children);
};

export default Waterfall;

```

## 图片加载优化 

> 前面的实现其实是有问题的，如果父组件中的source改变，会触发组件内部重新计算，但前面的图片依然会继续请求。
>
> 想象这么一个场景，一个每页有100张图片的瀑布流页面，用户在第一页还没完全加载完成之前，草草浏览了1.2秒立刻点到了第二页，这个时候第一页的图片请求还在继续请求，那么资源就会被占用，第二页不会立马呈现，需要等到第一页结束最大并发

但图片请求的取消，没有直接有效的办法，或者通过`fetch`等额外方式来达到取消的目的，所以我的思路是每次并发请求一定数量的图片，在每一张图片开始前判断是否有更新，[图片加载优化](https://codesandbox.io/s/reactpubuliuzujianzhongjiban-forked-ribsz?file=/src/Waterfall.url.js)全部代码

下面这段代码截取了一部分，重点在于**利用useEffect闭包对roundRef值的判断**

```jsx
const roundRef = useRef(Symbol());

useEffect(() => {
    roundRef.current = Symbol();
  }, [source, cols]);

useEffect(() => {
    const MAX_CONCURRENT = 5;
    const sourceCopy = [...source];
    const round = roundRef.current;
    const next = () => {
      const abort = round !== roundRef.current;
      if (sourceCopy.length && !abort) {
        const url = sourceCopy.shift();
        const img = new Image();
        img.src = url;
        img.onload = () => {
          layoutImg(img);
          next();
        };
        img.onerror = next;
      }
    };

    for (let i = 0; i < MAX_CONCURRENT; i++) {
      next();
    }
  }, [source, cols]);
```

## 实现图片按需加载

> 假如整个页面有1000张图片，而用户就随便上下翻了翻前面的一部分图片，但按照上面的方式，依然会加载出后面的所有图片，这对服务器来说是不必要的开销，或者某些云服务器按量收费，那。。。

所以我的思路是只加载视窗以内的图片，以及超过视窗的一部分，以作缓冲，随着用户向下滚动，再继续加载剩余的部分图片，[图片按需加载](https://codesandbox.io/s/tupianjiazaiyouhua-ribsz)

下面贴的代码是修改的部分，逻辑不算复杂，再每次`next`时通过`isOverflow`判断是否需要继续加载，而`isOverflow`的判断条件就是**最短的一列的高度是否超过：容器高度+缓冲高度+滚动高度**

最后再监听容器的滚动（可以加个节流，偷懒暂时没写）

```jsx
 useEffect(() => {
    const getMinHeightColIndex = () => {
      const minHeight = Math.min(...cols.map((c) => c.height));
      const index = cols.findIndex((c) => c.height === minHeight);
      return index;
    };

    const layoutImg = (img) => {
      const index = getMinHeightColIndex();
      cols[index].urls.push(img.src);
      cols[index].height += img.width ? img.height * (width / img.width) : 0;
      forceUpdate();
    };

    const isOverflow = () => {
      const minHeight = Math.min(...cols.map((c) => c.height));
      const { clientHeight, scrollTop } = containerRef.current;
      const loadHeight = clientHeight + bufferHeight + scrollTop;
      if (minHeight >= loadHeight) {
        return true;
      } else {
        return false;
      }
    };

    let concurrent = 0;
    const MAX_CONCURRENT = 5;
    const sourceCopy = [...source];
    const round = roundRef.current;
    const next = () => {
      const abort = round !== roundRef.current;
      const load = sourceCopy.length && !abort && !isOverflow();
      if (!load) {
        return;
      }
      if (concurrent >= MAX_CONCURRENT) {
        return;
      }
      concurrent++;
      const url = sourceCopy.shift();
      const img = new Image();
      img.src = url;
      img.onload = () => {
        concurrent--;
        layoutImg(img);
        next();
      };
      img.onerror = () => {
        concurrent--;
        next();
      };
      next();
    };

    next();

    const containerDom = containerRef.current;
    containerDom.addEventListener("scroll", next);
    return () => {
      containerDom.removeEventListener("scroll", next);
    };
  }, [source, cols]);
```