## electron+angular环境搭建(二)

[electron+angular环境搭建（一）](https://github.com/luoyang233/blog/blob/master/angular/electron1.md)

> 在经过一段时间的摸索，基本上对整个对electron开发整个脉络有一定的认识了，这是在上一篇文章的基础上继续衍生出的一些东西，主要是对开发流程做一个清晰的介绍
>
> [参考了这篇文章](http://www.voidcn.com/article/p-vqoylvwi-bnu.html)，项目结构有一定出入

### 项目目录结构

```json
├── dist //项目输出目录
├── installer //先不管这是啥
├── main-process
├── src //angular主目录
└── electron-main.ts //electron主文件(原本为main，这里更名了一下)
├── package.json
├── tsconfig.main.json //主进程tsc配置文件
```

### 设置主进程监听

下面是`tsconfig.main.json`的配置

```json
{
	"extends": "./tsconfig.base.json",
	"compilerOptions": {
			"outDir": "./dist",  // 打包目录
			"baseUrl": "./",
			"module": "commonjs",
			"typeRoots": [
					"../node_modules/@types"  // 引入 .d.ts 文件
			],
    	"types": [
					"node"
			],
	},
	"include": [
			"electron-main.ts",
			"main-process/**/*.ts", // 引入其余主进程文件
	],
	"target": "electron-main",
}

```

### 开启tsc监听模式

```
tsc -p ./tsconfig.main.json -w
```

在运行改命令以后，就会启用tsc的监听模式，注意上面的配置文件中的`include`字段中的文件，一旦这些文件有变更后，tsc将会自动运行编译打包命令，将编译的结果输出至`outDir`设置的目录当中，这里是直接输出在了dist目录

### 自动重启进程

在保持上面的服务运行的情况下，我们只需要通过electron命令运行dist目录中的主进程main文件（这里是`electron-main.ts`）即可

注意环境变量，因为现在我们做的依然是开发，并不是真的打包，所以依然要去main文件中依然要去监听angular端口，而不是直接读取angular的打包文件

做到了这一步并没有真正的成功，因为在开发中会发现，虽然dist目录中的主进程文件每次都被成功的更改了，但electron命令依然运行的是最初运行时的那个main文件

所以这个时候我们需要一个监听dist目录变化，自动重启electron命令的方式

```
npm install --save-dev nodemon
```

[nodemon库的一些说明](https://github.com/remy/nodemon)，暂时可以不用去看，下面有做简单的介绍

配置script命令

```json
"start": "cross-env NODE_ENV=dev nodemon --watch dist --exec './node_modules/.bin/electron' ./dist/electron-main.js",
```

- `cross-env NODE_ENV=dev`：不用说了，设置环境
- `--watch dist`：监听dist目录，可设置多个`--watch xxx`
- `--exec './node_modules/.bin/electron' ./dist/electron-main.js`，文件变更后执行的命令，这里也可以理解为`electron ./dist/electron-main.js`

### 配置完整命令

完成以上，我们也就完成了主进程的开发配置

但需要一提的是，在开始开发之前我们需要起3个服务

- `ng serve`
- `tsc -p ./tsconfig.main.json -w`
- `npm start`

所以现在需要做一些整合，需要用到[上一篇文章](https://github.com/luoyang233/blog/blob/master/angular/electron1.md)最后提到的`concurrently`命令行工具，将`package.json`文件做如下更改

```json
"scripts": {
		"start": "npm run watch",
		"watch":"concurrently 'npm run watch:render' 'npm run watch:main' 'npm run watch:dist'",
		"watch:main": "tsc -p ./tsconfig.main.json -w",
		"watch:render": "ng serve",
		"watch:dist":"cross-env NODE_ENV=dev nodemon --watch dist --exec './node_modules/.bin/electron' ./dist/electron-main.js"
	},
```

- `watch`也就是下面三个命令的整合

### 完整流程说明

最后再来理一下整个流程，首先是如果我们更改了angular的文件，那么`ng serve`中会自动监听更改，而主进程又是直接监听的端口，对主进程毫无影响，直接呈现页面即可，这里不用多说

接着是我们更改了主进程文件，然后tsc监听会自动将其重新编译进dist目录，这个时候nodemon又监听到了dist文件内的变化，重新运行electron命令

现在只需要一个`npm start`就可以开始愉快的开发了

