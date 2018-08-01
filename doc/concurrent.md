#nodejs并行实践问答 

Node具有完全不同的编程范式，一旦正确理解，就更容易看到解决问题的不同方式。您永远不需要Node应用程序中的多个线程，因为您有不同的方式来执行相同的操作。您创建多个流程;但它与Apache Web Server的Prefork mpm的功能非常不同。

现在，让我们假设我们只有一个CPU核心，我们将开发一个应用程序（以Node的方式）来完成一些工作。我们的工作是逐个字节地处理一个大文件。我们软件的最佳方法是从文件的开头开始工作，逐个字节地跟踪到最后。

- 嘿，hasan，我想你要么是新手要么和我爷爷一样的老派！为什么不创建一些线程并使其更快？

- 哦，我们只有一个CPU核心。

- 所以呢？创建一些线程，让它更快！

- 它不像那样工作。如果我创建线程，我将使它变慢。因为我将在系统中添加大量开销以在线程之间进行切换，并在我的进程内尝试在这些线程之间进行通信。除了所有这些事实之外，我还必须考虑如何将单个作业分成多个可以并行完成的作业。

- 好的，我觉得你很穷！我们用我的电脑，它有32个核心！

- 哇，亲爱的朋友，你真棒，非常感谢你。我很感激！

然后我们回头工作。现在我们拥有32个cpu核心，感谢我们的朋友。我们必须遵守的规则刚刚改变了。现在我们想要利用我们给予的所有这些财富。

要使用多个内核，我们需要找到一种方法将我们的工作分成可以并行处理的部分。如果它不是Node.js，我们会为此使用线程; 32个线程，每个cpu核心一个。但是，由于我们有Node.js，我们将创建32个Node进程。

线程可以是Node进程的一个很好的替代方案，甚至可能是更好的方法;但只有在已定义工作的特定工作中，我们才能完全控制如何处理工作。除此之外，对于其他任何类型的问题，如果工作来自外部，我们无法控制，我们希望尽快回答，Node的方式将会是无可争议地更加优越。

- 嘿，Hasan，你还在单线程吗？你怎么了，伙计？我刚刚为你提供了你想要的东西。你没有任何借口了。创建线程，让它运行得更快。

- 我已将作品分成几部分，每个进程都会同时处理其中一个部分。

- 你为什么不创建线程？

- 对不起，我觉得它不可用。要不你把你的电脑拿走？

- 不行，我很酷，我只是不明白为什么你不使用线程？

- 谢谢你的电脑。 :)我已经将工作分成几部分，我创建了并行处理这些部分的流程。所有CPU核心都将得到充分利用。我可以用线程代替进程来做到这一点;但Node有另外一种方式，而我的老板Parth Thakkar希望我使用Node。

- 好的，如果你需要另一台电脑，请告诉我。 ：p

如果我创建了33个进程而不是32个进程，那么操作系统的调度程序将暂停一个线程，启动另一个进程，在一些循环后暂停它，再次启动另一个进程......这是不必要的开销。我不想要这个。实际上，在具有32个内核的系统上，我甚至不想创建正好32个进程，31个可以更好。因为不仅仅是我的应用程序可以在这个系统上工作。为其他东西留一点空间可能会很好，特别是如果我们有32个房间。

我相信我们现在在同一问题上了，现在的问题是关于充分利用处理器来执行CPU密集型任务。

- 嗯，hasan，我很抱歉嘲笑你一点。我相信我现在更了解你。但是我还需要解释一下：运行数百个线程的哼哼唧唧是什么？我到处都看到线程比创建进程更快创建？你创建进程而不是线程，你认为它是你这样使用Node是最优的。难道Node不适合这种工作吗？

- 不用担心，我也很酷。每个人都说这些东西，所以我想我已经习惯了听到它们。

- 那么？Nodejs不擅长此道？

- 即使线程也很好，Nodejs对此非常有用。至于线程和进程创建开销;在你重复的事情上，每一毫秒都很重要。但是，我只创建了32个进程，这将花费很少的时间。它只会发生一次。它没有任何区别。

- 我什么时候想创建数千个线程呢？

- 你永远不想创建成千上万的线程。但是，在正在进行外部工作的系统上，例如处理HTTP请求的Web服务器;如果你为每个请求使用一个线程，你就真的会创建数千个线程的。

- Node.js有所不同吗？对？

- 对，就是这样。这是Node真正闪耀的地方。就像线程比进程轻得多，函数调用比线程轻得多。Node.js调用函数，而不是创建线程。在Web服务器的示例中，每个传入的请求都会导致函数调用。

- 嗯，有趣;但是如果不使用多个线程，则只能同时运行一个函数。当很多请求同时到达Web服务器时，它如何工作？

- 关于函数如何运行，一次一个，从不并行运行，你是完全正确的。我的意思是在一个进程中，一次只运行一个代码范围。操作系统调度程序不会暂停此功能并切换到另一个功能，除非它暂停进程以给另一个进程留出时间，而不是我们进程中的另一个进程。

- 那么一个进程如何一次处理2个请求？

- 只要我们的系统有足够的资源（RAM，网络等），一个进程就可以同时处理数万个请求。这些功能如何运行是关键的差异。

- 嗯，我现在应该兴奋吗？

- 也许:)Node.js在队列上运行一个循环。在这个队列中是我们的工作，即我们开始处理传入请求的调用。这里最重要的一点是我们设计运行函数的方式。我们不是开始处理请求并让调用者等到我们完成工作，而是在完成可接受的工作量后快速结束我们的功能。当我们需要等待另一个组件做一些工作并返回一个值而不是等待它时，我们只需完成我们的功能，将剩下的工作添加到队列中。

- 听起来太复杂了？

- 不，不，我可能听起来很复杂;但系统本身非常简单，而且非常有意义。

现在我想停止引用这两个开发人员之间的对话，并在最后一个关于这些功能如何工作的快速示例之后完成我的回答。

通过这种方式，我们正在做OSScheduler通常会做的事情。我们暂停我们的工作，并让其他函数调用（如多线程环境中的其他线程）运行，直到我们再次轮到我们。这比将工作留给OSScheduler好得多，后者试图给系统上的每个线程提供时间。我们知道我们做得比OSScheduler做得好得多，我们应该在我们停止时停止。

下面是一个简单的例子，我们打开一个文件并读取它来对数据做一些工作。

同步方式：

		打开文件
		重复这个：
		    阅读一些
		    做的工作
异步方式：

		打开文件并在准备就绪时执行：//我们的函数返回
		    重复这个：
		        阅读一些，当它准备就绪时：//再次返回
		            做一些工作

如您所见，我们的函数要求系统打开文件，而不是等待它打开。文件准备好后，通过提供后续步骤完成自己。当我们返回时，Node在队列上运行其他函数调用。在遍历所有函数之后，事件循环移动到下一轮...

总之，Node具有与多线程开发完全不同的范式;但这并不意味着它缺乏东西。对于同步作业（我们可以决定处理的顺序和方式），它可以和多线程并行一样工作。对于来自外部的工作，如对服务器的请求，它就是更优越的。


# Nodejs并行实践问答 - Which would be better for concurrent tasks on node.js? 

Node has a completely different paradigm and once it is correctly captured, it is easier to see this different way of solving problems. You never need multiple threads in a Node application(1) because you have a different way of doing the same thing. You create multiple processes; but it is very very different than, for example how Apache Web Server's Prefork mpm does.

For now, let's think that we have just one CPU core and we will develop an application (in Node's way) to do some work. Our job is to process a big file running over its contents byte-by-byte. The best way for our software is to start the work from the beginning of the file, follow it byte-by-byte to the end.

-- Hey, Hasan, I suppose you are either a newbie or very old school from my Grandfather's time!!! Why don't you create some threads and make it much faster?

-- Oh, we have only one CPU core.

-- So what? Create some threads man, make it faster!

-- It does not work like that. If I create threads I will be making it slower. Because I will be adding a lot of overhead to the system for switching between threads, trying to give them a just amount of time, and inside my process, trying to communicate between these threads. In addition to all these facts, I will also have to think about how I will divide a single job into multiple pieces that can be done in parallel.

-- Okay okay, I see you are poor. Let's use my computer, it has 32 cores!

-- Wow, you are awesome my dear friend, thank you very much. I appreciate it!

Then we turn back to work. Now we have 32 cpu cores thanks to our rich friend. Rules we have to abide have just changed. Now we want to utilize all this wealth we are given.

To use multiple cores, we need to find a way to divide our work into pieces that we can handle in parallel. If it was not Node, we would use threads for this; 32 threads, one for each cpu core. However, since we have Node, we will create 32 Node processes.

Threads can be a good alternative to Node processes, maybe even a better way; but only in a specific kind of job where the work is already defined and we have complete control over how to handle it. Other than this, for every other kind of problem where the job comes from outside in a way we do not have control over and we want to answer as quickly as possible, Node's way is unarguably superior.

-- Hey, Hasan, are you still working single-threaded? What is wrong with you, man? I have just provided you what you wanted. You have no excuses anymore. Create threads, make it run faster.

-- I have divided the work into pieces and every process will work on one of these pieces in parallel.

-- Why don't you create threads?

-- Sorry, I don't think it is usable. You can take your computer if you want?

-- No okay, I am cool, I just don't understand why you don't use threads?

-- Thank you for the computer. :) I already divided the work into pieces and I create processes to work on these pieces in parallel. All the CPU cores will be fully utilized. I could do this with threads instead of processes; but Node has this way and my boss Parth Thakkar wants me to use Node.

-- Okay, let me know if you need another computer. :p

If I create 33 processes, instead of 32, the operating system's scheduler will be pausing a thread, start the other one, pause it after some cycles, start the other one again... This is unnecessary overhead. I do not want it. In fact, on a system with 32 cores, I wouldn't even want to create exactly 32 processes, 31 can be nicer. Because it is not just my application that will work on this system. Leaving a little room for other things can be good, especially if we have 32 rooms.

I believe we are on the same page now about fully utilizing processors for CPU-intensive tasks.

-- Hmm, Hasan, I am sorry for mocking you a little. I believe I understand you better now. But there is still something I need an explanation for: What is all the buzz about running hundreds of threads? I read everywhere that threads are much faster to create and dumb than forking processes? You fork processes instead of threads and you think it is the highest you would get with Node. Then is Node not appropriate for this kind of work?

-- No worries, I am cool, too. Everybody says these things so I think I am used to hearing them.

-- So? Node is not good for this?

-- Node is perfectly good for this even though threads can be good too. As for thread/process creation overhead; on things that you repeat a lot, every millisecond counts. However, I create only 32 processes and it will take a tiny amount of time. It will happen only once. It will not make any difference.

-- When do I want to create thousands of threads, then?

-- You never want to create thousands of threads. However, on a system that is doing work that comes from outside, like a web server processing HTTP requests; if you are using a thread for each request, you will be creating a lot of threads, many of them.

-- Node is different, though? Right?

-- Yes, exactly. This is where Node really shines. Like a thread is much lighter than a process, a function call is much lighter than a thread. Node calls functions, instead of creating threads. In the example of a web server, every incoming request causes a function call.

-- Hmm, interesting; but you can only run one function at the same time if you are not using multiple threads. How can this work when a lot of requests arrive at the web server at the same time?

-- You are perfectly right about how functions run, one at a time, never two in parallel. I mean in a single process, only one scope of code is running at a time. The OS Scheduler does not come and pause this function and switch to another one, unless it pauses the process to give time to another process, not another thread in our process. (2)

-- Then how can a process handle 2 requests at a time?

-- A process can handle tens of thousands of requests at a time as long as our system has enough resources (RAM, Network, etc.). How those functions run is THE KEY DIFFERENCE.

-- Hmm, should I be excited now?

-- Maybe :) Node runs a loop over a queue. In this queue are our jobs, i.e, the calls we started to process incoming requests. The most important point here is the way we design our functions to run. Instead of starting to process a request and making the caller wait until we finish the job, we quickly end our function after doing an acceptable amount of work. When we come to a point where we need to wait for another component to do some work and return us a value, instead of waiting for that, we simply finish our function adding the rest of work to the queue.

-- It sounds too complex?

-- No no, I might sound complex; but the system itself is very simple and it makes perfect sense.

Now I want to stop citing the dialogue between these two developers and finish my answer after a last quick example of how these functions work.

In this way, we are doing what OS Scheduler would normally do. We pause our work at some point and let other function calls (like other threads in a multi-threaded environment) run until we get our turn again. This is much better than leaving the work to OS Scheduler which tries to give just time to every thread on system. We know what we are doing much better than OS Scheduler does and we are expected to stop when we should stop.

Below is a simple example where we open a file and read it to do some work on the data.

Synchronous Way:

Open File
Repeat This:    
    Read Some
    Do the work
Asynchronous Way:

Open File and Do this when it is ready: // Our function returns
    Repeat this:
        Read Some and when it is ready: // Returns again
            Do some work
As you see, our function asks the system to open a file and does not wait for it to be opened. It finishes itself by providing next steps after file is ready. When we return, Node runs other function calls on the queue. After running over all the functions, the event loop moves to next turn...

In summary, Node has a completely different paradigm than multi-threaded development; but this does not mean that it lacks things. For a synchronous job (where we can decide the order and way of processing), it works as well as multi-threaded parallelism. For a job that comes from outside like requests to a server, it simply is superior.

(1) Unless you are building libraries in other languages like C/C++ in which case you still do not create threads for dividing jobs. For this kind of work you have two threads one of which will continue communication with Node while the other does the real work.

(2) In fact, every Node process has multiple threads for the same reasons I mentioned in the first footnote. However this is no way like 1000 threads doing similar works. Those extra threads are for things like to accept IO events and to handle inter-process messaging.

UPDATE (As reply to a good question in comments)
@Mark, thank you for the constructive criticism. In Node's paradigm, you should never have functions that takes too long to process unless all other calls in the queue are designed to be run one after another. In case of computationally expensive tasks, if we look at the picture in complete, we see that this is not a question of "Should we use threads or processes?" but a question of "How can we divide these tasks in a well balanced manner into sub-tasks that we can run them in parallel employing multiple CPU cores on the system?" Let's say we will process 400 video files on a system with 8 cores. If we want to process one file at a time, then we need a system that will process different parts of the same file in which case, maybe, a multi-threaded single-process system will be easier to build and even more efficient. We can still use Node for this by running multiple processes and passing messages between them when state-sharing/communication is necessary. As I said before, a multi-process approach with Node is as well as a multi-threaded approach in this kind of tasks; but not more than that. Again, as I told before, the situation that Node shines is when we have these tasks coming as input to system from multiple sources since keeping many connections concurrently is much lighter in Node compared to a thread-per-connection or process-per-connection system.

As for setTimeout(...,0) calls; sometimes giving a break during a time consuming task to allow calls in the queue have their share of processing can be required. Dividing tasks in different ways can save you from these; but still, this is not really a hack, it is just the way event queues work. Also, using process.nextTick for this aim is much better since when you use setTimeout, calculation and checks of the time passed will be necessary while process.nextTick is simply what we really want: "Hey task, go back to end of the queue, you have used your share!"