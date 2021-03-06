title: 升级到webpack4所踩的坑（一）

date: 2019-03-19 14:18:45

author: chongyang.li, xi.lin

categories:

- web

tags:
- 构建工具
- webpack
---

## background
麦淘前端系统在初次重构时使用的webpack版本为3.10.0, vue版本为2.5.2。由于是多页面结构，项目在develop和product环境下的构建速度非常慢，并且随着项目规模的增大，问题更加明显。这一次希望通过webpack的升级，利用较新的特性来优化构建速度并且简化配置项，增加项目维护性。

## 记录第一个升级过程中卡住很久的issue
升级了相关的包和依赖之后尝试在develop环境下启动，结果出现如下报错：

![error1.jpg](https://img.maitao.com/3a38fef2-0d1e-4926-9a3d-f49d92b6fc61.jpg?imageView2/2/w/400/q/80/format/jpg)![error2.jpg](https://img.maitao.com/1bd568a3-89ee-4649-941d-38b1c3456e40.jpg?imageView2/2/w/400/q/80/format/jpg)

<!-- more -->

通过对比现有环境的代码执行路径发现，本该执行第一个分支的代码进入了第二个分支，而之前由于bfcache的某些原因，我们在脚本前期初始化了MesssageChannel为null,从而引发了这个问题。
回过头看为什么代码为什么没有进入第一分支，显然是setImmedate这个方法出现了问题，debug发现setImmediate方法是存在的但是并不是native code，而是由第三方模块setimmediate.js注入的代码。怀疑global.setImmedate方法被这个模块重写而导致了异常，阅读setImmediate.js代码后发现：
```javascript
   if (global.setImmediate) {
       return;
   }
```
那么应该是其他的模块重写了setImmediate,然而搜遍了整个项目没有找到与setImmediate相关的赋值代码，问题进入了死胡同。期间在网上查阅后，发现webpack有node的相关配置：
```javascript
   node: {
     setImmedate: false,
   }
```
用来取消webpack对于node保留变量不存在是的赋值，然而还是没有work。

打开旧环境，在devtool上执行setImmediate可以看到

```javascript
function setImmediate() {
    [native code]
}
```

的输出。但根据MDN文档，该方法只有IE做了实现，chrome默认应该没有此方法，为了找到旧代码是如何为setImmediate赋值的，我们利用iOS native的webview容器提前运行defineProperty来添加断点（为了保证能在页面所有代码加载之前运行，直接在header最前端添加script段应该也可以），注入的代码如下:

```javascript
var actualSetImmediate = window.setImmediate;
Object.defineProperty(window, 'setImmediate', {
    set: function(newValue) {
        debugger;
        actualSetImmediate = newValue;
    },
    get: function() {
        return actualSetImmediate;
    }
});
```

打开safari，调试simulator中的页面，可以看到是bable-polyfill中的core-js重写了setImmediate方法。而console中查看该方法显示为native code则是core-js为了保证lodash之类的第三方库能识别polyfill的方法而做的障眼法，[重写了toString](https://stackoverflow.com/a/42059050/535374)。

所以webpack需要polyfill来进行支持，而搜索了新环境的runtime后却没有发现babel-polyfill的影子，这时候突然想起在webpack配置文件升级的时候，把所有entry数组开头的['babel-polyfill']都删除了,原因是升级时并不知道这么写作用是什么。

```javascript
function getEntry(globPath) {
  var entries = {},filename;
  glob.sync(globPath).forEach(function (entry) {
      filename = path.basename(entry, path.extname(entry));
      entries[filename] = ['@babel/polyfill', entry];
  });
  entries['maitao'] = ['@babel/polyfill'];
  glob.sync(srcDir + '/common/**/*.js').forEach(function(item){
    entries['maitao'].push(item);
  })
  glob.sync(assetsDir+'/pc/js/*.js').forEach(function(entry){
     filename = path.basename(entry, path.extname(entry));
     entries[filename] = ['@babel/polyfill', entry];
  })
  return entries;
}
```
而由于babel-polyfill也升级到了最新版本，所以写法也有变更。看起来，webpack在处理每一个chunk时都会把polyfill的代码打包进去。
