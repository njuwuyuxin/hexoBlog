---
title: nodejs+express框架搭建简单后端服务
date: 2019-6-07 16:51:40
categories: 后端学习
tags:
- nodejs
- express
- 后端
---
----
## Node安装
由于后端服务通常部署在linux服务器上，因此简单说下linux环境下node的安装。 可以选择去官网下载编译好的二进制文件，软链接到环境目录下。也可以使用apt工具直接安装
```
sudo apt-get install node
```
### Express框架
express是一个功能十分强大的框架，可以同时兼顾前后端开发。但由于这次只是想用express实现后端服务，因此不需要express提供的前端开发模板相关功能。所以只是在项目中引入了express模块

```
npm install express
```
之后就可以在项目中通过require的方式使用express模块

#### Express的使用
首先需要在需要的文件中引入express模块

```
var express = require('epxress');
var app = express();
```
之后需要创建一个http服务器，但是由于我的网站而言，需要提供https服务，因此创建了一个https服务器

```
var httpsServer = https.createServer(options, app);
httpsServer.listen(parseInt(config.port),function(){
    console.log("Https server is running on: https://localhost:"+config.port);
});
```
创建https服务器时需要一个额外参数option，用来指定服务器所需证书的路径，只有证书有效，才能创建https服务。
至于端口号，可以自行指定，由于网站前端运行在默认443端口，因此选择不冲突的端口即可。

创建好服务器之后，我们就可以用app实例去监听对应的请求。
express框架为我们实现了路由功能，因此可以很方便的通过路径来区分各种请求。

```
app.get('/api/activities',newsApi.getActivities);
app.get('/api/activityCards',newsApi.getActivityCards);
app.post('/api/reviewCards',newsApi.getReviewCards);

function getActivities(req, res){
    ...
    ...
    res.send('...')
}
```
通过调用app的get和post方法，我们可以处理get和post请求，第一个参数即为路由的路径，第二个参数为一个函数闭包，用来处理对应的请求。该闭包会接受两个参数req和res，分别对应请求体和返回的内容