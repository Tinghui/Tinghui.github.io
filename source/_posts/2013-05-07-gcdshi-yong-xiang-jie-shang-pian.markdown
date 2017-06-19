---
layout: post
title: "GCD使用详解 上篇"
date: 2013-05-07 14:46
comments: true
published: true
categories: [Objc, GCD, 翻译]
---

## Dispatch Queues

Dispatch Queue，顾名思义，它是一个队列，用于存储要执行的任务。程序员可以用block语法编写要执行的任务，再通过`dispatch_async`函数将它加入到一个dispatch队列中。然后dispatch队列会按照FIFO的顺序执行这些任务。如图7-1所示。

![](/images/posts/GCD7-1_execution_on_a_dispatch_queue.png)

GCD中有2种dispatch队列。一种是串行队列，队列中前一个任务执行完毕后，后一个任务才开始执行。另一种是并行队列，并行队列可以同时执行多个任务。如图7-2所示。

<!--more-->

![](/images/posts/GCD7-2_Serial_Dispatch_Queue_and_Concurrent_Dispatch_Queue.png)

#### 两种类型的Dispatch队列
下面的代码用`dispatch_async`函数将任务添加到dispatch队列中。以这段代码为例，我们来对比一下这两种类型的队列有什么不同。

```objc
dispatch_async(queue, blk0);
dispatch_async(queue, blk1);
dispatch_async(queue, blk2);
dispatch_async(queue, blk3);
dispatch_async(queue, blk4);
dispatch_async(queue, blk5);
dispatch_async(queue, blk6);
dispatch_async(queue, blk7);
```

我们先来看看，当队列是串行队列时，这些任务是如何执行的呢。

#### 串行队列

首先blk0开始执行。blk0完成后，blk1开始执行。然后blk1完成后，blk2开始执行，如此顺序执行，一次只执行一个任务。因此，每次都会按照下面这种顺序执行：

```objc
blk0
blk1
blk2
blk3
blk4
blk5
blk6
blk7
```

接下来，我们再来看看并行队列是如何执行这些任务的。

#### 并行队列
首先，blk0开始执行，不管它是否已经完成，blk1也会开始执行，然后不管blk1是否已经完成，blk2也会开始执行，以类似的方式执行下去，同时会有多个任务在执行。同时能够执行的任务的个数取决于当前的系统状态。iOS或者OS X系统会根据当前的系统状态（例如队列中有多少个任务，CPU的核心数目或者CPU的使用情况）决定可以同时执行多少个任务。如图7-3所示，任务可以在多个线程上并发执行。

![](/images/posts/GCD7–3_Relationship_of_Serial_Dispatch_Queue_Concurrent_Dispatch_Queue_and_threads.png)

XNU内核(iOS和OS X的核心部件)会决定线程的个数，也负责创建用于执行任务的线程。当一个任务完成后，处于运行状态的任务的数目就会减少，XNU内核会终止不再需要的的线程。通过并行队列，XNU内核管理多线程，完美的并发运行各个任务。表格7-1，显示了上面的代码在多线程上的执行的情况。

![](/images/posts/Table7–1_Theexampleresultwithaconcurrentdispatchqueue.png)

这里，我们假设并行队列有4个线程。首先线程0开始执行blk0。接着，线程1开始执行blk1，线程2开始执行blk2，线程3开始执行blk3。然后，线程0上blk0执行完毕后，blk4开始执行。接着，线程2开始执行blk5，因为线程2上的blk2已经执行完毕了，但是线程1上的blk1仍然在运行。

这种方式下，并行队列中任务的执行顺序取决于任务本身的状态和系统状态等因素。执行顺序并不会像串行队列那样总是固定不变的。如果任务需要按照特定的顺序执行或者不想让任务并发执行，那应当使用串行队列。

那么我们如何获得这些队列呢？

## 获取Dispatch队列

有2种途径获得dispatch队列。一种是通过`dispatch_queue_create`函数创建一个队列；另一种是直接获取现成的主线程队列或者全局队列。

#### dispatch_queue_create

`dispatch_queue_create`这个函数用于创建一个dispatch队列。你可以用这个函数创建一个新的队列。下面的代码演示了如何创建一个串行队列。当然，也可以用它创建一个并行队列，稍后会作讲解。

```objc
dispatch_queue_t mySerialDispatchQueue =
	dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);
```

创建一个串行队列后，它和其他串行队列是相互独立的，虽然它们都是一次只执行一个任务。例如，四个串行队列，可能同时开始执行任务，但它们是相互独立的。如图7-4所示。

![](/images/posts/Figure7–4MultipleSerialDispatchQueues.png)

还有要特别注意的一点。当一个串行队列被创建，并向其中加入一个任务后，系统会为队列创建一个线程。每一个串行队列都有一个线程。如果创建了2000个串行队列，就会有2000个线程。这样，大量的线程会耗费大量的内存，也会因为需要大量切换上下文环境而拖慢系统速度。

![](/images/posts/Figure7–5_Problem_of_multiple_Serial_Dispatch_Queues.png)

因此，你应该只在需要防止数据冲突的情况下才使用串行队列。因为，多线程更新数据时会有资源竞争(race condition)问题，这也是多线程编程中的一个常见问题。

![](/images/posts/Figure_7–6_Situation_where_a_Serial_Dispatch_Queue_should_be_used.png)

串行队列应该是需要多少个才创建多少个。例如，当更新数据库的时候，应该为每一个表创建一个串行队列。当更新文件的时候，应该为文件创建一个串行队列或者为每个独立文件块创建一个串行队列。即使你可能会认为可以依靠串行队列来创建比并行队列更多的线程，也不要创建过多的串行队列。如果要执行的任务不会造成像数据冲突这种问题并且你想要并发的执行它们，你应该使用并行队列。即使不断的创建并发队列，也不会有什么问题，因为并发队列只会使用由XNU内核管理的线程。

回到`dispatch_queue_create`函数上来。函数的第一个参数是队列的名字，建议这个参数使用像示例代码中那样的反向全称域名。当你用Xcode或Instruments调试的时候，队列的名称会显示成这个参数的值。同样CrashLog中也会显示这个队列名字。因此，应该用一个开发人员所能够理解的名字来命名它。如果不想给它命名，直接设置成NULL也可以，但这样不利于调试。

第二个参数，如果要创建一个串行队列，就将其设置成NULL。如果要创建一个并行队列，就设置成DISPATCH_QUEUE_CONCURRENT，如下所示。

```objc
dispatch_queue_t myConcurrentDispatchQueue =
	dispatch_queue_create( "com.example.gcd.MyConcurrentDispatchQueue", DISPATCH_QUEUE_CONCURRENT);
```

`dispatch_queue_create`函数的返回值是一个`dispatch_queue_t`类型的值。前面所有的例子中，队列都是`dispatch_queue_t`类型。

```objc
dispatch_queue_t myConcurrentDispatchQueue =
	dispatch_queue_create( "com.example.gcd.MyConcurrentDispatchQueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(myConcurrentDispatchQueue, ^{NSLog(@"block on myConcurrentDispatchQueue");});
```

这个例子中，block会运行在并行队列上。

虽然编译器有一个很强大的自动内存管理机制(ARC)，但是程序员还是要自己负责释放自己创建的dispatch队列。因为，与block不同的是，dispatch队列不会被当成一个Objective-C对象。当你不再需要它们的时候，你需要调用`dispatch_release`函数来释放它们。（**译注: 注意iOS6.0以后，ARC已经可以自动管理dispatch对象的释放了。**）

```objc
dispatch_release(mySerialDispatchQueue);
```

同样，也有retain的函数：

```objc
dispatch_retain(myConcurrentDispatchQueue);
```

也就是说，在Objective-C中，要用引用计数技术来管理dispatch队列的内存。比如，前面例子中创建的并发队列"myConcurrentDispatchQueue"也需要释放掉。

```objc
dispatch_queue_t myConcurrentDispatchQueue =
	dispatch_queue_create( "com.example.gcd.MyConcurrentDispatchQueue", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(myConcurrentDispatchQueue, ^{NSLog(@"block on myConcurrentDispatchQueue");});
dispatch_release(myConcurrentDispatchQueue);
```

一个并行队列会用多个线程来执行任务。在这个例子中，把任务加入队列后，队列就被释放了。那么这样会不会有问题？

没问题的。当一个block被加入到队列后，block会用`dispatch_retain`函数retain队列，这样block就拥有了队列的所有权。不管是串行队列还是并行队列，都会这样。然后当block执行完毕后，block会通过`dispatch_release`函数释放掉对队列的所有权。

即使刚刚将block加入到队列后就将队列释放掉了，block稍后也会被执行。block执行完毕后，会释放掉队列，block也会被丢弃掉。`dispatch_retain`函数和`dispatch_release`函数不仅仅只用来管理队列。接下来我们会见到许多带有create单词的GCD函数，所有这种函数返回的对象都需要用`dispatch_release`函数来释放。如果你通过其他函数获得了某个对象，根据需要也可以用`dispatch_retain`来获得对象的所有权，然后不需要了就用`dispatch_release`来释放掉所有权。

#### 主线程队列和全局队列

另外一种获得dispatch队列的途径是直接获取系统已经创建好的队列。系统有两种不需要你创建的队列：主线程队列和全局队列。

主线程队列中的所有任务都在主线程上执行。因为只有一个主线程，因此主线程队列是一个串行队列。如图7-7所示，主线程队列中的任务会在主线程上的RunLoop中执行。因为是执行在主线程上，因此你应当只用它来执行必须在主线程中执行的操作，比如更新UI的操作。它和NSObject的`performSelectorOnMainThread`方法很像。

![](/images/posts/Figure7–7Main-Dispatch-Queue.png)

系统还提供了另外一种被称为全局队列(global dispatch queues)的队列。这种队列是并行队列，可以在应用程序的任何地方使用它们。大多数情况下，如果没有特殊原因(稍后会提到一些特殊情况)，你都不需要用`dispatch_create`函数来创建一个新的并行队列。你可以直接获取系统提供的全局队列，直接使用。全局队列有4种不同的优先级：high、default、low、background。XNU内核会管理全局队列用到的线程以及它们的优先级。在添加任务到一个全局队列的时候，要选择一个优先级适合的队列。XNU并不能保证线程的实时性，优先级只是用作一个参考。比如，如果你不太在乎一个任务是否会被执行，就应该用background这个优先级。表7-2中列出了系统提供的几种队列。

![](/images/posts/Table7–2Types-of-dispatch-queues.png)


以下代码演示了如何获取这些队列。

```objc
// 获取主线程队列
dispatch_queue_t mainDispatchQueue = dispatch_get_main_queue();

// 获取high优先级的全局队列
dispatch_queue_t globalDispatchQueueHigh = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);

// 获取default优先级的全局队列
dispatch_queue_t globalDispatchQueueDefault = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

// 获取low优先级的全局队列
dispatch_queue_t globalDispatchQueueLow = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);

// 获取background优先级的全局队列
dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
```

顺便提一下，如果你在主线程队列或者全局队列上调用`dispatch_retain`函数或者`dispatch_release`函数，是没有任何作用的，也不会引起任何问题。这也是为什么直接使用全局队列会比创建一个新队列更方便的原因。当然，也取决于你的代码，如果你觉得把主线程队列或者全局队列看待成是由`dispatch_queue_create`函数创建的更方便的话，也可以依照引用计数的规则用`dispatch_retain`和`dispatch_release`函数来管理它们。

最后，让我们来看一个如何使用主线程队列和全局队列的例子：

```objc
/*
* 在优先级为default的全局dispatch队列上执行一个block
*/
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	/*
	* 这里，是一些需要并发的执行的任务
	*/

	/*
	* 然后，在主线程队列上执行了一个block
	*/
	dispatch_async(dispatch_get_main_queue(), ^{

	/*
	* 这里，是一些会在主线程上面执行的操作.
	*/
	});
});
```

现在，我们已经学习了GCD的基础知识。接下来，我会介绍一些API，通过这些API我们可以更好的管理GCD队列，使它们更高效。

译自:  《[Pro Multithreading and Memory Management for iOS and OS X with ARC, Grand Central Dispatch, and Blocks"](http://www.amazon.com/Pro-Multithreading-Memory-Management-iOS/dp/1430241160)》 第7章

<p style="text-align:center"><img src="/images/posts/thx_money.png" width="30%" height="30%" /></p>