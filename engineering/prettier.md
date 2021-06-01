## 搞懂 ESLint 和 Prettier

多人合作有什么问题呢？**代码风格不一致**。这个问题会导致什么结果呢？

1. **git diff 的时候会导致很多仅仅是格式的修改出现**。比如你喜欢用两个空格，张三喜欢用一个 tab，而 Pony 老师则喜欢用三个空格。这时候你需要对 Pony 的代码进行修改，看到他三个空格很不爽，修改了逻辑的同时，顺便帮他调整了一下格式，自认为帮了人家一个大忙，然后提了 MR。Pony 老师作为 approver 看到自己的代码被改的面目全非而感到气气，于是很愤怒的拒绝掉了你的 MR ...
2. **每个团队有自己的标准导致新人的加入需要花很大时间熟悉代码风格**。经过一个多月的 debate，Pony 老师终于累了，于是放弃争辩决定以后跟你一样使用两个空格。你获得了胜利，认为既然团队统一了风格，那不会再有风格的问题出现了。直到团队来了个新人 - Tony 老师，Tony 老师多年大厂经验让他形成了自己的一套观点，那就是 -- 四个空格才是最完美的缩进，于是你的战斗又开始了...

兄弟，你不是一个人。曾经有个自由人 A 的眼睛被另外一个自由人 B 弄瞎了，B 想赔偿 A 一个金币，A 却认为应该赔偿两个金币，于是他们发生了争吵，于是当地治安官介入。治安官怎么解决这个问题呢？"诶，这个 so easy 啊。这汉谟拉比法典上不是写的清清楚楚么，'以眼还眼，以牙还牙'”。于是 B 的眼睛也瞎了，虽然最后谁也没有得到自己想要的，但总算，问题解决了。

## ESLint

这个故事告诉我们一个道理，要解决问题，就要**先把规矩都列出来写在文件里**，于是大名鼎鼎的 ESLint 出来了。

> ESLint 是什么呢？
> 是一个开源的 JavaScript 的 linting 工具，使用 [espree](https://link.zhihu.com/?target=https%3A//github.com/eslint/espree) 将 JavaScript 代码解析成抽象语法树 (AST)，然后通过AST 来分析我们代码，从而给予我们两种提示：

1. **代码质量问题：使用方式有可能有问题****(problematic patterns)**
2. **代码风格问题：风格不符合一定规则 (doesn’t adhere to certain style guidelines)**

(这里的两种问题的分类很重要，下面会主要讲）

终于有一天，你发现了 ESLint 这个工具，于是你拉上 Pony 老师，Tony 老师你们一起开了个会，在你苦口婆心的劝说下，终于大家一致同意使用两个空格作为缩进，于是

1. 你安装了 ESLint，并将下面的配置写入 `.eslintrc`文件。

```js
// .eslintrc    
{        
    "indent": ["error", 2]    
}
```

1. 你还统一让大家下载了 **ESLint 的 VSCode 插件**，没有通过 ESLint 校验的代码 VSCode 会给予下滑波浪线提示。
2. 为了万无一失，你还添加一个 pre-commit 钩子 `eslint --ext .js src`，确保**没有通过 lint 的代码不会被提交**。
3. 更让人开心的是，之前不统一的代码也能通过 `eslint --fix` 来修改成新的格式。

## Airbnb Style Guide

你很开心，这样一来，不管有多少新人来都不会再有缩进格式的问题了。但是，你后面又发现，新来的小红竟然在使用大括号的时候新起一行。于是你陷入了沉思，觉得不可能针对每个格式都要大家开会讨论。于是你打开了 Google，发现很多公司都有跟你一样的问题。很多大公司都提出了自己公司的标准，其中有一个叫 Airbnb 的公司做的是最好的，于是你又叫了所有人来开会，终于大家一致同意使用一套完整的规范。会后，你 install 了 [eslint-config-airbnb](https://link.zhihu.com/?target=https%3A//www.npmjs.com/package/eslint-config-airbnb) ，并且将 `.eslintrc` 文件改成了下面这样，终于大功告成。

```js
// .eslintrc
{
  "extends": ["airbnb"]
}
```

## Prettier

于是问题都解决了么？那是不可能的，解决了的话这篇文章也不会这么长 - 。-

上面我们说到，ESLint 主要解决了两类问题，但其实 ESLint 主要解决的是**代码质量问题**。另外一类**代码风格问题**其实 [Airbnb JavaScript Style Guide](https://link.zhihu.com/?target=https%3A//github.com/airbnb/javascript) 并没有完完全全做完，因为这些问题"没那么重要"，代码质量出问题意味着程序有潜在 Bug，而风格问题充其量也只是看着不爽。

- 代码质量规则 (code-quality rules)

- - [no-unused-vars](https://link.zhihu.com/?target=http%3A//eslint.org/docs/rules/no-unused-vars)
  - [no-extra-bind](https://link.zhihu.com/?target=http%3A//eslint.org/docs/rules/no-extra-bind)
  - [no-implicit-globals](https://link.zhihu.com/?target=http%3A//eslint.org/docs/rules/no-implicit-globals)
  - [prefer-promise-reject-errors](https://link.zhihu.com/?target=http%3A//eslint.org/docs/rules/prefer-promise-reject-errors)
  - ...

- 代码风格规则 (code-formatting rules)

- - [max-len](https://link.zhihu.com/?target=http%3A//eslint.org/docs/rules/max-len)
  - [no-mixed-spaces-and-tabs](https://link.zhihu.com/?target=http%3A//eslint.org/docs/rules/no-mixed-spaces-and-tabs)
  - [keyword-spacing](https://link.zhihu.com/?target=http%3A//eslint.org/docs/rules/keyword-spacing)
  - [comma-style](https://link.zhihu.com/?target=http%3A//eslint.org/docs/rules/comma-style)
  - ...

这时候就出现了 Prettier，Prettier 声称自己是一个有主见 (偏见) 的代码格式化工具 (opinionated code formatter)，Prettier 认为**格式很重要**，但是格式好麻烦，我来帮你们定好吧。简单来说，不需要我们再思考究竟是用 single quote，还是 double quote 这些乱起八糟的格式问题，Prettier 帮你处理。最后的结果，可能不是你完全满意，但是，绝对不会丑，况且，Prettier 还给予了一部分配置项，可以通过 `.prettierrc` 文件修改。

所以相当于 **Prettier 接管了两个问题其中的代码格式的问题，而使用 Prettier + ESLint 就完完全全解决了两个问题**。但实际上使用起来配置有些小麻烦，但也不是什么大问题。因为 Prettier 和 ESLint 一起使用的时候会有冲突，所以

1. 首先我们需要使用 eslint-config-prettier 来关掉 (disable) 所有和 Prettier 冲突的 ESLint 的配置（这部分配置就是上面说的，格式问题的配置，所以关掉不会有问题），方法就是在 .eslintrc 里面将 prettier 设为最后一个 extends

```js
// .eslintrc    
{      
    "extends": ["prettier"] // prettier 一定要是最后一个，才能确保覆盖    
}
```



\2. (可选，推荐) 然后再启用 `eslint-plugin-prettier` ，将 prettier 的 rules 以插件的形式加入到 ESLint 里面。这里插一句，为什么"可选" ？当你使用 Prettier + ESLint 的时候，其实格式问题两个都有参与，disable ESLint 之后，其实格式的问题已经全部由 prettier 接手了。那我们为什么还要这个 plugin？其实是因为我们期望报错的来源依旧是 ESLint ，使用这个，相当于**把 Prettier 推荐的格式问题的配置以 ESLint rules 的方式写入**，这样相当于可以统一代码问题的来源。

```js
// .eslintrc    
{      
    "plugins": ["prettier"],      
    "rules": {        
        "prettier/prettier": "error"      
    }    
}
```



将上面两个步骤和在一起就是下面的配置，也是官方的推荐配置

```js
// .eslintrc
{
  "extends": ["plugin:prettier/recommended"]
}
```

（全剧终）