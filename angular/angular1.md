# 初识Angular(一)

## 模版语法

- **ngFor*：*开头的都是结构型指令，可以重塑DOM结构

```html
<!--> 
结果就是div被遍历了n次 
注意这里的 let product of products并不是一个字符串，而是表达式
<-->

<div *ngFor="let product of products">
</div>
```

- 插值语法{{ }}

```html
<div *ngFor="let product of products">
  <h3>
      {{ product.name }}
  </h3>
</div>
```

- 属性绑定语法[ ]

```html
<!-->这里的title属性用[]包裹，也就是为了让其在属性中能够使用属性值<-->

<div *ngFor="let product of products">
  <h3>
    <a [title]="product.name + ' details'">
      {{product.name}}
    </a>
  </h3>
</div>
```

- **ngIf*指令

```html
<!-->
前面说了*开头的是结构型指令
所以这里的*ngIf很好理解，就是在product.description有值的时候做渲染，否则不创建P节点和其内部节点
<-->

<div *ngFor="let product of products">

  <h3>
    <a [title]="product.name + ' details'">
      {{ product.name }}
    </a>
  </h3>

  <p *ngIf="product.description">
    Description: {{ product.description }}
  </p>

</div>
```

- 事件绑定 ( )

```html
<!-->
事件绑定通过()将属性包裹，并通过"share()"调用，这里的share只是一个函数名
看到这里基本上明白，无论是属性绑定或是事件绑定，都是通过在“”双引号中完成，这里的双引号就类似与react jsx语法中使用{}
<-->

<div *ngFor="let product of products">

  <h3>
    <a [title]="product.name + ' details'">
      {{ product.name }}
    </a>
  </h3>

  <p *ngIf="product.description">
    Description: {{ product.description }}
  </p>

  <button (click)="share()">
    Share
  </button>

</div>
```

## 组件

1. 首先创建一个文件夹，其内部包含css+html+ts文件，结构如下

   ```
   > product-alerts
   	> product-alerts.component.css
   	> product-alerts.component.html
   	> product-alerts.component.ts
   ```

   - 那么可以说product-alerts文件就是一个完整的组件

2. ts文件中的代码如下

```typescript
import { Component, OnInit } from '@angular/core';
//这里为手动引入
import { Input } from '@angular/core';

@Component({
  //注意这句
  selector: 'app-product-alerts',
  templateUrl: './product-alerts.component.html',
  styleUrls: ['./product-alerts.component.css']
})
export class ProductAlertsComponent implements OnInit {
  //这里为另添加的代码
  @Input() product;
  constructor() { }

  ngOnInit() {
  }

}
```

- 装饰器中的`templateUrl`、`styleUrls`很好理解不用多说，注意这里的`selector`，按照目前我的理解就是一个对外暴露的组件名，某个地方要引用这个组件，就直接使用<app-product-alerts>
- `@Input() product;`按照我目前的理解，就类似于react中的props，即上层组件传来的参数`product`，而`@Input()`就是使用这个参数的方式，后面会说到具体怎么传参

3. 现在在上层组件中去使用这个组件

```html
<!-->这是一个上层组件<-->

<button (click)="share()">
  Share
</button>

<app-product-alerts
  [product]="product">
</app-product-alerts>
```

- 这里的`app-product-alerts`就是我们在组件的装饰器中定义的 `selector: 'app-product-alerts'`
- `[product]="product"`通过[ ]的方式向该组件传入参数

4. 现在如果是下层调用上层传入的函数，则不能像react那样直接调用，必须（好像是必须？）通过事件机制完成调用，所以这里也不能叫做上层传入函数，而是下层发送命令，上层接受到命令出发监听

   ```typescript
   // 这是上层组件 .ts文件
   // 这里定义了一个函数onNotify
   
   export class ProductListComponent {
     products = products;
     
    	share() {
       window.alert('The product has been shared!');
     }
     
     onNotify() {
       window.alert('You will be notified when the product goes on sale');
     }
   }
   ```

   ```html
   <!-->这是上层组件 .html文件<-->
   <button (click)="share()">
     Share
   </button>
   
   <app-product-alerts
     [product]="product" 
     (notify)="onNotify()">
   </app-product-alerts>
   ```

   - 这里通过之前提到的( )括号绑定方式向下传递函数

   

   接着来看子组件中如何使用

   ```typescript
   // 这是子组件的.ts文件
   
   import { Component } from '@angular/core';
   import { Input } from '@angular/core';
   import { Output, EventEmitter } from '@angular/core';
   
   export class ProductAlertsComponent {
     @Input() product;
     @Output() notify = new EventEmitter();
   }
   ```

   - 在子组件中，是发出一个消息，使上层组件接受到继而触发监听函数，所以这里是Output，代表输出，notify参数为一个事件发射器的实例

   

   ```html
   <!-->这是子组件 .html文件<-->
   
   <p *ngIf="product.price > 700">
     <button (click)="notify.emit()">Notify Me</button>
   </p>
   ```

   - 这里就是触发实例中的发射函数emit继而触发上层监听