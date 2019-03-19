title: 制作使用webpack构建的Node.js模块项目
date: 2018-11-30 18:39:38
author: xi.lin
categories:
- web
tags:
- webpack
- node.js
- module

---

## 创建一个Node.js模块

参考官方文档[^1]，使用`npm init --scope=maitao`快速创建一 个[scoped modules](https://docs.npmjs.com/about-scopes)。

注意，此时只能使用Node.js的模块化系统，使用`exports`导出方法。(ES6和Node.js的模块化区别可以参考引用文献2[^2])。

## 配置webpack

引入webpack可以让我们方便的使用新的语言特性，优化打包流程。阅读引用文献3[^3]可知，想要打包成工具库，需要配置**webpack.config.js** 文件里的 `library`/`libraryTarget`。

同时，可以添加`webpack-bundle-analyzer`用于分析生成的包。

<!-- more -->

```javascript
if (process.env.npm_config_report) { // 
  const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
  plugins.push(new BundleAnalyzerPlugin());
}
```

运行如下命令，查看生成的bundle可以看到引用的第三方库也被引入了。

> npm run build --report

因此需要使用`externals`配置选项排除。

具体的配置参考了引用文献4[^4]和5[^5]。

## 添加示例代码

为了便于开发，参考文献5[^5]和6[^6]配置了`devServer`，建立example目录引用bundle进行调试。

webpack生成的文件可以在 http://localhost:8080/webpack-dev-server 查看。

通过调试可以确认export的对象名与`output`里的`library`名称一致。

## 发布

作为模块文件，publish后被使用的最好是minify过的生产文件。所以修改`package.json`中的`main`字段为`dist/target.js`。

如果要发布到私有库上，需要先执行`npm login`，再`npm publish`。

## P.S.

1. `externals`没有完全生效，`lodash`被使用的部分还是出现在bundle中。不知道能否完全排除。
2. 示例项目可于 https://github.com/xilin/node-webpack-module-example 获取。

## 参考

[^1]: [Creating Node.js modules](https://docs.npmjs.com/creating-node-js-modules)
[^2]: [exports、module.exports 和 export、export default 到底是咋回事](https://juejin.im/post/597ec55a51882556a234fcef)
[^3]: [Authoring Libraries](https://webpack.js.org/guides/author-libraries/)
[^4]: [基于Webpack和ES6构建NPM包](https://juejin.im/post/5ac4a4d85188255c4c107e42)
[^5]: [使用 webpack2 和 NPM Scripts 进行 JavaScript 组件开发](https://www.h5jun.com/post/using-webpack2-and-npm-scripts.html)
[^6]: [Using webpack-dev-server](