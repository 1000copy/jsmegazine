# 封面故事

封面是一只猫，名字叫做三猫。三猫是公猫，有一对蓝色的眼睛，和一身雪白的、绒绒的毛。它对自己的毛非常爱惜，常常花费整个小时的时间，把它们打整的干干净净。每当它对自己的毛感觉非常良好的时候，对在主人面前晃来晃去的刷存在。

三猫换过几届主人，这些主人目前依然承认这个关系，但是三猫不一定承认了。它住过几个城市，坐过几回飞机，还做过平面模特，一次出工，可以赚几百磅的猫粮呢。算是见过世面。

但是日常起居宅得厉害。如同很多猫一样，它遵守了猫的日常起居：大约16小时碎觉，2小时自己玩，2小时逗主人玩，1小时犯迷糊，一小时吃饭，一小时遛弯，一个小时打扫个猫卫生。我上班的时候，它抬起头看看，然后继续睡，我回来的时候，它过来迎接，然后玩闹一会儿，瞌睡一会儿，然后晚上10点起来吃夜宵的虾，继续睡。

对照它的起居和态度，我看到了一只猫的给我的生活意见。

# 稿件须知

欢迎投稿或者给线索，请发邮件到`1000copy@gmail.com`。

稿件要求和JS相关，因为是免费杂志，因此不会有稿费，但是回报依然是有的，可以在文中附带你的介绍。还可以免费给你登广告。
对了，格式要求是Markdown的。

可能你关心多久出一期？我不知道，有内容的话，有时间的话，就会做。快得话一个月一次，慢的话或许几个月去了，说不定看了这期，下期就没了。

对。这个杂志，就是这么佛系。

对了，总编叫做reco，找好文章编辑下，也自己写，也翻译，手下一个兵丁也是没有的。有兴趣的话，发一个简历，我看得上你的话，可以一起工作。钱是没有的，一分也没有。

# Node.js 介绍

Ryan Dahl于2007年创造了Nodejs。现象表明，Nodejs非常火热。事实证明，Nodejs的成功，因为Ryan做了两个明智的选择：异步IO范式和执行引擎V8。

在Nodejs之前，主流的Web编程模式是同步IO，就是说当应用代码访问IO等消耗时间的代码时，线程会一直等待直到完成，在此期间，此线程无法做其他的工作。PHP，JSP等都是一样的做法。虽然，在等待期间，应用代码理应可以做其他的事儿，但是很少有人这样做。

Ryan的工作是编写写高性能Web服务，这个领域内，异步IO、事件驱动是常常被采用的方法，他评估了很多种高级语言，发现很多语言虽然同时提供了同步IO和异步IO，但是开发人员一旦用了同步IO，他们就再也懒得写异步IO了，所以，最终，Ryan瞄向了JavaScript。因为JavaScript是单线程执行，根本不能进行同步IO操作，所以，JavaScript的这一“缺陷”导致了它只能使用异步IO。

Ryan直接采用了Gooogle V8，号称最快的运行时引擎。这改变了人们对JavaScript运行缓慢的看法，于是，当发现Node把JavaScript带入到后端服务器开发时，大量的存量JavaScript开发人员，终于找到了在后端变现自己的知识的最佳通道。所以Node一下子就火了起来。

在接受Mappingthejourney访谈是，Ryan也提到了一个相对比较小的，但是也比较实际的要素：“老实说，构建一个Web服务器并不是最简单的事情，我认为很多这些系统都留给他们的社区来做，但是，事实上是，没有人这样做”，而Ryan做了。

当然，语言本身不算太差，甚至还有些亮点的。JavaScript语言本身是完善的函数式语言，还有表达能力很强但是非常简洁的字面量对象和JSON。曾经JavaScript被认为就是个“玩具”，但是，在Gmail,Google Map这样的优秀的应用的激发下，人们发现，通过模块化的JavaScript代码，加上函数式编程，直接使用最新的ECMAScript标准，其实是可以完全满足工程上的需求的。


