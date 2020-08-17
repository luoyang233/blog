# 理解表单数据流

> 看得似懂非懂之际，介绍一下我所理解的数据流

## 响应式

### 从视图到模型

>官方是这么介绍的(粗略浏览一下，下面有分析)：
>
>1. 最终用户在输入框元素中键入了一个值，这里是 "Blue"。
>2. 这个输入框元素会发出一个带有最新值的 "input" 事件。
>3. 这个控件值访问器 `ControlValueAccessor` 会监听表单输入框元素上的事件，并立即把新值传给 `FormControl` 实例。
>4. `FormControl` 实例会通过 `valueChanges` 这个可观察对象发出这个新值。
>5. `valueChanges` 的任何一个订阅者都会收到这个新值。

挨着来分析吧

1. 输入blue，完了
2. 也就是input框触发事件，顺便带上值
3. - 访问控制器这里说一下，就是一个连接FormCtrol实例和DOM之间的桥梁，暂且理解为通信用吧
   - 这个桥梁监听到了input发出的事件，把它的值传给了桥另一端的`FormControl` 实例
4. - 实例内部有个setValue方法，更新了value值
   - 实例内部的有个可观察对象valueChanges，也会得到新的value值
5. 注册了valueChanges的对象，都会得到最新的value

总结一下过程就是：

**输入input => 触发事件 => 实例获取且更改值 => 注册对象跟着变**

### 从模型到视图

> 官方是这么介绍的：
>
> 1. `favoriteColorControl.setValue()` 方法被调用，它会更新这个 `FormControl` 的值。
> 2. `FormControl` 实例会通过 `valueChanges` 这个可观察对象发出新值。
> 3. `valueChanges` 的任何订阅者都会收到这个新值。
> 4. 该表单输入框元素上的控件值访问器会把控件更新为这个新值。

挨着来分析吧

1. 调用实例的setValue方法更改值
2. 第二步+第三步，注册了valueChanges的对象，都会得到最新的value
3. 访问控制器去更新input那边的值

总结一下过程就是：

**更新value => 注册对象跟着变 => input变**

## 模板驱动表单

### 从视图到模型

> 官方是这么介绍的：
>
> 1. 最终用户在输入框元素中敲 "Blue"。
> 2. 该输入框元素会发出一个 "input" 事件，带着值 "Blue"。
> 3. 附着在该输入框上的控件值访问器会触发 `FormControl` 实例上的 `setValue()` 方法。
> 4. `FormControl` 实例通过 `valueChanges` 这个可观察对象发出新值。
> 5. `valueChanges` 的任何订阅者都会收到新值。
> 6. 控件值访问器 `ControlValueAccessory` 还会调用 `NgModel.viewToModelUpdate()` 方法，它会发出一个 `ngModelChange` 事件。
> 7. 由于该组件模板双向数据绑定到了 `favoriteColor`，组件中的 `favoriteColor` 属性就会修改为 `ngModelChange` 事件所发出的值（"Blue"）。

挨着来分析吧

1. 输入blue，完了
2. 也就是input框触发事件，顺便带上值
3. 访问控制器把input的值传给`FormControl` 的实例，实例内部的setValue方法更新了value值
4. 实例内部的有个可观察对象valueChanges，也会得到新的value值
5. ⚠️注意上面的第6步，这里的访问控制器做了两件事，一个刚刚说的传值给实例，另外一个就是调用`NgModel`指令内部的一个方法（不管他叫什么方法，想看的话，上面有），它会发出一个 `ngModelChange` 事件
6. 因为双向绑定，所以模型就收到且更改为最新的value值

总结一下过程就是(方便理解，暂且认为两条线同时进行)：

​									**=> 实例获取且更改值 => 注册对象跟着变**

**输入input => 触发事件 **

​									**=> 触发ngModelChange => 更新值**

### 从模型到视图

> 官方是这么介绍的：
>
> 1. 组件中修改了 `favoriteColor` 的值。
> 2. 变更检测开始。
> 3. 在变更检测期间，由于这些输入框之一的值发生了变化，Angular 就会调用 `NgModel` 指令上的 `ngOnChanges` 生命周期钩子。
> 4. `ngOnChanges()` 方法会把一个异步任务排入队列，以设置内部 `FormControl` 实例的值。
> 5. 变更检测完成。
> 6. 在下一个检测周期，用来为 `FormControl` 实例赋值的任务就会执行。
> 7. `FormControl` 实例通过可观察对象 `valueChanges` 发出最新值。
> 8. `valueChanges` 的任何订阅者都会收到这个新值。
> 9. 控件值访问器 `ControlValueAccessor` 会使用 `favoriteColor` 的最新值来修改表单的输入框元素。

挨着来分析吧

1. 模型中更改了值（这里没有绑定实例，所以是直接更改）
2. 触发`NgModel`指令中的勾子函数`ngOnChanges`
3. 异步更新它自己内部的`FormControl`实例
4. 可观察对象`valueChanges`的订阅者会收到新值
5. 访问控制器去修改input

总结一下过程就是：

**更新value => 触发NgModel勾子 => 更新内部实例 => 注册对象跟着变 => input变**

## 区别总结

现在把四个流程拿来对比

> 响应式视图到模型：
>
> **输入input => 触发事件 => 实例获取且更改值 => 注册对象跟着变**
>
> 模版驱动视图到模型：
>
> ​									**=> 实例获取且更改值 => 注册对象跟着变**
>
> **输入input => 触发事件 **
>
> ​									**=> 触发ngModelChange => 更新值**

可以看出区别其实就是模版驱动多了一个触发ngModelChange和更新值，这里还有一点为了方便区别没有表现出来，就是他的实例是它内部维护的，我们并不能得到这个实例对象，对我们来说它在 FormControl之外多包了一层ngModel（暂且这样理解吧，我也不知道对不对）

再来看接下来的对比

> 响应式模型到视图：
>
> **更新value => 注册对象跟着变 => input变**
>
> 模版驱动模型到视图：
>
> **更新value => 触发NgModel勾子 => 更新内部实例 => 注册对象跟着变 => input变**

这里为了方便对比，有一点也没有表现出来，响应式更新value是通过实例的setValue方法去更新，而模版驱动则是直接去更改。这里也可以看出中间多了两步，就是触发NgModel勾子和更新内部实例，也就更加印证了前面所说的，内部实例是由它自己维护，对我们来说暴露出来的只是外层的NgModel