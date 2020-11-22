## 可观察对象（Observable）的基本用法

> rxjs在Angular的项目中基本替代了promise，一直以来对Observable和Subject的概念和用法都有点傻傻分不清楚，准备来捋一捋

### 观察者（Observer）

这里的观察者也就是观察者模式，**声明式的**，也就是说当有人订阅他的时候才会去执行，可以理解为送报纸，没定怎么可能给你送过来，当你订阅了之后，才会去执行生产报纸，再送报纸（执行被订阅函数）这个过程。

同样，我们也不需要去关心报纸是怎么生产的，是怎么运送到我们手中的，我们只需要等他送过来拿来看就行了（得到订阅的值，直接拿来用就行了）

当然我们也可以随时取消订阅，订报纸也是一样

### 基本用法

首先得有个可观察对象，也就是得有个报纸厂才行

```javascript
const paperFactory = new Observable();
```

接下来我们就订阅这个对象，也就是定报纸了

```javascript
const paperFactory = new Observable();
paperFactory.subscribe();
```

等等，光有个报纸厂还不行，我们肯定是要生产报纸的

```javascript
// observer 可以理解送报员吧
// process 这个函数就是一整套生产配送流程了
const process = (observer) => {
	try {
		// 。。。 这里执行了一系列的函数
		// 。。。 也就是执行了一系列的工艺
		const paper = '这是一份报纸'
		observer.next(paper);
	} catch (e) {
		observer.error('生产过程出了些问题')
	}

	return {
		unsubscribe() {
			console.log('我要取消订阅！');
		}
	}
}
```

- 上面的`observer.next(paper)`也就是将报纸送达客户手中的这个过程了，订阅者得到的就是这个paper
- 返回的是一个取消订阅函数，可以订阅当然也必须可以取消

| 通知类型 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| next     | 必要。用来处理每个送达值。在开始执行后可能执行零次或多次。   |
| error    | 可选。用来处理错误通知。错误会中断这个可观察对象实例的执行过程。 |
| complete | 可选。用来处理执行完毕（complete）通知。当执行完毕后，这些值就会继续传给下一个处理器。 |

#### 订阅

```javascript
const process = ()=>{}; // 具体代码在上面
const paperFactory = new Observable(process);
paperFactory.subscribe();
```

现在报纸生产好了，我们需要去订阅他了

⚠️注意这个说法可能有些错误，只是为了方便理解，因为在文章的开头说了，当有订阅时，被订阅函数才会实际去执行，换句话说，当我们**先**定了报纸后，厂家才会**后**去生产报纸，送到我们手上。

```javascript
const process = () => { }; // 具体代码在上面
const paperFactory = new Observable(process);
const readPaper = (paper) => { console.log('这份报纸的内容是:' + paper) }
paperFactory.subscribe(readPaper);
```

`readPaper`也可以是一个对象，分别包含三种情况的处理方式，当然不需要全部写出来

```javascript
const readPaper = {
  next(paper){},
  error(e){},
  complete(){}
};
```

同时，也可以分别传入参数

```javascript
const nextFn = () => {};
const errFn = () => {};
const completeFn = () => {};
paperFactory.subscribe(nextFn,errFn,completeFn);
```

## 多播

假如出现了第二个订阅者，如果还是以前面的方式进行发布，那么就会出现这种现象，订阅者订阅一次，被订阅函数就执行一次（每个订阅者得到的对象都是相互独立的，这在有些时候可以这样做），换成前面的例子，也就是说只要出现一个定报纸的人，那工厂马上就要印一份送到用户手中，那工厂还干不干？

```javascript
let count = 1
// 这里把process稍做修改
const process = (observer) => {
		const paper = '这是一份报纸'
		observer.next(paper);
  	// 内部增加一行count++其他不变
  	count++;
}

paperFactory.subscribe(paper => console.log(count + paper));
// 过了0.5秒再触发
setTimeout(() => {
  paperFactory.subscribe(paper => console.log(count + paper));
}, 500);

// Logs:
// （立即打印）：1 这是一份报纸
// （0.5秒后打印）：2 这是一份报纸
```

以上，说明process每次都会被运行

所以不妨尝试换一个思路，我报纸厂就每天早中晚定时生产3批，无论何时订阅，都不会立即生产，都在下一个批次的时候收到报纸

⚠️当然这里有个bug，就是订阅后接下来的每一批生产出来的都会被订阅者收到，暂且理解为当天每个不同时间段发生的新闻吧，报纸内容会有所不同，所以当然后面的每一批都要发给订阅者

```javascript
import { Observable } from "rxjs";
function sequenceProcess() {
  const batch = ["早", "中", "晚"];
  const customers = [];
	let count = 0;
  
  return (customer) => {
    customers.push(customer);
    // 收到一个订阅的时候开始生产
    if (customers.length === 1) {
      setInterval(() => {
        customers.forEach((_customer) => {
          _customer.next(batch[count]);
        });
        count++;
      }, 1000);
    }
  };
}

// 订阅
const paperFactory = new Observable(sequenceProcess());
paperFactory.subscribe((paper) => console.log("第一个用户：", paper));
setTimeout(() => {
  paperFactory.subscribe((paper) => console.log("第二个用户：", paper));
}, 1500);

// Logs:
// 开始：第一个用户：早
// 第二秒：第一个用户：中
// 第二秒：第二个用户：中
// 第三秒：第一个用户：晚
// 第三秒：第二个用户：晚
```

⚠️上面的代码为了更直观的体现多播的核心思想，缺少了取消监听和取消计时器等其他细节

### Observable和Subject的区别

回到最开始的问题，实际上就是上面提到的多播，是rxjs库提供的一套处理多播的逻辑，具体使用方式参考[rxjs文档](https://cn.rx.js.org/)

这里再补充一个`BehaviorSubject`，平时开发中用得最多的就这三个，用前面的例子来讲和`Subject`的区别就是，在订阅后`Subject`获得的是下一批次的报纸，而`BehaviorSubject`在等待后面批次的同时，**能立即获得最近一批已经生产好了的报纸**，换句话说也就是`BehaviorSubject`**能立即获得当前的值**

