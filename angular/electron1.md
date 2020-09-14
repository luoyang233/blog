## electron+angular环境搭建过程

### 初始化angualr项目

> ng new td-demo --style=less
>
> cd td-demo
>
> npm i

这里我选择的是less，其他一路确认

### 安装electron依赖

> npm install --save-dev electron

安装electron依赖

### 配置electron文件

- 在src目录下建立一个`el-main .ts`的启动文件（因为本身存在`main.ts`的angular文件）
- 在`el-main.ts`中拷贝以下代码做测试用

```javascript
const {app, BrowserWindow} = require('electron')
    const url = require("url");
    const path = require("path");

    let mainWindow

    function createWindow () {
      mainWindow = new BrowserWindow({
        width: 800,
        height: 600,
        webPreferences: {
          nodeIntegration: true
        }
      })

     win.loadFile(path.join(__dirname, `/dist/index.html`));
      
      // Open the DevTools.
      mainWindow.webContents.openDevTools()

      mainWindow.on('closed', function () {
        mainWindow = null
      })
    }

    app.on('ready', createWindow)

    app.on('window-all-closed', function () {
      if (process.platform !== 'darwin') app.quit()
    })

    app.on('activate', function () {
      if (mainWindow === null) createWindow()
    })

```

- 在根目录下的`package.json`文件下写入运行命令`start:electron`
- 写入`main`启动文件的位置

```
 {
      "name": "td-demo",
      "version": "0.0.0",
      "main": "src/el-main.ts",
      "scripts": {
        "ng": "ng",
        "start": "ng serve",
        "build": "ng build",
        "test": "ng test",
        "lint": "ng lint",
        "e2e": "ng e2e",
        "start:electron": "ng build --base-href ./ && electron ."
      }, 
      // [...]
    }
```

完成到这里，从以上代码中可以明白大体构建思路如下

1. 首先运行`ng build`将构建后的代码放入dist文件中
2. electron运行后直接读取dist文件中的内容

### 等等 还没完

运行后我们可以成功的打开GUI页面，但却是一片空白，控制台中也有错误提示

```
Not allowed to load local resource: file:///Users/luoyang/Code/td-demo/src/dist/index.html
```

所以很简单，`el-main.ts`文件没有找的`index.html`文件，注意看一下错误中的目录结构和自己的目录结构，很容易找出问题

- angular输出后的内容放在了dist/td-demo文件夹下了
- electron又是直接去找的dist文件夹下的`.hmtl`文件

所以改一下`el-main.ts`中的`.hmtl`的文件地址就行了

```javascript
// 这里是更改后的地址，加了一个/td-demo
win.loadFile(path.join(__dirname, `/dist/td-demo/index.html`));
```

### 关于热更新

完成了以上步骤可以开始开发，但并不贴合实际的开发过程

electron每次是去读取的dist输出目录的东西，也就是说需要在每次更改后去build，并重新加载，这样做太麻烦了

electron的桌面有两种加载方式

```javascript
 // 直接读取dist文件
  win.loadFile(path.join(__dirname, `/dist/td-demo/index.html`));

  // 监听端口
  win.loadURL('http://localhost:4200');
```

- 一种是直接读取dist文件，这也是我们上面的加载的方式
- 另一种是直接监听端口，也就是相当于直接把网页上的内容映射下来

因为我们选用的技术栈是angular，所以不需要自己去配置热更，这一点很方便，所以我们接下来要做的就是像往常一样，直接在网页中开发angular就行了，代码变更后angular自动刷新，接着electron读取到端口变更后的内容，自动就会映射在桌面端上面

所以我们在开发过程中只需要采用监听端口的形式就可以了，在`main.ts`文件中做以下更改

```javascript
	// 将加载文件的方式注释了
  // win.loadFile(path.join(__dirname, `/dist/td-demo/index.html`));

  // 开发时监听端口
  win.loadURL('http://localhost:4200');
```

接下来要做的可以理解为我们在开发一个angular应用，然后electron的桌面端只是一面镜子，同步映射网页的内容而已（暂时粗俗的这样理解吧）

所以我们要做的就是，在正常的`ng serve`的基础上多开一个electron的监听

```json
//package.json

"start": "ng serve && electron ."
```

如果就如上面的方式去运行，会出现一些问题，当`ng serve`运行监听后会阻塞后面的执行，导致`electron .`无法执行，所以这里引入了一个npm包`npm install concurrently --save-dev`

```json
//package.json

"start": "concurrently 'ng serve' 'electron .'",
```

这样就能一次性执行两个监听了

> P.S.这里还是有点问题，初次加载`ng serve`比`electron .`慢，所以electron桌面会先加载出来并且一片空白，等ng加载完成后需要手动去刷新一下，暂时未解决。。

接下来就可以按照开发angualr同样的方式去开发electron了