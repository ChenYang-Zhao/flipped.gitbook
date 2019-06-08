## Stack上的内存分配

在CloudFare，我们大量的服务和应用使用Go语言编写，这篇blog中，我们深入的分析一下golang中的一些技术难点；

Goroutine是golang中最重要的特性之一，它们是廉价的，协同调度的线程，可以用在许多场景：比如超时机制等。为了使goroutine适用于尽可能多的场景，每个goroutine使用的内存应该尽可能的少，而且启动goroutine的配置应该尽可能的简单；

为了实现这一目的，golang使用了和其它语言一样的方式来管理stack，不过其实现方式却很不一样。

#### 线程stack简介

在介绍Go的stack管理之前，我们来看一下C语言是如何管理stack的。

当使用C语言开启一个线程时，C的标准库负责分配一块内存作为该thread的stack。流程是分配一块内存，然后告诉kernel这块内存的位置，然后由kernel负责thread的执行；当这块内存空间不够时，问题就来了；看下面的例子：

```C
int a(int m, int n) {
	if (m == 0) {
		return n + 1;
	} else if (m > 0 && n == 0) {
		return a(m - 1, 1);
	} else {
		return a(m - 1, a(m, n - 1));
	}
}

```

这段代码实现了了一个很深的递归，它很快可以占满栈内存的空间，要解决这个问题，有两种办法：第一是增加每个thread中stack空间的大小，但是，这样意味着每个threa的stack占用的内存空间都变大了，尽管一些thread不需要你那么大空间的stack；另一个办法是对每个thread定义不同的栈大小，但这样我们必须自己配置每个thread使用的栈大小，这是一项繁重且困难的工作；

#### Golang是如何解决这个问题的

Go不是给每个goroutine固定打下的stack空间，而是按goroutine栈内存的真实需求，分配给它们真正使用的栈大小，这样，programmer就无需考虑栈空间大小的问题，Go团队正在对这部分的实现做出改变，下面，我们介绍原来的处理方式及存在的不足，然后介绍最新的处理方式；

#### 分段堆栈（Segmented stacks）

golang原本实现stack内存分配的方式是分段堆栈，每当创建一个goroutine时，Go分配一个8KB的栈空间给它；当goroutine运行中stack分配的内存超过8KB时，Golang执行morestack申请更多的栈内存空间；morestack方法为stack申请了一块新的栈内存，然后再这个新的栈的底部生成一个struct数据结构，并向其中写入一些数据，其中包括之前stack的地址信息；获得了新的堆栈段（stack segment）之后，golang重启goroutine，并重新执行导致栈空间不足的那个function，这被称作stack split；

栈分裂以后的示意图如下：

```Shell
 +---------------+
  |               |
  |   unused      |
  |   stack       |
  |   space       |
  +---------------+
  |    Foobar     |
  |               |
  +---------------+
  |               |
  |  lessstack    |
  +---------------+
  | Stack info    |
  |               |-----+
  +---------------+     |
                        |
                        |
  +---------------+     |
  |    Foobar     |     |
  |               | <---+
  +---------------+
  | rest of stack |
  |               |
```

在新stack的底部，放入了一个方法名为lessstack的方法的栈帧，实际上，这个方法并没有被调用，使用它的目的是当之前导致stack分裂的方法退出时，进入lessstack的栈帧，它扫描被放入栈底的struct的数据，找到栈分裂之前的栈的地址，这样就可以回到上一个function的调用了；之后，新的stack段就可以被回收了；

#### 分段堆栈存在的问题

分段堆栈提供了可按需增长stack内存空间的功能，让开发者无需操心stack内存空间的管理，开启goroutine变得很廉价，而且开发者无需知道stack占用了多少空间。

但是这个方法有一些缺点，缩小栈空间的代价很大，尤其当在循环中不停的分配stack段和释放stack段时，付出的代价是很大的；这被称为 stack split problem；也是Go的开发者将stack内存管理切换到stack copying方法的原因；

#### 栈复制（Stack Copying）

和分段stack不同的是，栈复制方法并不是在新的栈中使用指针指向原来的栈，而是在stack空间不够是，分配一个两倍于当前栈大小的新栈，然后将原来stack的内容复制到新的stack中。这意味着，当栈的使用空间缩小到和原来一样的大小时，golang的运行时无需做任何工作；同样，在栈使用空间再变大时，使用之前释放的空间就可以了，无需做额外的工作。


#### 栈是如何被复制的

由于可能存在指针指向栈内的变量，当栈的内存地址变化时，这些指针的地址必须被重新计算，否则原来的内存地址就失效了。幸运的是，指向栈上变量的指针，只可能存在于当前的栈中；

由于在GC时需要知道指针具体的指向的内存地址，也知道stack中的变量是否是指针，这样，在移动栈时，修改其中指针的值，指向新stack中对应的变量地址就可以了。

由于栈复制时必须用到gc的信息，但是golang的runtime中很多是用C语言实现的，许多对rutime内方法的调用没有指针信息，所以就无法完成栈的复制；此时，goroutine的stack依然使用老的分段堆栈实现；




### 参考

- [How Stacks are Handled in Go](https://blog.cloudflare.com/how-stacks-are-handled-in-go/)
- [Contiguous stacks](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)