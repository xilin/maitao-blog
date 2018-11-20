title: Whistle配置教程
date: 2018-11-20 15:23:13
author: xi.lin
categories:
- 工具
tags:
- whistle
- debug
- 代理
- 抓包
- 工具
---

## 简介
![Whistle Logo](https://raw.githubusercontent.com/avwo/whistle/master/biz/webui/htdocs/img/whistle.png)
[Whistle](https://github.com/avwo/whistle)是基于Node实现的跨平台抓包调试代理工具，亮点在于使用类似于host的规则语法配置各类拦截功能。
<!-- more -->
## 安装
  - 参考[持续集成实施(十六)——Whistle抓包](https://diaojunxian.github.io/2017/01/18/%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E5%AE%9E%E6%96%BD-%E5%8D%81%E5%85%AD-%E2%80%94%E2%80%94Whistle%E6%8A%93%E5%8C%85/)

## 启动
> w2 start -p 9900

## 访问
  - 进入whistle界面，Chrome访问
  > localhost:9900

## 代理设置
  - https证书配置
    * https://avwo.github.io/whistle/webui/https.html
  - Chrome推荐在使用whistle时切换代理配置
    * 用Proxy SwitchyOmega比较方便
  - iOS和Android手机在wifi设置中配置代理
  - iOS需要在配置证书完成后，再开启whistle页面上https下的Intercept HTTPS CONNECTs

## 添加规则
  - 添加代理规则，本机nginx需要启动，可能需要添加server配置`(server_name  localhost m-test.maitao.com;)`
  > m-test.maitao.com http://localhost:8900

## 查看抓包
  - 在微信中试着访问m-test.maitao.com，观察Whistle Network面板，https是否转发生效