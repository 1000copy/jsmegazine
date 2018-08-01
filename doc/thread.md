# Threads in Node 10.5.0: a practical intro

by Fernando Doglio
翻译：reco

几天前，Node.js的10.5.0版本已经发布，其中包含的一个主要功能是增加了初始（和实验性）线程支持。这很有趣，特别是来自一种总是它是非常棒的异步I / O引以为傲的语言，因此需要线程。那么为什么我们需要Node中的线程？

快速而简单的答案是：让Node在过去难受的区域中表现出色：处理*繁重的CPU密集型计算*。就是这个原因，导致Node.js在AI，机器学习，数据科学等领域表现并不出色。nodejs正在努力寻求解决这个问题，但仍然没有像部署微服务那样高效。

因此，我将尝试简化官方文档提供的技术文档，把它变成更实用，更简单的示例集。希望这足以让你开始。


##那么我们如何使用新的Threads模块呢？

首先，您将需要名为“worker_threads”的模块。

请注意，这仅在执行脚本时使用--experimental-worker标志时才有效，否则将无法找到该模块。

注意标志如何引用工作者而不是线程，这是它们将在整个文档中引用的方式：工作线程或简单的工作者。

如果你过去使用过多处理，你会发现这种方法有很多相似之处，但如果你没有，请不要担心，我会尽可能多地解释。

##你能用它们做什么？

工作线程就像我之前提到的那样，用于CPU密集型任务，将它们用于I / O将浪费资源，因为根据官方文档，Node提供的处理异步I / O的内部机制更多比使用工作线程更有效，所以...不要打扰。

让我们从一个简单的例子开始，你将如何创建一个工人并使用它。

Example 1:

		const { Worker, isMainThread,  workerData } = require('worker_threads');
		let currentVal = 0;
		let intervals = [100,1000, 500]
		function counter(id, i){
			console.log("[", id, "]", i)
			return i;
		}
		if(isMainThread) {
			console.log("this is the main thread")
			for(let i = 0; i < 2; i++) {
				let w = new Worker(__filename, {workerData: i});
			}
			setInterval((a) => currentVal = counter(a,currentVal + 1), intervals[2], "MainThread");
		} else {
			console.log("this isn't")
			setInterval((a) => currentVal = counter(a,currentVal + 1), intervals[workerData], workerData);
		}


上面的例子将简单地输出一组显示增量计数器的行，这将使用不同的速度增加它们的值。


IF语句中的代码创建了2个工作线程，由于传递了`__filename`参数，因此它们的代码取自同一个文件。 工作人员现在需要文件的完整路径，他们无法处理相对路径，因此这就是使用此值的原因。

将2个worker作为全局参数发送，其形式为您在第二个参数中看到的workerData属性。 然后可以通过具有相同名称的常量访问该值（请参阅如何在文件的第一行中创建常量，并在稍后的最后一行中使用该常量）。

这个例子是你可以用这个模块做的最基本的事情之一，但它不是那么有趣，是吗？ 让我们看另一个例子。


Example 2: Actually doing something

	const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
	const request = require("request");
	if(isMainThread) {
		console.log("This is the main thread")
		let w = new Worker(__filename, {workerData: null});
		w.on('message', (msg) => { //A message from the worker!
			console.log("First value is: ", msg.val);
			console.log("Took: ", (msg.timeDiff / 1000), " seconds");
		})
		w.on('error', console.error);
		w.on('exit', (code) => {
			if(code != 0)
		      	console.error(new Error(`Worker stopped with exit code ${code}`))
	   });
		request.get('http://www.google.com', (err, resp) => {
			if(err) {
				return console.error(err);
			}
			console.log("Total bytes received: ", resp.body.length);
		})
	} else { //the worker's code
		function random(min, max) {
			return Math.random() * (max - min) + min
		}
		const sorter = require("./test2-worker");
		const start = Date.now()
		let bigList = Array(1000000).fill().map( (_) => random(1,10000))
		sorter.sort(bigList);
		parentPort.postMessage({ val: sorter.firstValue, timeDiff: Date.now() - start});
	}

and test2-worker.js:

	module.exports = {
		firstValue: null,
		sort: function(list) {
			let sorted = list.sort();
			this.firstValue = sorted[0]
		}
	}

让我们现在尝试做一些“重”计算，同时在主线程中做一些异步的东西。

这一次，我们请求Google.com的主页，同时对随机生成的100万个数字进行排序。这将花费几秒钟，因此我们很好地展示了它的表现。我们还将测量工作线程执行排序所需的时间，然后我们将该值（连同第一个排序值）发送到主线程，我们将在其中显示结果。

这个例子的主要内容是演示线程之间的通信。

工作人员可以通过on方法在主线程中接收消息。我们可以听到的事件是代码上显示的事件。每当我们使用parentPort.postMessage方法从实际线程发送消息时，就会触发消息事件。您还可以在工作器实例上使用相同的方法向线程的代码发送消息，并使用parentPortobject捕获它们。

如果你想知道，我使用的辅助模块的代码就在这里，虽然没有什么值得注意的。

现在让我们看一个非常相似的例子，但是使用更清晰的代码，让您最终了解如何构建工作线程的代码。

Example 3: bringing it all together

作为最后一个例子，我将坚持使用相同的功能，但向您展示如何清理它并具有更易维护的版本。

	const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
	const request = require("request");
	function startWorker(path, cb) {
		let w = new Worker(path, {workerData: null});
		w.on('message', (msg) => {
			cb(null, msg)
		})
		w.on('error', cb);
		w.on('exit', (code) => {
			if(code != 0)
		      	console.error(new Error(`Worker stopped with exit code ${code}`))
	   });
		return w;
	}
	console.log("this is the main thread")
	let myWorker = startWorker(__dirname + '/workerCode.js', (err, result) => {
		if(err) return console.error(err);
		console.log("[[Heavy computation function finished]]")
		console.log("First value is: ", result.val);
		console.log("Took: ", (result.timeDiff / 1000), " seconds");
	})
	const start = Date.now();
	request.get('http://www.google.com', (err, resp) => {
		if(err) {
			return console.error(err);
		}
		console.log("Total bytes received: ", resp.body.length);
		//myWorker.postMessage({finished: true, timeDiff: Date.now() - start}) //you could send messages to your workers like this
	}) 


并且您的线程代码可以在另一个文件中，例如：

	const {  parentPort } = require('worker_threads');
	function random(min, max) {
		return Math.random() * (max - min) + min
	}
	const sorter = require("./test2-worker");
	const start = Date.now()
	let bigList = Array(1000000).fill().map( (_) => random(1,10000))
	/**
	//you can receive messages from the main thread this way:
	parentPort.on('message', (msg) => {
		console.log("Main thread finished on: ", (msg.timeDiff / 1000), " seconds...");
	})
	*/
	sorter.sort(bigList);
	parentPort.postMessage({ val: sorter.firstValue, timeDiff: Date.now() - start});

分析一下可以得知：

主线程和工作线程现在将其代码放在不同的文件中。这更容易维护和扩展。
startWorker函数返回新实例，如果您愿意，可以稍后向其发送消息。
如果主线程的代码实际上是主线程（我们删除了主IF语句），您不再需要担心。
您可以在worker的代码中看到如何从主线程接收消息，从而允许双向异步通信。

请记住：这仍然是高度实验性的，这里解释的内容可能会在未来的版本中发生
去阅读PR评论和文档，有关于此的更多信息，我只关注它的基本步骤。
玩的开心！四处游玩，报告错误并提出改进建议，这刚刚开始！


------------------
# Threads in Node 10.5.0: a practical intro

by Fernando Doglio

A few days ago, version 10.5.0 of Node.js was released and one of the main features it contained was the addition of initial (and experimental) thread support.This is interesting, specially coming from a language that’s always pride itself of not needed threads thanks to it’s fantastic async I/O. So why would we need threads in Node?

The quick and simple answer is: to have it excel in the only area where Node has suffered in the past: dealing with *heavy CPU intensive computations*. This is mainly why Node.js is not strong in areas such as AI, Machine Learning, Data Science and similar. There are a lot of efforts in progress to solve that, but we’re still not as performant as when deploying microservices for instance.

So I’m going to try and simplify the technical documentation provided by the initial PR and the official docs into a more practical and simple set of examples. Hopefully that’ll be enough to get you started.


## So how do we use the new Threads module?

To start with, you’ll be requiring the module called “worker_threads”.

Note that this will only work if you use the --experimental-worker flag when executing the script, otherwise the module will not be found.

Notice how the flag refers to workers and not threads, this is how they’re going to be referenced throughout the documentation: worker threads or simply workers.

If you’ve used multi-processing in the past, you’ll see a lot of similarities with this approach, but if you haven’t, don’t worry, I’ll explain as much as I can.

## What can you do with them?

Worker threads are meant, like I mentioned before, for CPU intensive tasks, using them for I/O would be a waste of resources, since according to the official documentation, the internal mechanism provided by Node to handle async I/O is much more efficient than using a worker thread for that, so… don’t bother.

Let’s start with a simple example of how you would go about creating a worker and using it.

Example 1:

		const { Worker, isMainThread,  workerData } = require('worker_threads');
		let currentVal = 0;
		let intervals = [100,1000, 500]
		function counter(id, i){
			console.log("[", id, "]", i)
			return i;
		}
		if(isMainThread) {
			console.log("this is the main thread")
			for(let i = 0; i < 2; i++) {
				let w = new Worker(__filename, {workerData: i});
			}
			setInterval((a) => currentVal = counter(a,currentVal + 1), intervals[2], "MainThread");
		} else {
			console.log("this isn't")
			setInterval((a) => currentVal = counter(a,currentVal + 1), intervals[workerData], workerData);
		}


The above example will simply output a set of lines showing incremental counters, which will increase their values using different speeds.


The code inside the IF statement creates 2 worker threads, the code for them is taken from the same file, due to the `__filename` parameter passed. Workers need the full path to the files right now, they can’t handle relative paths, so that is why this value is used.

The 2 workers are sent a value as a global parameter, in the form of the workerData attribute you see as part of the second argument. That value can then be accessed through a constant with the same name (see how the constant is created in the first line of the file and used later on in the last line).

This example is one of the most basic things you can do with this module, but it’s not really that fun, is it? Let’s look at another example.

Example 2: Actually doing something

	const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
	const request = require("request");
	if(isMainThread) {
		console.log("This is the main thread")
		let w = new Worker(__filename, {workerData: null});
		w.on('message', (msg) => { //A message from the worker!
			console.log("First value is: ", msg.val);
			console.log("Took: ", (msg.timeDiff / 1000), " seconds");
		})
		w.on('error', console.error);
		w.on('exit', (code) => {
			if(code != 0)
		      	console.error(new Error(`Worker stopped with exit code ${code}`))
	   });
		request.get('http://www.google.com', (err, resp) => {
			if(err) {
				return console.error(err);
			}
			console.log("Total bytes received: ", resp.body.length);
		})
	} else { //the worker's code
		function random(min, max) {
			return Math.random() * (max - min) + min
		}
		const sorter = require("./test2-worker");
		const start = Date.now()
		let bigList = Array(1000000).fill().map( (_) => random(1,10000))
		sorter.sort(bigList);
		parentPort.postMessage({ val: sorter.firstValue, timeDiff: Date.now() - start});
	}

and test2-worker.js:

	module.exports = {
		firstValue: null,
		sort: function(list) {
			let sorted = list.sort();
			this.firstValue = sorted[0]
		}
	}

Let’s try now to do some “heavy” computation while at the same time, doing some async stuff in the main thread.

This time around, we’re requesting the homepage for Google.com and at the same time, sorting a randomly generated array of 1 million numbers. This is going to take a few seconds, so it’s perfect for us to show how well this behaves. We’re also going to measure the time it takes for the worker thread to perform the sorting and we’re going to send that value (along with the first sorted value) to the main thread, where we’ll display the results.


Here is the output from example #2
The main takeaway from this example, is the communication between threads.

Workers can receive messages in the main thread through the on method. The events we can listen to are the ones shown on the code. The message event is triggered whenever we send a message from the actual thread using the parentPort.postMessage method. You could also send a message to the thread’s code using the same method, on your worker instance and catching them using the parentPortobject.

In case you’re wondering, the code for the helper module I used is here, although there is nothing note-worthy about it.

Let’s now look at a very similar example, but with a cleaner code, giving you a final idea of how you could structure your worker thread’s code.

Example 3: bringing it all together
As a final example, I’m going to stick to the same functionality, but showing you how you could clean it up a bit and have a more maintainable version.

	const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
	const request = require("request");
	function startWorker(path, cb) {
		let w = new Worker(path, {workerData: null});
		w.on('message', (msg) => {
			cb(null, msg)
		})
		w.on('error', cb);
		w.on('exit', (code) => {
			if(code != 0)
		      	console.error(new Error(`Worker stopped with exit code ${code}`))
	   });
		return w;
	}
	console.log("this is the main thread")
	let myWorker = startWorker(__dirname + '/workerCode.js', (err, result) => {
		if(err) return console.error(err);
		console.log("[[Heavy computation function finished]]")
		console.log("First value is: ", result.val);
		console.log("Took: ", (result.timeDiff / 1000), " seconds");
	})
	const start = Date.now();
	request.get('http://www.google.com', (err, resp) => {
		if(err) {
			return console.error(err);
		}
		console.log("Total bytes received: ", resp.body.length);
		//myWorker.postMessage({finished: true, timeDiff: Date.now() - start}) //you could send messages to your workers like this
	}) 


And your thread code can be inside another file, such as:

	const {  parentPort } = require('worker_threads');
	function random(min, max) {
		return Math.random() * (max - min) + min
	}
	const sorter = require("./test2-worker");
	const start = Date.now()
	let bigList = Array(1000000).fill().map( (_) => random(1,10000))
	/**
	//you can receive messages from the main thread this way:
	parentPort.on('message', (msg) => {
		console.log("Main thread finished on: ", (msg.timeDiff / 1000), " seconds...");
	})
	*/
	sorter.sort(bigList);
	parentPort.postMessage({ val: sorter.firstValue, timeDiff: Date.now() - start});

Breaking this one down, we see:

Main thread and worker threads now have their code inside different files. This is easier to maintain and extend.
The startWorkerfunction returns the new instance, allowing you to later send messages to it if you so wanted.
You no longer need to worry if your main thread’s code is actually the main thread (we removed the main IF statement).
You can see in the worker’s code how you would receive a message from the main thread, allowing for a two-way asynchronous communication.

That is going to be it for this article, I hope you got enough to understand how to get started to play around with this new module. Remember that:

This is still highly experimental and things explained here can change in future releases
Go and read the PR comments and docs, there is more information about this in there, I just focused on the basic steps to get it going.
Have fun! Play around, report bugs and suggest improvements, this is just starting!
See you on the next one!