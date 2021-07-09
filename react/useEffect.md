> 实现一个SolidJS版本的useState和useEffect
>
> 参考卡颂老师[文章](https://mp.weixin.qq.com/s/ghpJf2-qxD4nXjek5dqLHA)，但并不是copy

## 和React的区别

```jsx
// React
const [state,setState]  =useState(0)
useEffect(()=>{
  console.log(state)
},[state])

// SolidJS版本
// 真正的SolidJS中不叫useState和useEffect，但用法如此
const [state,setState]  =useState(0)
// 不需要依赖state
// 函数获取state
useEffect(()=>{
  console.log(state())
})
```

## 实现基础版useState

没啥好说的，唯一的区别就是函数获取value

```jsx
function useState(value) {
  const getter = () => value;
  const setter = (newValue) => value = newValue;
  
  return [getter, setter];
}
```

## 实现基础版useEffect

第一次会立即执行，是的，按我目前的理解暂时就这么简单

```jsx
function useEffect(callback) {
  callback()
}
```

## 如何实现无依赖执行

**发布订阅！**

函数获取value的作用就来了

- getter时注册当前正在执行的callback
- setter时执行effect栈，也就是执行所有被注册的callback

现在有这么一段代码

```jsx
const [s1,setS1]  =useState(0)
const [s2,setS2]  =useState(0)
const [s3,setS3]  =useState(0)

useEffect(()=>{
  console.log(s1())
  console.log(s2())
})

useEffect(()=>{
  console.log(s1())
  console.log(s3())
})
```

我们想要的结果是，`setS1()`时，执行两个effect，`setS2`时只执行第一个effect

**所以我们需要一个对象来保存所有的订阅，存入每个useState中**

```jsx
function useState(value) {
  // 订阅列表
  // 想象一下effect之中调用两次getter,所以这里用了Set
  const subs = new Set();
  
  const getter = () => {
    // 在获取value时，如果执行环境时effect中，我们需要保存这个effect
    // 。。。
    // 所以应该做点什么
    return value
  };
  const setter = (newValue) => value = newValue;
  
  return [getter, setter];
}
```

如何确定当前执行的环境是effect之中呢

并且如何获取到这个正在执行的effect callback呢

**执行callback的时候保存下来！**

```jsx
// 正在执行的callback栈
const effectStack = []

function useEffect(callback) {
  // 执行前入栈
  effectStack.push(callback)
  callback()
  // 执行后出栈
  effectStack.pop()
}
```

这个时候执行到callback了，callback中又调用了getter，所以现在来修改一下getter函数

```jsx
function useState(value) {
  const subs = new Set();
  
  const getter = () => {
    // 这个时候effectStack的栈顶就是正在执行的effect callback
    const callback = effectStack[effectStack.length - 1];
    // 有effect说明执行环境就是在effect之中
    if (callback) {
      subs.add(callback)
    }
    return value
  };

  const setter = (newValue) => value = newValue;

  return [getter, setter];
}
```

## setter时如何执行订阅

现在轮到setter时执行被注册的effect了

```jsx
function useState(value) {
  const subs = new Set();
  
  const getter = () => {
    const callback = effectStack[effectStack.length - 1];
    if (callback) {
      subs.add(callback)
    }
    return value
  };
  
  const setter = (newValue) =>{
    value = newValue;
    Array.from(subs).forEach(callback=>callback())
  } 
  
  return [getter, setter];
}
```

## 检验一下

```jsx
export default function App() {
  const [s1, setS1] = useState("s1");
  const [s2, setS2] = useState("s2");
  const [s3, setS3] = useState("s3");

  useEffect(() => {
    console.log(s1());
    console.log(s2());
  });

  useEffect(() => {
    console.log(s1());
    console.log(s3());
  });

  return (
    <div className="App">
      <button onClick={() => setS1("s1变了")}>setS1</button>
      <button onClick={() => setS2("s2变了")}>setS2</button>
      <button onClick={() => setS3("s3变了")}>setS3</button>
    </div>
  );
}

// 初始化时打印
// s1
// s2
// s1
// s3

// 点击Button setS1时
// s1变了
// s2
// s1变了
// s3

// 点击Button setS2
// s1变了
// s2变了
```

成了！

## 实现自动跟踪依赖

现在来修改一下执行

```jsx
export default function App() {
  const [s1, setS1] = useState("s1");
  const [s3, setS3] = useState("s3");
  // 增加一个执行条件
  const [status, setStatus] = useState(true);

  useEffect(() => {
    console.log(s1());
    // 这里做了点改动
    if (status()) {
      console.log(s3());
    }
  });

  return (
    <div className="App">
      <button onClick={() => setS1("s1变了")}>setS1</button>
      <button onClick={() => setS3("s3变了")}>setS3</button>
      <button onClick={() => setStatus(false)}>false</button>
    </div>
  );
}
```

我们想要的效果是当`status`置为false时，显然会执行一次effect，但这里不再执行if中的s3，所以这个effect也就不再依赖s3，所以我们想要的效果就是**status变为false后，下一次修改s3时effect不再执行**

再来看下之前的代码

```jsx
function useState(value) {
  // effect的订阅一直存在于subs之中
  // 显然在下一次修改value后，之前的effect订阅同样会执行
  const subs = new Set();
  
  const getter = () => {
    const callback = effectStack[effectStack.length - 1];
    if (callback) {
      subs.add(callback)
    }
    return value
  };
  
  const setter = (newValue) =>{
    value = newValue;
    Array.from(subs).forEach(callback=>callback())
  } 
  
  return [getter, setter];
}
```

所以我们需要做清理工作，清理掉不再需要的effect订阅

问题来了，`subs`存在于函数的内部之中，并且effect不再依赖该state时，getter也不会被调用，所以我们也没有调用内部函数的机会，那么清理的工作肯定放在了外面，如何在外面清理内部的`subs`？

**引用！**

我们在effect每次调用的时候，清理掉所有的依赖，并且在执行时，重新构建依赖，这样就能确保每次effect调用时“依赖（动词）需要的依赖（名词）”，所以在effect内部我们需要引用useState中的`subs`，以便在不需要时清理他

```jsx
function useState(value) {
  const subs = new Set();
  
  const getter = () => {
    // 之前这里叫callback，现在改成了effect
    // 在useState函数中，我们既要让effect引用到subs，同时又要调用effect的callback
    // 所以effect的结构应为{excute,deps}
    // excute后面再说，暂时当成callback
  	// 之所以叫deps，不叫subs，是因为一个effect可能依赖很多个subs
    const effect = effectStack[effectStack.length - 1];
    if (effect) {
      subs.add(effect)
      effect.deps.add(subs)
    }
    return value
  };
  
  const setter = (newValue) =>{
    value = newValue;
    Array.from(subs).forEach(sub=>sub.excute())
  } 
  
  return [getter, setter];
}
```

结下来是运行effect的时候的清理和构建工作

```jsx
const effectStack = []

function useEffect(callback) {
  // 这里也就是上面提到的excute
  // 因为getter调用时，需要执行的不单单是callback，还有其他工作，所以一起打包在这个函数中
   const excute = () => {
    // 开始时清理所有依赖
    cleanup(effect);
    effectStack.push(effect);
    // 构建其实就已经放在了前面的getter中
    callback();
    effectStack.pop();
  };
  const effect = {
    excute,
    deps: new Set()
  };
  effect.excute();
}

function cleanup(effect) {
  // 清理所有state的subs中的当前effect
 	Array.from(effect.deps).forEach(subs=>{
   	subs.delete(effect)
 	})
  // 清理当前effect中所有的依赖
  effect.deps.clear()
}
```

所以就有了这张依赖图

![key1.png](https://github.com/luoyang233/blog/blob/master/images/useEffect_1.png)

## 测试一下

```jsx
export default function App() {
  const [s1, setS1] = useState("s1");
  const [s3, setS3] = useState("s3");
  const [status, setStatus] = useState(true);

  useEffect(() => {
    console.log(s1());
    if (status()) {
      console.log(s3());
    }
  });

  return (
    <div className="App">
      <button onClick={() => setS1("s1变了")}>setS1</button>
      <button onClick={() => setS3("s3变了")}>setS3</button>
      <button onClick={() => setStatus(false)}>false</button>
    </div>
  );
}

// 点击setS3 打印
// s1
// s3变了

// 点击false 打印
// s1

// 点击setS1 打印
// s1变了

// 再次点击setS3 打印
// 无任何输出
```

**成了！**

## 扩展一下 实现一个useMemo

还是首先实现一个最基础版本

```jsx
function useMemo(callback) {
 	return callback()
}
```

这显然不够，useMemo的效果是，当callback中的依赖变化时，callback自动被调用，否则直接返回上一次结果，所以我们就需要记住上一次的结果

```jsx
function useMemo(callback) {
  const [v,set] = useState(callback())
 	return v()
}
```

这还不够，这里只记住了第一次运行后的结果，并未实现依赖变化时重新运行callback导出新的结果

```jsx
function useMemo(callback) {
  const [v,set] = useState()
  useEffect(()=>set(callback()))
 	return v
}
```

**完成！**

