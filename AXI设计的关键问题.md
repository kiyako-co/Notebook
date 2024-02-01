# **AXI设计的关键问题**

## 1、设计AXI接口IP的考虑

### 1.1、AXI Feature回顾

**首先我们看一下针对AXI接口的IP设计**，在介绍之前我们先回顾一下**AXI所具有的一些feature**。![img](D:\lqh\Typora\图片\v2-7e415d6dfd279b65a01c5a2be4d7f39d_720w.webp)

**首先是Cache相关的问题**，Cache是提高系统Performance的一种很好的方式，AXI通过**AxCache[3:0]**信号来决定是否可以访问Cache，是否可以分配Cache。

**然后是原子操作**(Atomic access)，原子操作我专门花了一篇文章进行讲解，大家使用的时候尽量只使用Exclusive Access，Locked Access会阻塞总线，导致带宽无法充分利用，对系统性能影响较大。其实诸如此类的设计，对性能及吞吐量敏感的地方都应该避免使用，我们称为**阻塞性xxx**。为了支持Exclusive Access，我们自然需要Exclusive Monitor，来满足协议的要求，如何进行设计之前的文章也有提及，不再细说。

**<u>然后我们看一下AXI是如何提高性能的</u>**，这也是面试比较喜欢问的。如上面的图所示，主要有三种方式，我称之为提高AXI性能的三板斧，排名分先后。
**Outstanding**对性能影响最显著，尤其是主从之间通讯需要多拍的时候，它充分利用了这段时间发起别的传输。其实就是通过这些操作去掩盖原本的访存延时。
**然后是乱序（Out of Order）**，这个提高性能主要体现在Slave无法及时响应，就可以让后发起的请求先完成。就比如你去超市买菜，你前面的人刚好手机没电了，你是等他充好电还是先让你付款？
最后是**Interleave**，即交织读写，这个其实就是在空闲的Cycle中（气泡Cycle）插入读写事务，其实有点像乱序，可以理解成更加细粒度的乱序。当然粒度这么细，导致设计很复杂，所以在**AXI4中写交织就被移除了**。

![img](D:\lqh\Typora\图片\v2-122308ee38fe21410c8ece069333f456_720w.webp)

**最后是Memory Type的不同**，基于Memory类型的不同，可以决定你的读写是否可以Bufferable或者Cacheable。比如某些外设寄存器，你写完是希望直接改变系统当前的运行状态的，希望即刻生效，这种当然既不可以Cacheable也不可以Bufferable。而有些外设寄存器，比如写Flash Controller的寄存器，你写完以后，稍微延迟一些Cycle或者有乱序是可以接受的，这种情况就可以Bufferable。![img](D:\lqh\Typora\图片\v2-9f5ed435f73df433a730d09ee8c72d30_720w.webp)

### 1.2、Ordering consideration

首先我们看一下Write Channel和Read Channel之间的一个Order，**AXI协议并没有对它们之间的Order有一个约束**，如果想要实现该机制，就应该在之前的transaction收到最后的response以后才允许发起新的transaction。

你可以通过配置某个寄存器，来开启该功能。这个设计起来不是很复杂。如果你使用的是ARM或者X86或者主流的商用CPU，你可以使用[**memory barrier**](#jump0)指令来确保这个顺序。![img](D:\lqh\Typora\图片\v2-ce6f7681e40ce6f9c42870583085d8ad_720w.webp)

而对于同一ID而言，即Master发出的同一AWID/ARID而言，相应的数据必须是顺序的，如果不是顺序的，怎么知道哪个数据对应于哪一个地址？![img](D:\lqh\Typora\图片\v2-653adf5be05ed7c706f77614414286ce_720w.webp)

**我们看一下Slave的实现**，Slave想实现Order Model，比Master要复杂的多。我们想象一下这样的一个例子，一个Master去访问一个Slave，而这个Slave有很多不同的Memory Type，比如既有访问较快的寄存器，也有相对慢一点的SRAM，甚至还有DDR/FLASH。比如你去写一个DDR Controller，你去写它的配置寄存器，很快，你去写DDR，其实也要走这个DDR Controller，这样一次访问就非常非常的慢。但是，由于是同一个Master的访问，其ID是相同的，相同ID要保序，那难不成每次写DDR，都要等着数据回来？显然非常的不合理，那么在其内部就可以做一个**转换表**，如下图所示，来的时候是同样的ID，但Slave可以将其转换成不同的ID，这样就可以Outstanding了，也可以乱序了，非常的Amazing啊。注意，Slave内部是可以Out of Order返回的，但你返回给Master还是得顺序的。需要一块额外的存储空间来缓存这些需要返回的数据及相关信息，当来了，你就返回给Master。![img](D:\lqh\Typora\图片\v2-51dd13975c9b70f7d9261ae57a3de497_720w.webp)

我们再看一下Master的实现，当Master有多个Engine，比如DMA。每个DMA发出的ID是不一样的，当发出多个Outstanding的请求，Slave也相应的根据请求把数据拿回来了。相应的准备若干个Queue，当ID为0的返回的时候送给Queue0，当ID为0的最后一笔返回的时候即RLAST拉高的时候，做相应的处理返回给发出请求的Engine。其实就是需要一定的缓冲机制，来避免接不到数据等情况。

- Master需要数据queue来支持out of order；
- Master需要buffer来支持outstading；

![img](D:\lqh\Typora\图片\v2-34a2ac771a3cb9d0e9de70512f61a16b_720w.webp)

### 1.3、Debugging

- 使用counter来监视发起了多少次传输事务（transaction），以Master那边为例，当发出一次outstading请求，相应的counter+1，当收到最后的response，counter-1。这样系统挂死以后，发现某个Master的monitor记录counter不为0，这样就可以快速定位错误；
  - axvalid&axready=1，cnt+1
  - 对于写，当bvalid&bready=1，cnt-1
  - 对于读，当rvalid&rready&rlast，cnt-1
  - 还可以记录transfer数量
- 还可以记录更多的信息，如axlen、axid、axsize等；
- 使用timeout机制；

## 2、Interconnect拓扑结构

假设基于AXI的Master和Slave已经设计好了，如何对它们进行连接呢？**当多个Master和Slave进行通信的时候，就需要Interconnect**。

Interconnect的拓扑结构多种多样，**基于AXI的Interconnect一般采用Crossbar，如ARM-NIC400**，这里我们也只讲这种方式。![img](D:\lqh\Typora\图片\v2-f1e0d814e7bc0da5367f4ec93c8787ef_720w.webp)

我们看一下基于Crossbar的方式，**首先是最简单的点对点方式**，这种情况比较少见，比较典型的如ACP接口，Master需要直接访问Slave的cache（此时CPU作为Slave，并且是唯一的Slave），这种情况就可以点对点，而不用走多对多的总线。<img src="D:\lqh\Typora\图片\v2-cefc673e1e897b3f10ea12079178863d_720w.webp" alt="img" style="zoom:80%;" />

然后是一对多的情况，如下图所示，这种情况也很常见，比如一个CPU要去访问多个Slave外设。<img src="D:\lqh\Typora\图片\v2-2201f94e52795e7df294dc007e49db49_720w.webp" alt="img" style="zoom:80%;" />

然后是多个Master访问多个Slave，这种情况就需要引入仲裁逻辑了，相对的设计也会更加复杂，此时多个Master共享Interconnect,相应的会影响到带宽。<img src="D:\lqh\Typora\图片\v2-60f071e16d7591491e3b6f54c861b8c2_720w.webp" alt="img" style="zoom:80%;" />

然后我们看一下下面的例子，左边是完全映射，即Master可以访问所有的Slave。右图则是部分映射，可以看到一个Master只能访问指定的Slave。部分映射可以简化逻辑设计，节约面积，让能跑的主频更高一点。实际上也不需要每个Master都能访问每个Slave。根据自己的需求来就行。![image-20240118155344116](D:\lqh\Typora\图片\image-20240118155344116.png)

接下来我们看一下如果想设计一个好的互连，需要满足以下的特点：

- 支持多个Master和多个Slave
- 扩展灵活，可复用性强
- 时序好，从Interconnect本身去解决timing violation的问题

## 3、仲裁和时序收敛问题

接下来我们看一下仲裁相关的问题。仲裁本身是个很复杂的问题，这里只简单介绍。当多个Master需要访问同一个Slave的时候，这个时候就需要选择其中的一个。这里讲两种仲裁机制，Least Recently Granted和Round Robin仲裁机制。

首先看一下**Least Recently Granted**仲裁：这个和Cache替换中的LRU差不多，我们一开始会分好组，组和组之间是有固定优先级区别的，高优先级的总是会获得仲裁。对于同一个组的而言，我们会让最近没有被授权的Master获得仲裁权，因此我们需要有一个寄存器去记录历史信息，也可以用状态机实现。

![img](D:\lqh\Typora\图片\v2-ecfbe801348cee5079e77d4bc5400aea_720w.webp)

然后是**Round Robin**仲裁机制，这种仲裁机制下所有的Master的优先级都是相同的，采用轮询的方式进行仲裁，相应的也要进行记录历史信息，以决定如何获得仲裁。如下图所示，我们甚至可以用移位寄存器实现，下图是带Weight的Round Robin，可以看到M1占据了2个SLOT。因此使用频率是其它Master的两倍。![img](D:\lqh\Typora\图片\v2-801becb8927f7252bf2e79bf8d20a48c_720w.webp)

我们再看一下**时序如何收敛**，因为AXI本身采用握手机制，因此实际上非常灵活，晚了一个周期早了一个周期都无所谓，只要满足基本的各通道间依赖关系即可。没有AHB和APB那样的硬性1T Cycle delay的要求。因此我们可以在critical path上插入寄存器，以优化时序。

一般是有三种方式，打断Valid，打断Ready或者都打断。在IC设计中，无论使用的是否是AXI总线，**只要用Valid和Ready握手机制来传递数据，都可以使用该方法让时序收敛**，不一定是局限于某个总线协议。大家完全可以将其用在模块内部和模块与模块间的流传输。![img](D:\lqh\Typora\图片\v2-6f54f200256230047f730de029c0981f_720w.webp)

**上面这种机制用在总线上，一般称之为Register Slice**。先说说Slice的作用，在SoC中，如果Master和Slave的距离比较远，那么它们之间的bus信号要满足timing就可能有点困难，比如AXI中的 ARADDR、ARVALID这些信号从Master出来，要去很远的Slave，那么中间就要加很多的buffer，这引入的buffer delay就可能导致我们希望的timing不满足。这个时候就需要插入slice，给每个控制信号在中间加一级寄存器，把较长的走线缩短。当然插入的slice依然要保持bus的协议标准。简单来说，一个slice就是下面中间的那个模块。![img](D:\lqh\Typora\图片\v2-4becb320526b208a411393c6494c7297_720w.webp)

我们看一下在Register插入的位置，可以如下图所示（其实有很多可以插入的位置）。一般AXI各个通道是分开的，**我们在哪个通道加入register slice，相应的就会晚了一拍，实际上晚了一拍完全不影响逻辑功能。**![img](D:\lqh\Typora\图片\v2-9e1df44266a85c0657cfd34e83abdee7_720w.webp)

![img](D:\lqh\Typora\图片\v2-6a2182c2bf1b9c490fe2cb81a2f981f6_r.jpg)

# <a id = "jump0">**Memory Barrier**</a>

## **问题产生**![img](D:\lqh\Typora\图片\v2-c6e00c86d1851e075bbcb32878e85e5c_720w.webp)

CPU 0读需要对某一个地址进行写操作，但是这个数据不在local cache中，所以CPU 0向Bus发送Invalidate信号，其他CPU收到Invalidate信号后，查询这个数据是否在自己的local cache中，如果有则发送Acknowledgement信号，并将该数据从local cache中清除，CPU 0收到Ackonwledgement信号后，将数据写入自己的local cache中，最后将新的值写入。

**问题：CPU 0 在等待其他CPU的ack消息时是停滞(stall)状态，大部分时间都是在等待消息，出现性能损失！**

------------------------

## Store buffer

![img](D:\lqh\Typora\图片\v2-2c7bf2cfd7b6c3eb541c4c122755da56_720w.webp)

stroe buffer允许CPU将写数据先放入buffer中，然后执行其他指令，等到其他CPU的ack信号到来，再从buffer中拿出数据写到local buffer中。

==虽然store buffer可以提高一定的性能，但是又会出现新的问题==：

```c
a = 0, b = 0;
a = 1;
b = a + 1;
assert(b == 2);
```

假设a变量在CPU 1 的cache line中，b变量在CPU 0中cache line中，则：

1. CPU 0 执行a=1操作
2. CPU 0的local cache中没有a，出现write miss，所以CPU 0发出read invalidate 
3.  CPU 0将a=1放入store buffer中
4. CPU 1收到invalidate信号，回应了read response和ack消息，把自己local cache中的a清除
5. CPU 0收到response消息得到的a=0 
6. CPU 0 从 cache line 中读取了 a 值为 0 。
7. CPU 0执行b=a+1操作，b被CPU 0独占，所以b=1直接被写入CPU 0的local cache
8. CPU将store buffer中的a写入local cache，a变成1
9. CPU 0执行assert(b==2)，程序报错。

也就是在store buffer的数据没有及时的写到cache line中，导致指令取到的是其他CPU的旧值，出现stotre buffer中数据和cachelineh中数据不一致。

使用**store forwarding**，当CPU进行读操作的时候，同时从store buffer和cache中读数据，如果store buffer中有数据，就直接用store buffer中的数据，这样就解决了store buffer和cache中数据不一致导致使用旧值的问题。

**但是store forwarding只能解决同一个CPU下的不一致问题，如果读操作和写操作发生在不同的CPU中，store forwarding就不起作用了。此时出现Memory odering问题。**

---------------------------

## **Memory ordering**

```c++
a = 0 , b = 0;
void fun1() {   
  a = 1;   
  b = 1;
}

void fun2() {  
  while (b == 0) continue;  
  assert(a == 1);
}
```

CPU 0执行fun1，CPU1执行fun2。a变量在CPU 1里，b变量在CPU 0里。

1. CPU 0执行a=1写操作，由于a不在local cache中，因此，CPU 0将a的值放入store buffer中之后，发送read invalidate消息到总线上
2. CPU 1执行while(b==0)循环，由于b不在CPU 1中，因此CPU 1发出read miss
3. CPU 0继续执行b = 1操作，由于b就在local cache中，因此CPU 0直接将b=1写入cacheline
4. CPU 0收到read miss消息，将b=1反馈给CPU 1，并将b的状态从exclusive变成shared
5. CPU 1收到CPU 0的response消息，将b=1写入cacheline(shared)
6. 此时b不在是0，CPU 1跳出循环
7. CPU 1执行assert(a == 1)，这时CPU 1的local cache中还是旧的a值，因此assert(a==1)失败
8. CPU 1收到CPU0的invalidate消息，发送responsed和ack，并清除自己的cacheline中a的数据
9. CPU 0收到CPU 1的response和ack消息，得到a=0，并将store buffer中a=1写入cacheline

**原因**：CPU 0对a的写操作还没有执行完，但是CPU 1对a的读操作已经执行完了。

毕竟CPU并不知道哪些变量有相关性，这些变量是如何相关的。不过CPU设计者可以间接提供一些工具让软件工程师来控制这些相关性。这些工具就是 memory barrier 指令。要想程序正常运行，必须增加一些 memory barrier 的操作。

-----------------

## **Store Memory Barrier**

```c
a = 0 , b = 0;
void fun1() {   
  a = 1;   
  smp_mb();   
  b = 1;
}

void fun2() {   
  while (b == 0) continue;
  assert(a == 1);
}
```

smp_mb() 这个内存屏障的操作会在执行后续的store操作之前，首先flush store buffer（也就是将之前的值写入到cacheline中）。smp_mb() 操作主要是为了让数据在local cache中的操作顺序是符合program order的顺序的，为了达到这个目标有两种方法：方法一就是让CPU stall，直到完成了清空了store buffer（也就是把store buffer中的数据写入cacheline了）。方法二是让CPU可以继续运行，不过需要在store buffer中做些文章，也就是要记录store buffer中数据的顺序，在将store buffer的数据更新到cacheline的操作中，严格按照顺序执行，即便是后来的store buffer数据对应的cacheline已经ready，也不能执行操作，要等前面的store buffer值写到cacheline之后才操作。增加smp_mb() 之后，操作顺序如下：

1. CPU 0执行a=1的赋值操作，由于a不在local cache中，因此，CPU 0将a值放 store buffer中之后，发送了read invalidate命令到总线上去。
2. CPU 1执行 while (b == 0) 循环，由于b不在CPU 1的cache中，因此，CPU发送一个read message到总线上，看看是否可以从其他cpu的local cache中或者memory中获取数据。
3. CPU 0执行smp_mb()函数，给目前store buffer中的所有项做一个标记（后面我们称之marked entries）。当然，针对我们这个例子，store buffer中只有一个marked entry就是“a=1”。
4. CPU 0继续执行b=1的赋值语句，虽然b就在自己的local cache中（cacheline处于modified状态或者exclusive状态），不过在store buffer中有marked entry，因此CPU0并没有直接操作将新的值1写入cache line，取而代之是b的新值”1“被写入store buffer，当然是unmarked状态。
5. CPU 0收到了read message，将b值”0“（新值”1“还在store buffer中）回送给CPU 1，同时将b cacheline的状态设定为shared。
6. CPU 1收到了来自CPU 0的read response消息，将b变量的值（”0“）写入自己的cacheline，状态修改为shared。
7. 完成了bus transaction之后，CPU 1可以load b到寄存器中了（local cacheline中已经有b值了），当然，这时候b仍然等于0，因此循环不断的loop。虽然b值在CPU 0上已经赋值等于1，但是那个新值被安全的隐藏在CPU 0的store buffer中。
8. CPU 1收到了来自CPU 0的read invalidate消息，以a变量的值进行回应，同时清空自己的cacheline。
9. CPU 0将store buffer中的a值写入cacheline，并且将cacheline状态修改为modified状态。
10. 由于store buffer只有一项marked entry（对应a=1），因此，完成step 9之后，store buffer的b也可以进入cacheline了。不过需要注意的是，当前b对应的cacheline的状态是shared。
11. CPU 0发送invalidate消息，请求b数据的独占权。
12. CPU 1收到invalidate消息，清空自己的b cacheline，并回送acknowledgement给CPU 0。
13. CPU 1继续执行while (b == 0)，由于b不在自己的local cache中，因此 CPU 1发送read消息，请求获取b的数据。
14. CPU 0收到acknowledgement消息，将b对应的cacheline修改成exclusive状态，这时候，CPU 0终于可以将b的新值1写入cacheline。
15. CPU 0收到read消息，将b的新值1回送给CPU 1，同时将其local cache中b对应的cacheline状态修改为shared。
16. CPU 1获取来自CPU 0的b的新值，将其放入cacheline中。
17. 由于b值等于1了，因此CPU 1跳出while (b == 0)的循环，继续执行。
18. CPU 1执行assert(a == 1)，不过这时候a值没有在自己的cacheline中，因此需要通过cache一致性协议从CPU 0那里获得，这时候获取的是a的最新值，也就是1值，因此assert成功。

**上述这个例子展示了 write memory barrier , 简单来说在屏障之后的写操作必须等待屏障之前的写操作完成（写数据从store buffer写到cacheline）才可以执行，读操作则不受该屏障的影响。**

-------------------------

## **Store Sequences Result in Unnecessary Stalls**(**Invalidate Queue**）

**新的问题：每个CPU的的store buffer不会实现的太大，其entry的数目也就不会太多。当CPU以中等的频率执行store操作的时候(假设所有的store操作都导致了cache miss)，store buffer会很快的被填满。在这种状况下，CPU只能又进入等待状态，直到cache line完成invalidate和ack的交互后，可以将store buffer中的entry写入cache line，从而为新的store让出空间之后，CPU才继续执行。**

这种情况也可能发生在调用memory barrier指令之后，因为一旦store buffer中的某个entry被标记了，那么随后的store即使没有发生cache miss也必须进入store buffer(此时这些store不属于标记状态，因为没有miss)，等待invalidate指令完成。

为了解决这个问题引入了 **invalidate queues** 可以缓解这个状况。store buffer之所以很容易被填充满，主要是其他CPU回应invalidate acknowledge比较慢，如果能够加快这个过程，让store buffer尽快进入cacheline，那么也就不会那么容易填满了。

invalidate acknowledge不能尽快回复的主要原因是invalidate cacheline的操作没有那么快完成，特别是cache比较繁忙的时候，这时，CPU往往进行密集的loading和storing的操作，而来自其他CPU的，对本CPU local cacheline的操作需要和本CPU的密集的cache操作进行竞争，只有完成了invalidate操作之后，本CPU才会发生invalidate acknowledge。此外，如果短时间内收到大量的invalidate消息，CPU有可能跟不上处理，从而导致其他CPU不断的等待。

然而，CPU其实不需要完成invalidate操作就可以回送acknowledge消息，这样，就不会阻止发生invalidate请求的那个CPU进入无聊的等待状态。CPU可以buffer这些invalidate message（放入Invalidate Queues），然后直接回应acknowledge，表示自己已经收到请求，随后会慢慢处理。当然，再慢也要有一个度，例如对a变量cacheline的invalidate处理必须在该CPU发送任何关于a变量对应cacheline的操作到bus之前完成。![img](D:\lqh\Typora\图片\v2-aad03e8cfcfe520fcc4a8a2c9d0597fb_720w.webp)

有了Invalidate Queue的CPU，在收到invalidate消息的时候首先把它放入Invalidate Queue，同时立刻回送acknowledge 消息，无需等到该cacheline被真正invalidate之后再回应。当然，如果本CPU想要针对某个cacheline向总线发送invalidate消息的时候，那么CPU必须首先去Invalidate Queue中看看是否有相关的cacheline，如果有，那么不能立刻发送，需要等到Invalidate Queue中的cacheline被处理完之后再发送。一旦将一个invalidate（例如针对变量a的cacheline）消息放入CPU的Invalidate Queue，实际上该CPU就等于作出这样的承诺：在处理完该invalidate消息之前，不会发送任何相关（即针对变量a的cacheline）的MESI协议消息。

---------------------

## **Load Memory Barrier**

```c
a = 0 , b = 0;
void fun1() {    
  a = 1;    
  smp_mb();    
  b = 1;
}

void fun2() {    
   while (b == 0) continue;
   assert(a == 1);
}
```

假设 a 存在于 CPU 0 和 CPU 1 的 local cache 中，b 存在于 CPU 0 中。CPU 0 执行 fun1() , CPU 1 执行 fun2() 。操作序列如下：

1. CPU 0执行a=1操作，由于a本来是shared状态，此时CPU 0对a进行独占写操作，需要先发送invalidate消息让CPU 1清local cache中的a，同时CPU 0 要将a=1存到store buffer。
2. CPU 1执行while (b==0)，由于b不在CPU 1的cacheline中，于是发送read miss给CPU 0。
3. CPU 1收到CPU 0的invalidate消息，放入invalidate queue，并立即返回ack
4. CPU 0收到CPU 1的invalidate ack后，即可以越过程序设定的内存屏障(smp_mb)，这样a的新值从store buffer进入cacheline，状态变成Modufied。
5.  CPU 0 越过memory barrier后继续执行b=1的赋值操作，由于b值在CPU 0的local cache中，因此store操作完成并进入cache line。
6. CPU 0收到read消息后将b的最新值1回送给CPU 1，并修改a的cache line为shared。
7. CPU 1收到read response，将b=1加载到cacheline。
8. CPU 1跳出while(b==0)循环，继续执行后续代码
9. CPU 1执行assert(a==1)，由于CPU 1没有完成invalidate的处理，a仍然是旧值0，因此assert失败
10.  Invalidate Queue中针对a cacheline的invalidate消息最终会被CPU 1执行，将a设定为无效。

很明显，在上面场景中，加速 ack 导致fun1()中的memory barrier失效了，因此，这时候对 ack 已经没有意义了，毕竟程序逻辑都错了。怎么办？其实我们可以让memory barrier指令和Invalidate Queue进行交互来保证确定的memory order。具体做法是这样的：当CPU执行memory barrier指令的时候，对当前Invalidate Queue中的所有的entry进行标注，这些被标注的项次被称为marked entries，而随后CPU执行的任何的load操作都需要等到Invalidate Queue中所有marked entries完成对cacheline的操作之后才能进行。因此，要想保证程序逻辑正确，我们需要给 fun2() 增加内存屏障的操作，具体如下：

```c
a = 0 , b = 0;
void fun1() {    
  a = 1;    
  smp_mb();    
  b = 1;
}

void fun2() {    
  while (b == 0) continue;    
  smp_rmb();
  assert(a == 1);
 }
```

**当 CPU 1 执行完 while(b == 0) continue; 之后， 它必须等待 Invalidate Queues 中的 Invalidate 变量 a 的消息被处理完，将 a 从 CPU 1 local cache 中清除掉。然后才能执行 assert(a == 1)。CPU 1 在读取 a 时发生 cache miss ，然后发送一个 read 消息读取 a ，CPU 0 会回应一个 read response 将 a 的值发送给 CPU 1。**

---------------

许多CPU architecture提供了弱一点的memory barrier指令只mark其中之一。如果只mark invalidate queue，那么这种memory barrier被称为read memory barrier。相应的，write memory barrier只mark store buffer。一个全功能的memory barrier会同时mark store buffer和invalidate queue。

我们一起来看看读写内存屏障的执行效果：对于read memory barrier指令，它只是约束执行CPU上的load操作的顺序，具体的效果就是CPU一定是完成read memory barrier之前的load操作之后，才开始执行read memory barrier之后的load操作。read memory barrier指令象一道栅栏，严格区分了之前和之后的load操作。同样的，write memory barrier指令，它只是约束执行CPU上的store操作的顺序，具体的效果就是CPU一定是完成write memory barrier之前的store操作之后，才开始执行write memory barrier之后的store操作。全功能的memory barrier会同时约束load和store操作，当然只是对执行memory barrier的CPU有效。
