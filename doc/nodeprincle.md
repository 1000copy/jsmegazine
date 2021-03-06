# 浅谈Node.js的工作原理及优缺点

原文：https://blog.csdn.net/kaosini/article/details/8090510
编辑：reco

如果您听说过Node，或者阅读过一些文章，宣称Node是多么多么的棒，那么您可能会想：“Node究竟是什么东西？” 即便是在参阅Node的主页之后，您甚至可能还是不明白 Node为何物？Node肯定不适合每个程序员，但它可能是某些程序员一直苦苦追寻的东西。

## Node旨在解决什么问题？

Node公开宣称的目标是 “旨在提供一种简单的构建可伸缩网络程序的方法”。当前的服务器程序有什么问题？我们来做个数学题。在Java™和PHP这类语言中，每个连接都会生成一个新线程，每个新线程可能需要2MB的配套内存。在一个拥有8GBRAM的系统上，理论上最大的并发连接数量是4,000个用户。随着您的客户群的增长，如果希望您的Web应用程序支持更多用户，那么，您必须添加更多服务器。当然，这会增加服务器成本、流量成本和人工成本等成本。除这些成本上升外，还有一个潜在技术问题，即用户可能针对每个请求使用不同的服务器，因此，任何共享资源都必须在所有服务器之间共享。鉴于上述所有原因，整个Web应用程序架构（包括流量、处理器速度和内存速度）中的瓶颈是：服务器能够处理的并发连接的最大数量。

Node解决这个问题的方法是：更改连接到服务器的方式。每个连接发射一个在Node引擎的进程中运行的事件，而不是为每个连接生成一个新的OS线程（并为其分配一些配套内存）。Node声称它绝不会死锁，因为它根本不允许使用锁，它不会直接阻塞 I/O 调用。Node还宣称，运行它的服务器能支持数万个并发连接。

现在您有了一个能处理数万个并发连接的程序，那么您能通过Node实际构建什么呢？如果您有一个Web应用程序需要处理这么多连接，那将是一件很 “恐怖” 的事！那是一种 “如果您有这个问题，那么它根本不是问题” 的问题。在回答上面的问题之前，我们先看看Node的工作原理以及它的设计运行方式。

## Node肯定不是什么？

没错，Node是一个服务器程序。但是，基础Node产品肯定不像Apache或Tomcat。本质上，那些服务器“安装就绪型”服务器产品，支持立即部署应用程序。通过这些产品，您可以在一分钟内启动并运行一个服务器。Node肯定不是这种产品。Apache能通过添加一个PHP模块来允许开发人员创建动态Web页，添加一个SSL模块来实现安全连接，与此类似，Node也有模块概念，允许向Node内核添加模块。实际上，可供选择的用于 Node的模块有数百个之多，社区在创建、发布和更新模块方面非常活跃，一天甚至可以处理数十个模块。本文后面将讨论Node的整个模块部分。

## Node如何工作？

Node本身运行V8 JavaScript。等等，服务器上的JavaScript？没错，您没有看错。对于只在客户机上使用JavaScript的程序员而言，服务器端JavaScript可能是一个新概念，但这个概念本身并非遥不可及，因此为何不能在服务器上使用客户机上使用的编程语言？

什么是V8？V8 JavaScript引擎是Google用于其Chrome浏览器的底层JavaScript引擎。很少有人考虑JavaScript在客户机上实际做了些什么？实际上，JavaScript引擎负责解释并执行代码。Google使用V8创建了一个用C++编写的超快解释器，该解释器拥有另一个独特特征；您可以下载该引擎并将其嵌入任何应用程序。V8 JavaScript引擎并不仅限于在一个浏览器中运行。因此，Node实际上会使用Google编写的V8 JavaScript引擎，并将其重建为可在服务器上使用。太完美了！既然已经有一个不错的解决方案可用，为何还要创建一种新语言呢？

## 事件驱动编程

许多程序员接受的教育使他们认为，面向对象编程是完美的编程设计，这使得他们对其他编程方法不屑一顾。Node使用了一个所谓的事件驱动编程模型。

例一：

    $("#myButton").click(function(){
        if ($("#myTextField").val() != $(this).val())
            alert("Field must match button text");
    });

实际上，服务器端和客户端没有任何区别。没错，这没有按钮点击操作，也没有向文本字段键入的操作，但在一个更高的层面上，事件正在发生。一个连接被建立，这是一个事件；数据通过连接进行接收，这也是一个事件；数据通过连接停止，这还是一个事件！

为什么这种设置类型对Node很理想？JavaScript是一种很棒的事件驱动编程语言，因为它允许使用匿名函数和闭包，更重要的是，任何写过代码的人都熟悉它的语法。事件发生时调用的回调函数可以在捕获事件处进行编写。这样可以使代码容易编写和维护，没有复杂的面向对象框架，没有接口，没有过度设计的可能性。只需监听事件，编写一个回调函数，其他事情都可以交给系统处理！

## 示例Node应用程序

最后，我们来看一些代码！让我们将讨论过的所有内容汇总起来，从而创建我们的第一个Node应用程序。我们已经知道，Node对于处理高流量应用程序很理想，所以我们将创建一个非常简单的Web应用程序，一个为实现最快速度而构建的应用程序。下面是“老板”交代的关于我们的样例应用程序的具体要求：创建一个随机数字生成器RESTful API。这个应用程序应该接受一个输入：一个名为“number”的参数。然后，应用程序返回一个介于0和该参数之间的随机数字，并将生成的数字返回给调用者。由于“老板”希望该应用程序成为一个广泛流行的应用程序，因此它应该能处理50000个并发用户。我们来看看以下代码：


    http.createServer(function(request, response) {
        response.writeHead(200, {"Content-Type": "text/plain"});
        var params = url.parse(request.url, true).query;
        var input = params.number;
        var numInput = new Number(input);
        var numOutput = new Number(Math.random() * numInput).toFixed(0);
        response.write(numOutput);
        response.end();
    }).listen(80);
    console.log("Random Number Generator Running...");


启动应用程序

上面的代码放入一个名为“random.js”的文件中。现在，要启动这个应用程序并运行它（以便创建HTTP服务器并监听端口80上的连接），只需在您的命令提示中输入以下命令：% node random.js。下面是服务器已经启动并运行时看起来的样子：

    # node random.js
    Random Number Generator Running...

访问应用程序

应用程序已经启动并运行。Node正在监听所有连接，我们来测试一下。由于我们创建了一个简单的RESTful API，所以可以使用Web浏览器来访问这个应用程序。键入以下地址（确保您已完成了上面的步骤）：http://localhost/?number=27。

您的浏览器窗口将更改到一个介于0到27之间的随机数字。单击浏览器上的“重新载入”按钮，您会得到另一个随机数字。就是这样，这就是您的第一个Node应用程序！

## Node对什么有好处？

到此为止，您可能能够回答“Node是什么”这个问题了，但您可能还有一个问题：“Node有什么用途？” 这是一个需要提出的重要问题，因为肯定有些东西能受益于Node。

它对什么有好处？

正如您此前所看到的，Node非常适合以下情况：在响应客户端之前，您预计可能有很高的流量，但所需的服务器端逻辑和处理不一定很多。Node表现出众的典型示例包括：

1、RESTful API

提供RESTful API的Web服务接收几个参数，解析它们，组合一个响应，并返回一个响应（通常是较少的文本）给用户。这是适合Node的理想情况，因为您可以构建它来处理数万条连接。它仍然不需要大量逻辑；它本质上只是从某个数据库中查找一些值并将它们组成一个响应。由于响应是少量文本，入站请求也是少量的文本，因此流量不高，一台机器甚至也可以处理最繁忙的公司的API需求。

2、Twitter队列

想像一下像Twitter这样的公司，它必须接收tweets并将其写入数据库。实际上，每秒几乎有数千条tweet达到，数据库不可能及时处理高峰时段所需的写入数量。Node成为这个问题的解决方案的重要一环。如您所见，Node能处理数万条入站tweet。它能快速而又轻松地将它们写入一个内存排队机制（例如memcached），另一个单独进程可以从那里将它们写入数据库。Node在这里的角色是迅速收集tweet，并将这个信息传递给另一个负责写入的进程。想象一下另一种设计（常规PHP服务器会自己尝试处理对数据库本身的写入）：每个tweet都会在写入数据库时导致一个短暂的延迟，因为数据库调用正在阻塞通道。由于数据库延迟，一台这样设计的机器每秒可能只能处理2000条入站tweet。每秒处理100万条tweet则需要500个服务器。相反，Node能处理每个连接而不会阻塞通道，从而能够捕获尽可能多的tweets。一个能处理50000条tweet的Node机器仅需20台服务器即可。

3、电子游戏统计数据

如果您在线玩过《使命召唤》这款游戏，当您查看游戏统计数据时，就会立即意识到一个问题：要生成那种级别的统计数据，必须跟踪海量信息。这样，如果有数百万玩家同时在线玩游戏，而且他们处于游戏中的不同位置，那么很快就会生成海量信息。Node是这种场景的一种很好的解决方案，因为它能采集游戏生成的数据，对数据进行最少的合并，然后对数据进行排队，以便将它们写入数据库。使用整个服务器来跟踪玩家在游戏中发射了多少子弹看起来很愚蠢，如果您使用Apache这样的服务器，可能会有一些有用的限制；但相反，如果您专门使用一个服务器来跟踪一个游戏的所有统计数据，就像使用运行Node的服务器所做的那样，那看起来似乎是一种明智之举。

## Node模块

尽管不是本文最初计划讨论的主题，但应广大读者要求，本文已经扩展为包含一个 Node Modules和Node Package Manager简介。正如已经习惯使用Apache的开发人员那样，您也可以通过安装模块来扩展Node的功能。但是，可用于Node的模块极大地增强了这个产品，那些模块非常有用，将使用Node的开发人员通常会安装几个模块。因此，模块也就变得越来越重要，甚至成为整个产品的一个关键部分。

在“参考资料”部分，我提供了一个指向模块页面的链接，该页面列示了所有可用模块。为了展示模块能够提供的可能性，我在数十个可用模块中包含了以下几个模块：一个用于编写动态创建的页面（比如PHP），一个用于简化MySQL使用，一个用于帮助使用WebSockets，还有一个用来协助文本和参数解析的模块。我不会详细介绍这些模块，这是因为这篇概述文章旨在帮助您了解Node并确定是否需要深入学习（再次重申），如果需要，那么您肯定有机会用到这些可用模块。

另外，Node的一个特性是Node Package Module，这是一个内置功能，用于安装和管理Node模块。它自动处理依赖项，因此您可以确定：您想要安装的任何模块都将正确安装并包含必要的依赖项。它还支持将您自己的模块发布到Node社区，假如您选择加入社区并编写自己的模块的话。您可以将NPM视为一种允许轻松扩展Node功能的方法，不必担心这会破坏您的Node安装。同样，如果您选择深入学习Node，那么NPM将是您的Node解决方案的一个重要组成部分。

## 结束语

阅读本文之后，您在本文开头遇到的问题“Node.js究竟是什么东西?” 应该已经得到了解答，您应该能通过几个清晰简洁的句子回答这个问题。如果这样，那么您已经走到了许多程序员的前面。我和许多人都谈论过Node，但他们对Node究竟用于做什么一直很迷惑。可以理解，他们具有的是Apache的思维方式，认为服务器就是一个应用程序，将HTML文件放入其中，一切就会正常运转。由于大多数程序员都熟悉Apache及其用途，因此，描述Node的最简单方法就是将它与Apache进行比较。Node是一个程序，能够完成Apache能够完成的所有任务（借助一些模块），而且，作为一个可以将其作为基础进行构建的可扩展JavaScript平台，Node还能完成更多的任务。

从本文可以看出，Node完成了它提供高度可伸缩服务器的目标。它使用了Google的一个非常快速的JavaScript引擎，即V8引擎。它使用一个事件驱动设计来保持代码最小且易于阅读。所有这些因素促成了Node的理想目标，即编写一个高度可伸缩的解决方案变得比较容易。

与理解Node是 什么同样重要的是，理解它不是什么。Node并不只是Apache的一个替代品，它旨在使PHP Web应用程序更容易伸缩。事实远非如此。尽管Node还处于初始阶段，但它发展得非常迅速，社区参与度非常高，社区成员创建了大量优秀模块，一年之内，这个不断发展的产品就有可能出现在您的企业中。