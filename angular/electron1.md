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

      mainWindow.loadURL(
        url.format({
          pathname: path.join(__dirname, `/dist/index.html`),
          protocol: "file:",
          slashes: true
        })
      );
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
mainWindow.loadURL(
    url.format({
      // 这里是更改后的地址，加了一个/td-demo
      pathname: path.join(__dirname, `../dist/td-demo/index.html`),
      protocol: "file:",
      slashes: true
    })
  );
```

