## 背景
我们使用了[hexo](https://hexo.io/zh-cn/docs/)和它的[NexT](https://theme-next.iissnan.com/)主题。

## 安装
npm install -g hexo-cli

## 写作
hexo new post <title>

> 可以参考source/_posts下已有的文章配置一些元信息

## 备注
为了使用gitment，需要修改`themes/next/layout/_third-party/comments/gitment.swig`文件里的js/css link为
```
<script src="https://cdn.jsdelivr.net/gh/theme-next/theme-next-gitment@1/gitment.browser.js"></script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/theme-next/theme-next-gitment@1/default.css"/>
```
https://github.com/imsun/gitment/issues/170
