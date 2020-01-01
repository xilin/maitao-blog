title: WebKit/Chromium源码编译
date: 2018-11-20 15:43:47
author: xi.lin
categories:
- 源码学习
tags:
- webkit
- chromium
- bfcache
---

> 19年已经有了进展，可以参考
> 1. [Back-forward cache](https://www.chromestatus.com/feature/5815270035685376)
> 2. [Exploring a back/forward cache for Chrome](https://developers.google.com/web/updates/2019/02/back-forward-cache)
> 3. [Google implements backward-forward cache in Chrome 79 Canary](https://www.ghacks.net/2019/09/20/google-implements-backward-forward-cache-in-chrome-79-canary/)

在升级VUE版本后发现移动站的bfcache功能失效了，为了更好的调试问题编译了两大移动平台的webview源码进行探究。
<!-- more -->

# WebKit

* [官方文档](https://webkit.org/getting-started/)比较完善，mac用户按教程往下走即可

## 代码获取
```
$ wget -c https://s3-us-west-2.amazonaws.com/archives.webkit.org/WebKit-SVN-source.tar.bz2  //snapshot of the WebKit source code
```
- 这个命令可以直接下载一个source code的tarball存档，在阿里云服务器上速度下载速度特别快，1.8G的文件几分钟完成

## 编译
```
$ ./Tools/Scripts/build-webkit --debug
```
默认的编译的生成目录是WebKitBuild。在配置为*i5-4670/32G RAM/机械硬盘*的电脑上用时如下。

![WebKit build succeeded.png](https://upload-images.jianshu.io/upload_images/722899-cce4876063f08346.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 调试
- 添加MiniBrowser
- 修改build location


## 参考
[1. webkit官方文档](https://webkit.org/getting-started/)
[2. 【技巧】编译运行WebKit Demo（Mac调试版本）](https://www.jianshu.com/p/84ef19c5af2d)
[3. 调试 WebKit](https://zhuanlan.zhihu.com/p/26144355)
[4. 解决UIWebView 前进、后退刷新的坑](https://blog.csdn.net/wadahana/article/details/50168643)

# Chromium

*核心问题：需要备一个好梯子*
## 代理准备
- 如果你用的是ss或者类似的代理工具
  * 最好提供http代理，可以使用ss-ng或者privoxy
  * 配好代理环境变量
```
$ export ALL_PROXY=http://127.0.0.1:1082
$ git config --global https.proxy http://127.0.0.1:1082
$ git config --global http.proxy http://127.0.0.1:1082
```
  * 在安装的最后可能会有后续的warning出现，不一定会真的失败，如果失败的话可以参考[这个](https://github.com/ChenYilong/WebRTC/blob/master/WebRTC%E5%9C%A8iOS%E7%AB%AF%E7%9A%84%E5%AE%9E%E7%8E%B0/WebRTC%E5%9C%A8iOS%E7%AB%AF%E7%9A%84%E5%AE%9E%E7%8E%B0.md)
>  NOTICE: You have PROXY values set in your environment, but gsutil in depot_tools does not (yet) obey them. Also, --no_auth prevents the normal BOTO_CONFIG environment variable from being used. To use a proxy in this situation, please supply those settings in a .boto file pointed to by the NO_AUTH_BOTO_CONFIG environment var.


## 安装depot_tools
```
$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
$ export PATH="$PATH:/path/to/depot_tools"
```
- depot_tools是chromium的源码管理工具，由一系列脚本组成，其中在本教程中最重要的两个脚本是:
1. `fetch`
> usage: fetch.py [options] <config> [--property=value [--property2=value2 ...]]
  - 其中参数`--nohooks`在跳过hook执行，`--no-history`保证是shallow clones
  - fetch可用的config都放在`fetch_configs`目录下，包括我们要用到的`chromium.py`，如果有好用的国内镜像，可以在这里替换spec url
2. `gclient`
> Usage: gclient.py <command> [options]
  - 一个可以管理多git/svn项目的脚本
  - 会参考`chromium/src/DEPS`这个文件定义的配置，包括sync和hook的具体内容
  - 两个命令
  a. `runhooks`，如果之前fetch用了`--nohooks`，可以再手动执行此命令
  b. `sync`，同步所有module的代码，适用于在第一次fetch因各种原因中断的时候继续跑

## 获取代码
```
$ mkdir chromium && cd chromium
$ git config --global core.precomposeUnicode true // Ensure that unicode filenames aren't mangled by HFS:
$ fetch  --nohooks  --no-history chromium // 先不执行hooks，减少网络传输
```
这一步视网络状态而定，如果中间中断的话可以通过以下语句继续
```
$ gclient sync
```
在漫长的等待后，代码获取完成，再执行hooks
```
$ cd src
$ gclient runhooks
```
继续一次漫长的等待

## 准备编译
```
$ gn gen out/gn --ide=xcode
```
- 使用[gn](https://chromium.googlesource.com/chromium/src/tools/gn/)进行编译配置
  * gn是一个用于生成ninja文件的meta-build系统
  * 执行`gn help gen`可以看到支持的参数，此处我们指定使用xcode，可以参考[这一段文档](https://chromium.googlesource.com/chromium/src/+/master/docs/mac_build_instructions.md#using-xcode_ninja-hybrid)
 
## 编译
```
$ ninja -C out/gn chrome
```
*漫长的等待第三遍*，在配置为*i5-4670/32G RAM/机械硬盘*的电脑上用了大概6小时。

## bfcache相关
[Investigate faster back/forward page navigation](https://bugs.chromium.org/p/chromium/issues/detail?id=511340) 未实现
[Back/Forward Cache](https://docs.google.com/document/d/13ZlBgcA4X-AEbe6Xt4OdY9rVhYwFfeXVC1KndFufrfA) qq浏览器的设计讨论

## 参考
* [1. 官方mac build指南](https://chromium.googlesource.com/chromium/src/+/master/docs/mac_build_instructions.md)
* [2. 官方depottools介绍](https://dev.chromium.org/developers/how-tos/depottools)
* [3. 管理Chromium源代码的利器——depot_tools](http://gclxry.com/use-depot_tools-to-manage-chromium-source/)
* [4. 从Chrome源码看浏览器如何构建DOM树](https://zhuanlan.zhihu.com/p/24911872)
* [5. mac上面进行chromium编译](https://www.jianshu.com/p/4ba1d2275767)
