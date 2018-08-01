# Understanding the NodeJS cluster module

by Antonio Santiago http://www.acuriousanimal.com/
翻译：reco

NodeJS进程在单个进程上运行，这意味着默认情况下它不会从多核系统中获益。如果你有一个8核CPU并通过$ node app.js运行NodeJS程序，它将在一个进程中运行，其余的CPU是浪费的。

幸运的是，NodeJS提供的集群模块，包含一组功能和属性可帮助我们创建充分使用所有CPU的程序。毫无疑问，集群模块用于最大化CPU使用率的机制是通过创建多个进程，类似于旧的fork（）系统调用Unix系统。

##介绍集群模块

集群模块是一个NodeJS模块，它包含一组功能和属性，可帮助我们分配流程以利用多核系统。特别是如果您在HTTP服务器应用程序中工作

通过集群模块，主进程可以fork任意数量的工作进程，并与它们通过IPC通信发送消息进行通信。请记住，进程之间没有共享内存。

接下来的行是NodeJS文档中的句子汇编。我复制它们过来，以便可以帮助你理解整个事情的运作方式。

1. Node.js的单个实例运行起来时是在单线程的。为了利用多核系统，用户有时会想要启动一个Node.js进程集群来处理负载。
2. 群集模块允许轻松创建所有共享服务器端口的子进程。
3. 使用child_proces.fork()方法生成worker（子）进程，以便它们可以通过IPC与父进行通信并来回传递服务器句柄。 child_process.fork()专门用于生成新的Node.js进程。返回的ChildProcess将内置一个额外的通信通道，允许通过send()方法在父节点和子节点之间来回传递消息。有关详细信息，请参阅subprocess.send()。
4. 请务必记住，生成的Node.js子进程独立于父进程，但两者之间建立的IPC通信通道除外。每个进程都有自己的内存，并有自己的V8实例。由于需要额外的资源分配，因此不建议生成大量子Node.js进程。

因此，大多数魔法都是由child_process模块​​完成的，该模块可以生成新进程并帮助它们之间进行通信，例如，创建管道。你可以在Node.js Child Processes找到一篇很棒的文章：你需要知道的一切。

## A basic example

啰嗦完毕，让我们看到一个真正的例子。接下来我们展示一个非常基本的代码，它可以创建一个主进程，用于检索CPU的数量并为每个CPU分配一个工作进程，并且每个子进程在控制台中打印一条消息并退出。


    const cluster = require('cluster');
    const http = require('http');
    const numCPUs = require('os').cpus().length;
    if (cluster.isMaster) {
      masterProcess();
    } else {
      childProcess();  
    }
    function masterProcess() {
      console.log(`Master ${process.pid} is running`);
      for (let i = 0; i < numCPUs; i++) {
        console.log(`Forking process number ${i}...`);
        cluster.fork();
      }
      process.exit();
    }
    function childProcess() {
      console.log(`Worker ${process.pid} started and finished`);
      process.exit();
    }

将代码保存在app.js文件中并运行执行：$ node app.js. 输出应类似于：

    $ node app.js

输出的样子：

    Master 8463 is running
    Forking process number 0...
    Forking process number 1...
    Forking process number 2...
    Forking process number 3...
    Worker 8464 started and finished
    Worker 8465 started and finished
    Worker 8467 started and finished
    Worker 8466 started and finished

### Code explanation

当我们运行app.js程序时，会创建一个开始运行我们代码的操作系统进程。在开始时，导入`const cluster = require（'cluster'）`并在if语句中检查isMaster属性。

因为进程是第一个进程，所以isMaster属性为true，然后我们运行masterProcess函数的代码。此函数没有太大的秘密，它根据您的机器的CPU数量循环，并使用cluster.fork()方法分叉当前进程。

fork()真正做的是创建一个新的Nodejs进程，就像你通过命令行使用$ node app.js运行它一样，就是你有很多进程运行你的app.js程序。

创建并执行子进程时，它与主进程相同，即导入集群模块并执行if语句。一旦与子进程存在差异，cluster.isMaster的值为false，因此它们结束运行childProcess函数。

注意，我们使用process.exit()显式终止master和worker进程，默认情况下返回零值。


## Comunicating master and worker processes

创建工作进程时，在工作者和主服务器之间创建IPC通道，允许我们使用send（）方法在它们之间进行通信，该方法接受JavaScript对象作为参数。 请记住，它们是不同的进程，因此我们不能使用共享内存作为通信方式。

在主进程中，我们可以使用进程引用（即someChild.send（{...}））向工作进程发送消息，并且在工作进程中，我们可以使用当前进程引用向主进程发送消息，即process.send（）。

我们已经更新了以前的代码，以允许主进程向工作进程发送和接收消息，工作人员也可以从主进程接收和发送消息：

    function childProcess() {
      console.log(`Worker ${process.pid} started`);
      process.on('message', function(message) {
        console.log(`Worker ${process.pid} recevies message '${JSON.stringify(message)}'`);
      });
      console.log(`Worker ${process.pid} sends message to master...`);
      process.send({ msg: `Message from worker ${process.pid}` });
      console.log(`Worker ${process.pid} finished`);
    }

工作进程只是为了理解。 首先，我们使用process.on（'message'，handler）方法监听注册侦听器的消息事件。 稍后我们使用process.send（{...}）发送消息。 请注意，该消息是一个纯JavaScript对象。

    let workers = [];
    function masterProcess() {
      console.log(`Master ${process.pid} is running`);
      // Fork workers
      for (let i = 0; i < numCPUs; i++) {
        console.log(`Forking process number ${i}...`);
        const worker = cluster.fork();
        workers.push(worker);
        // Listen for messages from worker
        worker.on('message', function(message) {
          console.log(`Master ${process.pid} recevies message '${JSON.stringify(message)}' from worker ${worker.process.pid}`);
        });
      }
      // Send message to the workers
      workers.forEach(function(worker) {
        console.log(`Master ${process.pid} sends message to worker ${worker.process.pid}...`);
        worker.send({ msg: `Message from master ${process.pid}` });    
      }, this);
    }

masterProcess函数分为两部分。 在第一个循环中，我们分配的工作量与我们拥有的CPU一样多。 cluster.fork（）返回表示工作进程的工作对象，我们将引用存储在数组中并注册一个侦听器以接收来自该工作器进程实例的消息。

稍后，我们遍历工作进程数组并从主进程向该具体工作程序发送消息。

如果您运行代码：

    $ node app.js

输出将类似于：

    Master 4045 is running
    Forking process number 0...
    Forking process number 1...
    Master 4045 sends message to worker 4046...
    Master 4045 sends message to worker 4047...
    Worker 4047 started
    Worker 4047 sends message to master...
    Worker 4047 finished
    Master 4045 recevies message '{"msg":"Message from worker 4047"}' from worker 4047
    Worker 4047 recevies message '{"msg":"Message from master 4045"}'
    Worker 4046 started
    Worker 4046 sends message to master...
    Worker 4046 finished
    Master 4045 recevies message '{"msg":"Message from worker 4046"}' from worker 4046
    Worker 4046 recevies message '{"msg":"Message from master 4045"}'

这里我们不是用process.exit（）来终止进程，所以要关闭你需要使用ctrl + c的应用程序。

## Conclusion

集群模块为NodeJS提供了使用CPU全部能力而所需的功能。虽然在这篇文章中没有看到，但是群集模块补充了子进程模块，该模块提供了大量工具来处理进程：启动，停止和管道输入/输出等。集群模块允许我们轻松创建工作进程。 此外，它神奇地创建了一个IPC通道，用于传递传递JavaScript对象的主进程和工作进程。

# Clusted HTTP

集群模块允许我们提高多核CPU系统中应用程序的性能。无论是使用API还是基于ExpressJS的Web服务器，这都非常重要，我们希望利用我们的NodeJS应用程序运行的每台机器上的所有CPU。集群模块允许我们在一组工作进程之间对传入请求进行负载均衡，并因此提高了应用程序的吞吐量。

在这篇文章中，我们将看到如何在创建HTTP服务器时使用集群模块，使用普通的HTTP模块和ExpressJS.Lets，看看我们如何创建一个真正基本的HTTP服务器来获取集群模块的利润。

    const cluster = require('cluster');
    const http = require('http');
    const numCPUs = require('os').cpus().length;
    if (cluster.isMaster) {
      masterProcess();
    } else {
      childProcess();  
    }
    function masterProcess() {
      console.log(`Master ${process.pid} is running`);
      for (let i = 0; i < numCPUs; i++) {
        console.log(`Forking process number ${i}...`);
        cluster.fork();
      }
    }
    function childProcess() {
      console.log(`Worker ${process.pid} started...`);
      http.createServer((req, res) => {
        res.writeHead(200);
        res.end(`Hello World from ${process.pid}`);
      }).listen(3000);
    }

我们将代码分为两部分，一部分对应于主进程，另一部分是初始化工作进程的部分。 这样，masterProcess函数就每个CPU代码分配一个工作进程。 另一方面，childProcess只是在端口3000上创建一个HTTP服务器侦听器，并返回一个带有200状态代码的漂亮的Hello World文本字符串。

如果运行代码，输出必须显示如下内容：

    $ node app.js

输出：

    Master 1859 is running
    Forking process number 0...
    Forking process number 1...
    Forking process number 2...
    Forking process number 3...
    Worker 1860 started...
    Worker 1862 started...
    Worker 1863 started...
    Worker 1861 started...

从客户端访问的效果：

    $ curl http://localhost:3000/
    Hello World from 7036
    $ curl http://localhost:3000/
    Hello World from 7037
    $ curl http://localhost:3000/
    Hello World from 7038
    $ curl http://localhost:3000/
    Hello World from 7039

基本上，我们的主进程正在为每个CPU运行一个新的工作进程，该进程运行处理请求的HTTP服务器。正如您所看到的，这可以提高您的服务器性能，因为有一个处理一百万个请求而不是四个进程参与这些请求是不一样的。

## How cluster module works with network connections ?

前面的例子很简单，但隐藏了一些棘手的东西，一些神奇的特性简化了我们作为开发人员的生活。

在任何操作系统中，进程都可以使用端口与其他系统进行通信，这意味着，给定端口只能由该进程使用。那么，问题是，工作进程如何使用相同的端口？

答案是主进程是在给定端口中侦听并在所有子进程/工作进程之间对请求进行负载均衡的进程。从官方文档：

    使用child_process.fork（）方法生成工作进程，以便它们可以通过IPC与父进程通信并来回传递服务器句柄。

群集模块支持两种分发传入连接的方法。

第一个（除了Windows之外的所有平台上都是默认的）是轮询方法，其中主进程侦听端口，接受新连接并以轮询方式在工作者之间分配它们，其中一些是内置的智慧以避免工作进程过载。

第二种方法是主进程创建侦听套接字并将其发送给感兴趣的工作者。然后工作进程直接接受传入的连接。

只要有一些工作进程还活跃着，服务器就会继续接受连接。如果没有工作进程，则将删除现有连接，并拒绝新连接。

##群集模块负载平衡的其他替代方法

集群模块允许主进程接收请求并在所有工作进程之间对其进行负载平衡。这是一种提高性能的方法，但它不是唯一的方法。

在Node.js后的进程中，负载均衡性能：比较集群模块，iptables和Nginx，您可以找到以下性能比较：节点集群模块，iptables和nginx反向代理。

##结论

如今，任何Web应用程序都必须具有性能，我们需要支持高吞吐量并快速提供数据。

集群模块是一种可能的解决方案，它允许我们拥有一个主进程并为每个核心创建一个工作进程，以便它们运行HTTP服务器。群集模块提供两个强大功能：

通过创建IPC通道并允许使用process.send（）发送消息，简化主服务器和工作服务器之间的通信，
允许工作进程共享同一个端口。这样做使得主进程成为接收请求并在工作者之间多路复用的进程。


# Using PM2 to manage NodeJS cluster

集群模块允许我们创建工作进程以提高NodeJS应用程序的性能。这在Web应用程序中尤其重要，其中主进程接收所有请求并在工作进程之间对它们进行负载平衡。

但所有这些功能都伴随着管理与流程管理相关的所有复杂性的应用程序所需的成本：如果工作进程意外存在，工作进程如何正常退出，如果需要重新启动所有工作人员等等，会发生什么？ 。

在这篇文章中，我们介绍了PM2工具。虽然它是一个通用的流程管理器，但这意味着它可以管理任何类型的流程，如python，ruby 等，而不仅仅是NodeJS流程，该工具专门用于管理想要使用群集模块的NodeJS应用程序。

##介绍PM2

如前所述，PM2是一个通用流程管理器，即控制其他流程执行的程序（如检查您是否有新电子邮件的python程序），并执行以下操作：检查您的流程是否正在运行，重新执行你的过程如果由于某种原因意外退出，记录其输出等。

对我们来说最重要的是PM2简化了NodeJS应用程序的执行，以便作为集群运行。是的，您编写应用程序而不必担心群集模块，并且PM2创建了一定数量的工作进程来运行您的应用程序。

## PM2方式

在继续之前，您需要在系统上安装PM2。通常它安装为全局模块，使用
  
  $ npm install pm2 -g

当使用PM2时，我们可以忘记与主进程相关的代码部分，这将是PM2的责任，所以我们非常基本的HTTP服务器可以重写为：

    const http = require('http');
    console.log(`Worker ${process.pid} started...`);
    http.createServer((req, res) => {
      res.writeHead(200);
      res.end('Hello World');
      process.exit(1);
    }).listen(3000);

选择这样运行起来：

    $ pm2 start app.js -i 3 

请注意选项-i，用于指示要创建的实例数。 想法是该数字与您的CPU核心数相同。 如果您不了解它们，可以设置-i 0让PM2自动检测到它。


##结论

虽然NodeJS集群模块是一种提高性能的强大机制，但它需要管理应用程序可以找到的所有情况所需的复杂性：如果工作者存在会发生什么，我们如何在没有停机的情况下重新加载应用程序集群等等。PM2是一个专门设计用于NodeJS集群的流程管理器。 它允许集群应用程序，重新启动或重新加载，除了提供工具以查看日志输出，监视等外，还没有所需的代码复杂性。

# Graceful shutdown NodeJS HTTP server when using PM2

所以你创建了一个收到大量请求的NodeJS服务器，你真的很开心，但是，就像每一个软件一样，你发现了一个bug或者为它添加了一个新功能。 很明显，您需要关闭NodeJS进程并重新启动，以便进行新代码。 问题是：如何以优雅的方式实现这一点，以便继续提供传入的请求？

##启动HTTP服务器

在了解我们必须如何关闭HTTP服务器之后，让我们看看通常如何创建一个。 下一个代码显示了一个带有ExpressJS服务的非常基本的代码，它将返回Hello World！ 访问/ hello路径时。 你也可以传递一个路径参数，即/ hello / John的名字，这样它就会返回Hello John !!!


    const express = require('express')
    const expressApp = express()
    // Responds with Hello World or optionally the name you pass as path param
    expressApp.get('/hello/:name?', function (req, res) {
      const name = req.params.name
      if (name) {
        return res.send(`Hello ${name}!!!`)
      }
      return res.send('Hello World !!!')
    })
    // Start server
    expressApp.listen(3000, function () {
      console.log('App listening on port 3000!')
    })

app.listen（）函数的作用是使用核心http模块启动一个新的HTTP服务器，并返回对HTTP服务器对象的引用。 具体来说，listen（）的源代码如下：

    app.listen = function listen() {
      var server = http.createServer(this);
      return server.listen.apply(server, arguments);
    };

注意：创建快速服务器的另一种方法是将我们的expressApp引用直接传递给http。 createServer（），类似于：const server = http.createServer（app）.listen（3000）。

## How to shutdown properly an HTTP server ?

关闭HTTP服务器的正确方法是调用server.close（）函数，这将阻止服务器接受新连接，同时保留现有连接直到响应它们。

下一代码提供了一个新的/关闭端点，一旦被调用将停止HTTP服务器并退出应用程序（停止nodejs进程）：

    app.get('/close', (req, res) => {
      console.log('Closing the server...')
      server.close(() => {
        console.log('--> Server call callback run !!')
        process.exit()
      })
    })

很明显，通过端点关闭服务器不是正确的方法。

## Graceful shutdown/restart with and without PM2

优雅关闭的目标是关闭到服务器的传入连接，但不会杀死我们正在处理的当前连接。

当使用像PM2这样的流程管理器时，我们管理一个进程集群，每个进程都充当HTTP服务器。 PM2实现平稳重启的方式是：

1. 向每个工作进程发送SIGNINT信号，
2. 工作进程负责捕获信号，清理或释放任何使用过的资源并完成其过程，
3. 最后PM2产生了一个新的进程

因为这是按照我们的集群流程顺序完成的，所以客户不能受到重启的影响，因为始终会有一些进程在工作并参与请求。

当我们部署新代码并希望重新启动服务器以便新的更改生效而不会有传入请求的风险时，这非常有用。 我们可以在app中实现这个下一个代码：

    // Graceful shutdown
    process.on('SIGINT', () => {
      const cleanUp = () => {
        // Clean up other resources like DB connections
      }
      console.log('Closing server...')
      server.close(() => {
        console.log('Server closed !!! ')
        cleanUp()
        process.exit()
      })
      // Force close server after 5secs
      setTimeout((e) => {
        console.log('Forcing server close !!!', e)
        cleanUp()
        process.exit(1)
      }, 5000)
    })

当它捕获的SINGINT信号时，我们调用server.close（）以避免接受更多请求，一旦它关闭，我们清理我们的应用程序使用的任何资源，如关闭数据库连接，关闭打开的文件等，调用cleanUp（）函数 最后，我们使用process.exit（）退出进程。 此外，如果由于某种原因我们的代码花费太多时间来关闭服务器，我们强制它在setTimeout（）中运行非常相似的代码。

## Conclusions

在创建HTTP服务器时，无论是服务于页面还是API的Web服务器，我们都需要考虑到它需要及时更新以及需要错误修复的事实，因此我们需要思考如何可以做到对客户的以最小化的影响。

在集群模式下运行nodejs进程是提高应用程序性能的常用方法，我们需要考虑如何正常关闭所有这些进程以不影响传入请求。

使用process.exit（）终止节点进程在使用HTTP服务器时是不够的，因为它会突然终止所有通信，我们需要先停止接受新连接，释放我们的应用程序使用的任何资源，最后停止整个进程。

-------------------
# Understanding the NodeJS cluster module

by Antonio Santiago http://www.acuriousanimal.com/

NodeJS processes runs on a single process, which means it does not take advantage from multi-core systems by default. If you have an 8 core CPU and run a NodeJS program via $ node app.js it will run in a single process, wasting the rest of CPUs.

Hopefully for us NodeJS offers the cluster module that contains a set of functions and properties that help us to create programs that uses all the CPUs. Not a surprise the mechanism the cluster module uses to maximize the CPU usage was via forking processes, similar to the old fork() system call Unix systems.

## Introducing the cluster module

The cluster module is a NodeJS module that contains a set of functions and properties that help us forking processes to take advantage of multi-core systems. It is propably the first level of scalability you must take care in your node application, specifally if you are working in a HTTP server application, before going to a higher scalability levels (I mean scaling vertically and horizontally in different machines).

With the cluster module a parent/master process can be forked in any number of child/worker processes and communicate with them sending messages via IPC communication. Remember there is no shared memory among processes.

Next lines are a compilation of sentences from the NodeJS documentation I have taken the liberty to copy&pasta to put it in a way I think can help you understand thw whole thing in a few lines.

1. A single instance of Node.js runs in a single thread. To take advantage of multi-core systems the user will sometimes want to launch a cluster of Node.js processes to handle the load.
2. The cluster module allows easy creation of child processes that all share server ports.
3. The worker (child) processes are spawned using the child_proces.fork() method, so that they can communicate with the parent via IPC and pass server handles back and forth. The child_process.fork() method is a special case of child_process.spawn() used specifically to spawn new Node.js processes. Like child_process.spawn(), a ChildProcess object is returned. The returned ChildProcess will have an additional communication channel built-in that allows messages to be passed back and forth between the parent and child, through the send() method. See subprocess.sen() for details.
4. It is important to keep in mind that spawned Node.js child processes are independent of the parent with exception of the IPC communication channel that is established between the two. Each process has its own memory, with their own V8 instances. Because of the additional resource allocations required, spawning a large number of child Node.js processes is not recommended.

So, most of the magic is done by the child_process module, which is resposible to spawn new process and help communicate among them, for example, creating pipes. You can find a great article at Node.js Child Processes: Everything you need to know.

## A basic example

Stop talking and lets see a real exampe. Next we show a very basic code that:

Creates a master process that retrives the number of CPUs and forks a worker process for each CPU, and
Each child process prints a message in console and exit.

    const cluster = require('cluster');
    const http = require('http');
    const numCPUs = require('os').cpus().length;
    if (cluster.isMaster) {
      masterProcess();
    } else {
      childProcess();  
    }
    function masterProcess() {
      console.log(`Master ${process.pid} is running`);
      for (let i = 0; i < numCPUs; i++) {
        console.log(`Forking process number ${i}...`);
        cluster.fork();
      }
      process.exit();
    }
    function childProcess() {
      console.log(`Worker ${process.pid} started and finished`);
      process.exit();
    }

Save the code in app.js file and run executing: $ node app.js. The output should be something similar to:

    $ node app.js

output is :

    Master 8463 is running
    Forking process number 0...
    Forking process number 1...
    Forking process number 2...
    Forking process number 3...
    Worker 8464 started and finished
    Worker 8465 started and finished
    Worker 8467 started and finished
    Worker 8466 started and finished

### Code explanation

When we run the app.js program an OS process is created that starts running our code. At the beginning the cluster mode is imported `const cluster = require('cluster')` and in the if sentence we check if the isMaster property.

Because the process is the first process the isMaster property is true and then we run the code of masterProcess function. This function has not much secret, it loops depending on the number of CPUs of your machine and forks the current process using the cluster.fork() method.

What the fork() really does is to create a new node process, like if you run it via command line with $node app.js, that is you have many processes running your app.js program.

When a child process is created and executed, it does the same as the master, that is, imports the cluster module and executes the if statement. Once of the differences is for the child process the value of cluster.isMaster is false, so they ends running the childProcess function.

Note, we explicitly terminate the master and worker processes with process.exit(), which by default return value of zero.

NOTE: NodeJS also offers the Child Processes module that simplifies the creation and comunication with other processes. For example we can spawn the ls -l terminal command and pipe with another process that handles the results.

## Comunicating master and worker processes

When a worker process is created, an IPC channel is created among the worker and the master, allowing us to communicated between them with the send() method, which accepts a JavaScript object as parameter. Remember they are different processes (not threads) so we can’t use shared memory as a way of communcation.

From the master process, we can send a message to a worker process using the process reference, i.e. someChild.send({ ... }), and within the worker process we can messages to the master simply using the current process reference, i.e. process.send().

We have updated slighly the previous code to allow master send and receive messages from/to the workers and also the workers receive and send messages from the master process:

    function childProcess() {
      console.log(`Worker ${process.pid} started`);
      process.on('message', function(message) {
        console.log(`Worker ${process.pid} recevies message '${JSON.stringify(message)}'`);
      });
      console.log(`Worker ${process.pid} sends message to master...`);
      process.send({ msg: `Message from worker ${process.pid}` });
      console.log(`Worker ${process.pid} finished`);
    }

The worker process is simply to understand. First we listen for the message event registering a listener with the process.on('message', handler) method. Later we send a messages with process.send({...}). Note the message is a plain JavaScript object.

    let workers = [];
    function masterProcess() {
      console.log(`Master ${process.pid} is running`);
      // Fork workers
      for (let i = 0; i < numCPUs; i++) {
        console.log(`Forking process number ${i}...`);
        const worker = cluster.fork();
        workers.push(worker);
        // Listen for messages from worker
        worker.on('message', function(message) {
          console.log(`Master ${process.pid} recevies message '${JSON.stringify(message)}' from worker ${worker.process.pid}`);
        });
      }
      // Send message to the workers
      workers.forEach(function(worker) {
        console.log(`Master ${process.pid} sends message to worker ${worker.process.pid}...`);
        worker.send({ msg: `Message from master ${process.pid}` });    
      }, this);
    }

The masterProcess function has been divided in two parts. In the first loop we fork as much workers as CPUs we have. The cluster.fork() returns a worker object representing the worker process, we store the reference in an array and register a listener to receive messages that comes from that worker instance.

Later, we loop over the array of workers and send a message from the master process to that concrete worker.

If you run the code the output will be something like:

    $ node app.js

output is :

    Master 4045 is running
    Forking process number 0...
    Forking process number 1...
    Master 4045 sends message to worker 4046...
    Master 4045 sends message to worker 4047...
    Worker 4047 started
    Worker 4047 sends message to master...
    Worker 4047 finished
    Master 4045 recevies message '{"msg":"Message from worker 4047"}' from worker 4047
    Worker 4047 recevies message '{"msg":"Message from master 4045"}'
    Worker 4046 started
    Worker 4046 sends message to master...
    Worker 4046 finished
    Master 4045 recevies message '{"msg":"Message from worker 4046"}' from worker 4046
    Worker 4046 recevies message '{"msg":"Message from master 4045"}'

Here we are not terminating the process with process.exit() so to close the application you need to use ctrl+c.

## Conclusion

The cluster module offers to NodeJS the needed capabilities to use the whole power of a CPU. Although not seen in this post, the cluster module is complemented with the child process module that offers plenty of tools to work with processes: start, stop and pipe input/out, etc.

Cluster module allow us to easily create worker processes. In addition it magically creates an IPC channel to communicate the master and worker process passing JavaScript objects.

In my next post I will show how important is the cluster module when working in an HTTP server, no matter if an API or web server working with ExpressJS. The cluster module can increase performance of our application having as many worker processes as CPUs cores.

# Clusted HTTP

The cluster module allow us improve performance of our application in multicore CPU systems. This is specially important no matter if working on an APIs or an, i.e. ExpressJS based, web servers, what we desire is to take advantage of all the CPUs on each machine our NodeJS application is running.

The cluster module allow us to load balance the incoming request among a set of worker processes and, because of this, improving the throughput of our application.

In this post we are going to see how to use the cluster module when creating HTTP servers, both using plain HTTP module and with ExpressJS.Lets go to see how we can create a really basic HTTP server that takes profit of the cluster module.

    const cluster = require('cluster');
    const http = require('http');
    const numCPUs = require('os').cpus().length;
    if (cluster.isMaster) {
      masterProcess();
    } else {
      childProcess();  
    }
    function masterProcess() {
      console.log(`Master ${process.pid} is running`);
      for (let i = 0; i < numCPUs; i++) {
        console.log(`Forking process number ${i}...`);
        cluster.fork();
      }
    }
    function childProcess() {
      console.log(`Worker ${process.pid} started...`);
      http.createServer((req, res) => {
        res.writeHead(200);
        res.end(`Hello World from ${process.pid}`);
      }).listen(3000);
    }

We have diveded the code in two parts, the one corresponding to the master process and the one where we initialize the worker processes. This way the masterProcess function forks a worker process per CPU code. On the other hand the childProcess simply creates an HTTP server listenen on port 3000 and returning a nice Hello World text string with a 200 status code.

If you run the code the output must show something like:

    $ node app.js

output is :

    Master 1859 is running
    Forking process number 0...
    Forking process number 1...
    Forking process number 2...
    Forking process number 3...
    Worker 1860 started...
    Worker 1862 started...
    Worker 1863 started...
    Worker 1861 started...

if visit by client :

    $ curl http://localhost:3000/
    Hello World from 7036
    $ curl http://localhost:3000/
    Hello World from 7037
    $ curl http://localhost:3000/
    Hello World from 7038
    $ curl http://localhost:3000/
    Hello World from 7039


Basically our initial process (the master) is spawing a new worker process per CPU that runs an HTTP server that handle requests. As you can see this can improve a lot your server performance because it is not the same having one processing attending one million of requests than having four processes attending one millun requests.

## How cluster module works with network connections ?

The previous example is simple but hides something tricky, some magic NodeJS make to simplify our live as developer.

In any OS a process can use a port to communicate with other systems and, that means, the given port can only be used by that process. So, the question is, how can the forked worker processes use the same port?

The answer, the simplified answer, is the master process is the one who listens in the given port and load balances the requests among all the child/worker processes. From the offical documentation:

    The worker processes are spawned using the child_process.fork() method, so that they can communicate with the parent via IPC and pass server handles back and forth.

The cluster module supports two methods of distributing incoming connections.

The first one (and the default one on all platforms except Windows), is the round-robin approach, where the master process listens on a port, accepts new connections and distributes them across the workers in a round-robin fashion, with some built-in smarts to avoid overloading a worker process.

The second approach is where the master process creates the listen socket and sends it to interested workers. The workers then accept incoming connections directly.

As long as there are some workers still alive, the server will continue to accept connections. If no workers are alive, existing connections will be dropped and new connections will be refused.

## Other alternatives to cluster module load balancing

Cluster module allow the master process to receive request and load balance it among all the worker processes. This is a way to improve performance but it is not the only one.

In the post Node.js process load balance performance: comparing cluster module, iptables and Nginx you can find a performance comparison among: node cluster module, iptables and nginx reverse proxy.

## Conclusions

Nowadays performance is mandatory on any web applications, we need to support high throughput and serve data fast.

The cluster module is one possible solution, it allows us to have one master process and create a worker processes for each core, so that they run an HTTP server. The cluster module offers two great features:

simplifies communication among master and workers, by creating an IPC channel and allowing send messages with process.send(),
allow worker processes share the same port. This is done making the master process the one which receives requests and multiplexe them among workers.


# Using PM2 to manage NodeJS cluster

The cluster module allows us to create worker processes to improve our NodeJS applications performance. This is specially important in web applications, where a master process receives all the requests and load balances them among the worker processes.

But all this power comes with the cost that must be the application who manages all the complexity associated with process managements: what happens if a worker process exists unexpectedly, how exit gracefully the worker processes, what if you need to restart all your workers, etc.

In this post we present PM2 tool. although it is a general process manager, that means it can manage any kind of process like python, ruby, … and not only NodeJS processes, the tool is specially designed to manage NodeJS applications that want to work with the cluster module.

## Introducing PM2

As said previously, PM2 is a general process manager, that is, a program that controls the execution of other process (like a python program that check if you have new emails) and does things like: check your process is running, re-execute your process if for some reason it exits unexpectedly, log its output, etc.

The most important thing for us is PM2 simplifies the execution of NodeJS applications to run as a cluster. Yes, you write your application without worrying about cluster module and is PM2 who creates a given number of worker processes to run your application.

## The PM2 way

Before continue, you need to instal PM2 on your system. Typically it is installed as a global module with $ npm install pm2 -g or $ yarn global add pm2.When using PM2 we can forget the part of the code related with the master process, that will responsibility of PM2, so our very basic HTTP server can be rewritten as:

    const http = require('http');
    console.log(`Worker ${process.pid} started...`);
    http.createServer((req, res) => {
      res.writeHead(200);
      res.end('Hello World');
      process.exit(1);
    }).listen(3000);

Now run PM2 with 

    $ pm2 start app.js -i 3 

Note the option -i that is used to indicate the number of instances to create. The idea is that number be the same as your number of CPU cores. If you don’t know them you can set -i 0 to leave PM2 detect it automatically.


## Conclusions

Although the NodeJS cluster module is a powerful mechanism to improve performance it comes at the cost of complexity required to manage all the situations an application can found: what happens if a worker exists, how can we reload the application cluster without down time, etc.

PM2 is a process manager specially designed to work with NodeJS clusters. It allow to cluster an application, restart or reload, without the required code complexity in addition to offer tools to see log outputs, monitorization, etc.

# Graceful shutdown NodeJS HTTP server when using PM2

So you have created a NodeJS server that receives tons of requests and you are really happy but, as every piece of software, you found a bug or add a new feature to it. It is clear you will need to shutdown your NodeJS process/es and restart again so that the new code takes place. The question is: how can you do that in a graceful way that allows continue serving incoming requests?

## Starting a HTTP server

Before see how we must shutdown a HTTP server lets see how usually create one. Next code shows a very basic code with an ExpressJS service that will return Hello World !!! when accessing the /hello path. You can also pass a path param, i.e. /hello/John with a name so it returns Hello John !!!.


    const express = require('express')
    const expressApp = express()
    // Responds with Hello World or optionally the name you pass as path param
    expressApp.get('/hello/:name?', function (req, res) {
      const name = req.params.name
      if (name) {
        return res.send(`Hello ${name}!!!`)
      }
      return res.send('Hello World !!!')
    })
    // Start server
    expressApp.listen(3000, function () {
      console.log('App listening on port 3000!')
    })
What app.listen() function does is start a new HTTP server using the core http module and return a reference to the HTTP server object. In concrete, the source code of the listen() is as follows:

    app.listen = function listen() {
      var server = http.createServer(this);
      return server.listen.apply(server, arguments);
    };

NOTE: Another way to create an express server is pass our expressApp reference directly to the http. createServer(), something like: const server = http.createServer(app).listen(3000).

## How to shutdown properly an HTTP server ?

The proper way to shutdown an HTTP server is to invoke the server.close() function, this will stop server from accepting new connections while keeps existing ones until response them.

Next code presents a new /close endpoint that once invoked will stop the HTTP server and exit the applications (stopping the nodejs process):

    app.get('/close', (req, res) => {
      console.log('Closing the server...')
      server.close(() => {
        console.log('--> Server call callback run !!')
        process.exit()
      })
    })

It is clear shutting down a server through an endpoint is not the right way to it.

## Graceful shutdown/restart with and without PM2

The goal of a graceful shutdown is to close the incoming connections to a server without killing the current ones we are handling.

When using a process manager like PM2, we manage a cluster of processes each one acting as a HTTP server. The way PM2 achieves the graceful restart is:

sending a SIGNINT signal to each worker process,
the worker are responsible to catch the signal, cleanup or free any used resource and finish the its process,
finally PM2 manager spawns a new process
Because this is done sequentially with our cluster processes customers must not be affected by the restart because there will always be some processes working and attending requests.

This is very useful when we deploy new code and want to restart our servers so the new changes take effect without risk for incoming requests. We can achieve this putting next code in out app:

    // Graceful shutdown
    process.on('SIGINT', () => {
      const cleanUp = () => {
        // Clean up other resources like DB connections
      }
      console.log('Closing server...')
      server.close(() => {
        console.log('Server closed !!! ')
        cleanUp()
        process.exit()
      })
      // Force close server after 5secs
      setTimeout((e) => {
        console.log('Forcing server close !!!', e)
        cleanUp()
        process.exit(1)
      }, 5000)
    })

When the SINGINT signal it catch we invoke the server.close() to avoid accepting more requests and once it is closed we clean up any resource used by our app, like close database connection, close opened files, etc invoking the cleanUp() function and, finally, we exits the process with process.exit(). In addition, if for some reason our code spends too much time to close the server we force it running a very similar code within a setTimeout().

## Conclusions

When creating a HTTP server, no matter if a web server to serve pages or an API, we need to take into account the fact it will be updated in time with new features and bug fixes, so we need to think in a way to minimize the impact on customers.

Running nodejs processes in cluster mode is a common way to improve our applications performance and we need to think on how to graceful shutdown all them to not affect incoming requests.

Terminating a node process with process.exit() is not enough when working with an HTTP server because it will terminate abruptly all the communications, we need to first stop accepting new connections, free any resource used by our application and, finally, stop the process.