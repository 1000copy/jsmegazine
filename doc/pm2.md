# PM2概述

翻译：reco

为什么要使用PM2？在本概述的最后，您将更好地了解使用PM2作为流程管理器的好处。

##永远激活

一旦启动，您的应用程序将永远存在，并在崩溃和计算机重新启动时自动重新启动。
	
	var http = require("http");
	http.createServer(function (request, response) {
	   response.writeHead(200, {'Content-Type': 'text/plain'});
	   response.end('Hello World\n');
	}).listen(3000);
	console.log('Server running at http://127.0.0.1:3000/');

这就像运行一样简单：

	pm2 start app.js

## 进程管理

您的所有应用程序都在后台运行，并且可以轻松管理。

PM2创建一个进程列表，您可以使用以下方法访问：

	pm2 ls


使用`pm2 start`和`pm2 delete`添加和删除进程列表中的进程。

使用`pm2 start`，`pm2 stop`，`pm2 restart`管理你的进程。

## 日志管理

应用程序日志保存在服务器的硬盘中，放在`~/.pm2/logs`中。

使用以下方法访问实时日志：

	pm2 log <app_name>


## 负载均衡

PM2可以通过创建共享同一服务器端口的多个子进程来扩展您的应用程序。这样做还允许您以零秒停机时间重新启动应用程序。

以群集模式启动：

	pm2 start -i max


## 终端监控

您可以在终端中监控您的应用程序并检查应用程序运行状况（CPU使用率，使用的内存，请求/分钟等）。

	pm2 monit


##使用SSH轻松部署

自动部署，避免逐个在所有服务器中进行ssh。

	pm2 deploy



# PM2 Overview

Why use PM2 ? At the end of this overview, you will better understand the benefits of using PM2 as a process manager.

## Forever Alive

Once started, your app is forever alive, auto-restarting across crashes and machine restarts.

	var http = require("http");
	http.createServer(function (request, response) {
	   response.writeHead(200, {'Content-Type': 'text/plain'});
	   response.end('Hello World\n');
	}).listen(3000);
	console.log('Server running at http://127.0.0.1:3000/');

This is as simple as running:

	pm2 start app.js

## Process Management


All your applications are run in the background and can be easily managed.

PM2 creates a list of processes, that you can access with:

	pm2 ls


Add and delete processes to your process list with `pm2 start` and `pm2 delete`.

Manage your processes with `pm2 start`, `pm2 stop`, `pm2 restart`.

## Log Management

Application logs are saved in the hard disk of your servers into `~/.pm2/logs/`.

Access your realtime logs with:

pm2 logs <app_name>


## Zero-config Load-Balancer

PM2 can scale up your application by creating several child processes that share the same server port. Doing this also allow you to restart your app with zero-seconds downtimes.

Start in cluster mode with:

	pm2 start -i max


## In-terminal monitoring

You can monitor your app in the terminal and check app health (CPU usage, memory used, request/min and more).

	pm2 monit


## Easy deploy with SSH

Automate your deployment and avoid to ssh in all your servers one by one.

	pm2 deploy
