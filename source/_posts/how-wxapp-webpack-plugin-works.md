title: wxapp-webpack-plugin源码学习

date: 2018-12-12 12:02:43

author: xi.lin

categories:

- 源码学习

tags:

- 微信小程序

- webpack

---

# `wxapp-webpack-plugin`简介

以下摘抄自项目的readme文档[^1]：

> - 微信小程序开发需要有多个入口文件（如 `app.js`, `app.json`, `pages/index/index.js` 等等），使用这个插件只需要引入 `app.js` 即可，其余文件将会被自动引入
> - 若多个入口文件（如 `pages/index/index.js` 和 `pages/logs/logs.js`）引入有相同的模块，这个插件能避免重复打包相同模块
> - 支持自动复制 `app.json` 上的 `tabbar` 图片 (v0.17.0 或以上)

即：该插件能简化使用webpack开发微信小程序项目时的配置。

此插件支持如下配置选项:

- `clear`: 在启动 `webpack` 时清空 `dist` 目录。默认为 `true`
- `commonModuleName`: 公共 `js` 文件名。默认为 `common.js`
- `extensions`: 脚本文件后缀名。默认为 `['.js']`

<!-- more -->

# webpack plugin开发简介

> **Plugins** are the [backbone](https://github.com/webpack/tapable) of webpack. webpack itself is built on the **same plugin system** that you use in your webpack configuration![^2]

一个webpack plugin实质上是一个拥有`apply`方法的js对象。webpack compiler在运行过程中调用已注册plugin的apply，并传入compiler实例，从而让plugin能够访问整个编译对象。

```javascript
const pluginName = 'ConsoleLogOnBuildWebpackPlugin';

class ConsoleLogOnBuildWebpackPlugin {
  apply(compiler) {
    compiler.hooks.run.tap(pluginName, compilation => {
      console.log('The webpack build process is starting!!!');
    });
  }
}
```

webpack的相关的工作流程可以阅读参考这篇插件介绍的文章[^3]和另一个很详细的源码解析[^4]。

# 调试环境配置

阅读源码时单步调试对于理解有很好的辅助作用。参考文献[^5]可以快速配置出VS Code下的webpack插件源码调试环境。一个范例启动配置`launch.json`如下：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Launch Local Webpack",
      "program": "${workspaceFolder}/node_modules/webpack/bin/webpack.js",
      "args": [
         "--config", "./webpack.config.babel.js"
      ],
      "cwd": "${workspaceFolder}/test",
    }
  ]
}
```

有点奇怪的是我这vs code下的断点行号与实际代码不一致。可以通过`debugger`语句解决。

另外，调试的时候如果Local下的this是undefined，可以去Closure下找_thisX。

# 源码分析

本次分析基于[commit 60603e6f](https://github.com/Cap32/wxapp-webpack-plugin/commit/60603e6f834750d3063ee9676b04facf7ee32365)进行。该插件有三处export: `createTarget`/`Targets`/`WXAppPlugin`。其中`createTarget`应该是不必要的多余导出，只是`Targets`的辅助生成方法。

## `Targets`

`Targets`用于webpack配置中的target字段[^6]，针对微信、支付宝、百度小程序都给出了一个自定义的function，声明了对应要引用的插件。

对比`webpack`预定义的target[^7]可以发现，小程序的环境类似于web，只去除了`FetchCompileWasmTemplatePlugin`。毕竟目前小程序环境下不支持`WebAssembly`。

`createTarget`包裹了一层wrapper，用于动态生成同名方法，我没看出这一步的意义何在，感觉可以直接`return miniProgramTarget`。

## `WXAppPlugin`

### `constructor`

参数处理。有一个辅助的`deprecated`方法，把`scriptExt`/`forceTarget`转成了`extensions`和`enforceTarget`。

### `apply`

先调用`enforceTarget`规范化target对象。

然后依次绑定了`run`/`watch-run`/`emit`/`after-emit`四个生命周期hook[^8]。在`run`的hook中，还会绑定`compilation hook`。

### `run hook`

`run hook`绑定了`run`方法。

首先，获取base路径(`getBase`)。如果参数指定了base，直接使用。否则，尝试返回entry中app.js所在的目录。否则，使用context路径。

然后，获取所有的`entryResource`。这一步实现了小程序所有资源的自动发现。

- 在`getEntryResource`方法中，找到app.json文件，获取pages/subPackages/tarBar信息。
- 调用`getComponents`方法，递归读取page和相应component的json文件，并忽略外部引用的插件。利用一个叫components的Set避免了重复引入。
- `getTabBarIcons`方法返回了tabBar图标信息。
- 最后赋值给`this.entryResources`的数组内容包括:
  - `app[.js]`
  - 所有的页面pages
  - subPackages里定义的pages
  - 所有的components。

接着，绑定了compiler的`compilation hook`。这里用了一个处理proposal状态的bind syntax[^9]，保证`toModifyTemplate`方法的this是当前类实例。

之后，调用`compileScripts`编译脚本资源。

- 执行`applyCommonsChunk`，拿到所有`entryResources`的完整路径，添加`CommonsChunkPlugin`，保证所有*不属于* `entryResources`的js文件都被打包成`commonModuleName`。
- 把所有app.js以外的资源文件，调用`addScriptEntry`添加到entry中。这个注册方法是绑定了`make hook`，用`SingleEntryPlugin.createDependency`创建依赖，再调用`compilation.addEntry`添加。*这里一个tricky的地方在于添加依赖时name带了路径，配合webpack配置中output规则指定`[name].js`，保证输出的时候会生成在指定目录下。*

最后，调用`compileAssets`引入 assets。

- 注册`compiler`的`compilation hook`，在`compilation`的`before-chunk-assets hook`中，把`assetsChunkName`对应的下一步骤中引入 的所有资源文件剔除出chunks。
- 用`globby`匹配到所有非`.js`文件，即所有entryResource相关的`.json/.wxml/.wxss`文件。作为chunk用`MultiEntryPlugin`注册为assets。*这样，配合`file-loader`的规则，可以实现将资源文件输出到对应的目录下。*
  - `file-loader`的执行时机是在`compilation`的`buildModule`阶段，会在`beforeChunkAssets`之前执行。

### `watch-run hook`

同`run hook`处理。

### `compilation hook`

调用`toModifyTemplate`方法，做了三件事：

- 注册chunkTemplate的`render hook`，在生成的source结尾添加一段注入`commonModuleName`的代码：

- ```javascript
  const injectContent = `; function webpackJsonp() { require("./${posixPath}"); ${globalVar}.webpackJsonp.apply(null, arguments); }`;
  ```

- 注册mainTemplate的`bootstrap hook`，把`window`替换为global变量，在微信小程序中即是`wx`

- 注册mainTemplate的`require-ensure hook`，去除动态加载能力

### `emit hook`

如果参数中指定了`clear`，在第一次运行时会清空output目录。然后调用`toEmitTabBarIcons`，遍历之前通过`app.json`文件获得的`tabBarIcons`资源，添加到`compilation`的`assets`数组中。

### `after-emit hook`

调用`toAddTabBarIconsDependencies`方法，保证`compilation`的`fileDependencies`里存在`tabBarIconPath`。这里的判断条件用了一个小技巧，通过按位取反`~`来判断是否存在[^10]。

# 修改实战

当前[commit 60603e6f](https://github.com/Cap32/wxapp-webpack-plugin/commit/60603e6f834750d3063ee9676b04facf7ee32365)尚未支持app.json中声明的`usingComponents`。我们来动手添加一下支持。

根据前面的源码分析，component的支持实现其实只要保证entry中出现对应的components即可。所以，只需要在`getEntryResource`方法中读取app.json中的`usingComponents`声明。

具体实现可以参考这个[PR](https://github.com/Cap32/wxapp-webpack-plugin/pull/30)。

# 总结

总结一下该插件的实现：

- 读取`app.json`文件，拿到所有的页面与组件信息，添加进entry。
- 将所有其他js文件设为common chunk打包到common.js文件中。
- 将对应的同目录下小程序资源文件添加到chunk里，保证file-loader能复制输出。
- 修改输出的模板文件，添加对common chunk的依赖，并重置window为wx。

# 参考

[^1]: [wxapp-webpack-plugin](https://github.com/Cap32/wxapp-webpack-plugin)
[^2]: [webpack plugins](https://webpack.js.org/concepts/plugins/)
[^3]: [看清楚真正的 Webpack 插件](https://juejin.im/entry/5a4cb7906fb9a04500037399)
[^4]: [Webpack打包流程细节源码解析（P1）](https://github.com/879479119/879479119.github.io/issues/1)
[^5]: [Debugging Webpack with VS Code](https://medium.com/@jsilvax/debugging-webpack-with-vs-code-b14694db4f8e)
[^6]: [Target](https://webpack.js.org/configuration/target/)
[^7]: [Webpack Source Code](https://github.com/webpack/webpack/blob/master/lib/WebpackOptionsApply.js#L98-L109)
[^8]: [Compiler Hooks](https://webpack.js.org/api/compiler-hooks/)
[^9]: [ECMAScript This-Binding Syntax](https://github.com/tc39/proposal-bind-operator)
[^10]: [JavaScript Tilde ~ (Bitwise Not operator)](https://wsvincent.com/javascript-tilde/)

