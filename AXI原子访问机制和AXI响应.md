# AXI原子访问机制和AXI响应

## 1、Atomic访问机制

### 1.1、Atomic信号

众所周知，操作系统的很多机制需要底层硬件的支持，如并发、虚拟化等。随着多处理器的流行，ARM也自然而然的要在其总线上加入了和并发相关的信号，以满足对并发、并行等机制的支持。

首先我们看一下AXI中和原子访问相关的信号，AXI协议中使用的是**AxLOCK**信号来实现原子相关的操作，考虑这样的一个场景，先让处理器1读某个地址的数据做相应的运算，再让处理器2写同样的地址，我们当然不希望一半的时间是处理器1的访问，一半的时间是处理器2的访问，这样处理器1可能拿到处理器2更新以后的值，也有可能拿到更新以前的值，最后的结果变成了**Non-deterministic**，会带来一系列的问题，最好的方法就是处理器1读的时候，你处理器2就不要去写，读完了你再去操作，或者就用一个监控机制，来确保处理器2的操作不会对处理器1的运算结果产生影响。

AXI3和AXI4的原子操作不太一样，如下图所示，下面分别对二者进行介绍。

![img](D:\lqh\Typora\图片\v2-0ba883dfdca30549c2521ea5dfe52384_720w.webp)

对于**AXI3**而言，将原子操作分成了**Exclusive**和**Locked**两种。

当为'b10的时候，即**Locked访问**的时候。如果Master 1正在访问某个Slave，此时其它的Master也想访问该Slave的时候，那对不起，已经锁住了，不允许发起访问。相应的Interconnect也必须支持这一机制，仲裁器此时不允许别的Master访问相应的Slave。（类似于AHB的HMASTERLOCK）

当为'b01的时候，即**Exclusive访问**的时候，这种时候不会将整个访问路径都给阻塞住，而是用一个flag标志，比如当Master 1访问该Slave的时候，会有个**Monitor**记录下这个地址，当其它的Master要访问这个地址的时候，就知道不允许访问，它可能会一直的loop，也有可能进入IDLE模式，当Master 1访问结束以后，通过中断唤醒它，再去进行相应的访问，这种模式就不会将整个总线阻塞住。

听上去Exclusive对总线性能影响更小，正是因为如此，**AXI4只有Exclusive的方式，舍弃掉了Locked Access的模式。**

上面的描述可能还不够Make sense，下面通过具体的实例给大家进一步解释。

### 1.2、Atomic Access

**首先我们看一下Locked access**，这种访问机制只在**AXI3**中使用，如下图所示，如果Master0正在用locked access访问Slave的话，其它所有的Master访问Slave的路径都会被阻塞住，图中只给了两个Master的例子，更多的Master也是一样的。这种机制，很自然的可以知道，**非常慢！它对总线性能影响是很大的**。因为其它Master可以访问的是这个Slave的别的Region，根本不会产生竞争问题，但也给阻塞住了。设计的不好甚至会导致一个Master独占某个Slave，其它的Master的访问都被hang住了。

对于ARM处理器而言、**可以通过SWP指令**（用在早期的Legacy Arm Processors，比如ARM7，这种指令在新的ARM处理器中已经不使用了，应该是在ARMv6以后就移除了）发出这种访存操作。当然也可以自己设计Master，自定义某种情况，用这种Locked Access。实际上已经不怎么使用了，因为对总线性能影响太大。![img](D:\lqh\Typora\图片\v2-e6df376f5d2a03cea61406843445f31c_720w.webp)

接下来我们看一下**Exclusive access**整个流程是怎么样的：

- Master对某个地址发起exclusive read；
- 过了一会，Master尝试对同样的地址进行exclusive write来完成整个exclusive operation；
- 如果**<u>在读写之间没有别的Master</u>**往这个地址写数据，则上述的exclusive write成功，否则失败。

-----------------------------------------**NOTE**-------------------------------------

重点针对写操作！也就是一个master进行独占操作的时候，会标记这个master，然后如果其他的master对这个地址进行写操作，那么标记会解除，后面这个master要对这个这个地址进行操作的时候，因为标记解除了，所有无法成功写入，除非其他master对这个地址的标记解除。

---------------------------

整个流程和RISC-V的**LR/SC指令**也非常的像，可以说是非常的amazing。这个特点是我在一篇论文中看到的，论文题目为**ATUNs: Modular and Scalable Support for Atomic Operations in a Shared Memory Multiprocessor**，想学习具体微架构设计的可以看看这篇论文。![img](D:\lqh\Typora\图片\v2-1f02877f7257ac8a2dcf28c388b72d3d_720w.webp)

此外上面的机制肯定是需要额外的逻辑进行支持的，一般是在Interconnect或者Slave上增加**Monitor**来实现，接下来我们看一下具体例子是怎么实现的。（以下所有的访问都是Exclusive访问）

首先看第一个例子：

- Master0首先读0xA000这个地址，没有问题，Monitor会记录下这个地址以及对应的Master ID号。返回ExOKAY，读回来的值为0x1；
- Master1读0xB000这个地址，Monitor同样记录下这个地址以及对应的Master ID，返回ExOKAY，读回来的值为0x2；
- Master0去写0xA000这个地址，这个地址之前已经读过了返回ExOKAY，并且中间没有别的Master写过，所以没问题，写下0x3，返回ExOKAY；
- Master1去写0xB000这个地址，这个地址之前已经读过了返回ExOKAY，并且中间没有别的Master写过，所以没问题，写下0x4，返回ExOKAY；

![img](D:\lqh\Typora\图片\v2-dbee69f22db6df610ad0bc92741c2a9f_720w.webp)

我们看一下第二个例子：

- Master0首先读0xA000这个地址，没有问题，Monitor会记录下这个地址以及对应的Master ID号。返回ExOKAY，读回来的值为0x1；
- Master1读0xA000这个地址，Monitor同样记录下这个地址以及对应的Master ID，返回ExOKAY，读回来的值为0x1；
- Master0去写0xA000这个地址，这个地址之前已经读过了返回ExOKAY，并且中间没有别的Master**写过**，所以没问题，写下0x3，返回ExOKAY；
- Master1去写0xA000这个地址，这个地址之前已经读过了返回ExOKAY，**但是中间有别的Master写过，所以Exclusive write失败，返回OKAY**；（对于Exclusive access操作而言，如果Slave返回的是OKAY而不是ExOKAY则代表Exclusive Failure）

![img](D:\lqh\Typora\图片\v2-24623adac725dc81f647035ba0b45afa_720w.webp)

上述机制，其具体的微架构实现取决于应用场景，一般是用一个MAP即表格进行记录，表项从32~1024不等，本人之前参与设计过一款原子操作单元，大致就是使用上述的机制实现的。这种模块一般是位于Interconnect内部。BOOM处理器也有类似的模块，留意一下s2_sc_fail的条件。

表项内部如何进行赋值，可以用状态机实现相应的逻辑。一开始是Open状态，当需要Load Exclusive的时候，进入Exclusive模式，相应的地址和ID存到表格中，如果同样的Master后续的Store Exclusive成功的话或者清空的话，则跳回到Open状态，代表完成了一次Exclusive Operation，相应的表项也可以清空。如果别的Master对同样的地址进行Store Exclusive的话，则会查表发现还占着，因此Fail，写不进去。![img](D:\lqh\Typora\图片\v2-b8a4cda6515340baa2261bde457387d2_720w.webp)

再说一遍，上述只是一种实现机制，只要你能满足协议的要求，你的微架构想怎么设计都可以，取决于你自己的个人习惯。

## 2、AXI的响应信号

AXI的响应有两种信号，分别是RRESP和BRESP。其中的**RRESP**实际上是对应READ burst的每次transfer(beat)，而BRESP对应的是整个WRITE burst，即一个完整的transaction全部完成以后才会回**BRESP**。什么是transfer，什么是burst和transaction，我之前已经讲过了，大家可以回过头看一下，这里默认大家清楚这些概念。

- **00代表OKAY**，普通的访问如果是OKAY的话则代表访问成功，如果是Exclusive访问的话，返回OKAY则代表失败；
- **01代表EXOKAY**，代表Exclusive访问成功；
- **10代表SLVERR**，即Slave ERROR，Slave指定的ERROR。代表访问Slave成功了，但是Slave要求返回Error，比如写了不该写的地方；
- **11代表DECERR**，即Decode ERROR，用来代表访问了禁止区域，比如跨4K了，或者不准读写的地方，或者访问到了Default Slave；

实际上10和11没有绝对的界限，比如同一种错误，访问禁止区域，你回复SLVERR或者DECERR实际上都可以的，不一定要完全符合ARM官方文档的描述。

![img](D:\lqh\Typora\图片\v2-b773274d6d710df5d7ea5863690ec798_720w.webp)