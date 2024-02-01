# AXI突发传输和AXI控制信号

## 1、AXI突发传输

### 1.1、AXI突发传输时序图

**AXI总线是基于突发传输的，并且AXI的突发是只需要给一次地址信号即可**，这样就免去了地址计算的逻辑。对于只存在给一个地址给一个数的传输场景，不建议使用AXI总线，APB即可。

以**<u>读突发传输</u>**为例，可以看到在T2的时刻AR通道握手成功，成功传输了地址信息。然后SLAVE就可以根据该地址返回相应的读数据，每次RVALID和RREADY握手成功的时候，返回一个数据，称之为一次transfer，握手一次返回一次数，当返回最后一笔数据时候，相应的**RLAST**也需要拉高，来表示最后一笔数据已经给出，整个Transaction到此结束。![img](D:\lqh\Typora\图片\v2-a6e51551babb93b78c9d270b606488f9_720w.webp)

之前已经说过，AXI支持Outstanding，在Master给出一个地址以后，可以紧接着再给出一个地址。即使第一个地址的数据还没有返回，这种情况的波形如下图所示：![img](D:\lqh\Typora\图片\v2-19287a9f0b12581352755c6cfd2740b9_720w.webp)

我们再看一下**<u>突发写操作</u>**，同样的Master给出写地址，握手成功，然后给出写数据，当给出最后一个写数据的时候，**WLAST**拉高，然后B通道返回**BRESP**和**BVALID**，握手成功代表写transaction结束，通信完成。

有一点需要注意，AXI3中给出了WLAST即可返回BVALID，但是AXI4中规定了，必须写地址通道握手成功了，也给了WLAST才允许返回BVALID，显然下面这个图满足了AXI4这个要求。这个实际上还是比较能够理解的，因为作为Slave，即便你已经收到了所有的数据，如果你没有收到相应的地址，你是不知道写往何处的，这个时候给出BVALID当然是不太合乎逻辑的。（此外虽然没有硬性规定写数据必须在写地址通道之后，但我建议还是最好这样设计，这样更符合直觉，和读也更加相对称）![img](D:\lqh\Typora\图片\v2-d0271bfdbc05fe218c3a7ab8c2388ed2_720w.webp)

### 1.2、AXI突发传输信号

看完了AXI的突发读和突发写时序图，我们进一步学习突发传输相关的信号，细心的读者可能发现了，上面的控制信号只给了地址信号，写数据大小，突发长度都没有体现，实际上面只是简化版本的，忽略了控制信号细节，以下为大家梳理一下，**跟突发相关的控制信号**总共是有以下三个信号。

| 信号         | 描述                                           |
| ------------ | ---------------------------------------------- |
| AxLEN[3:0]   | 1-16笔突发传输数据(AXI3)                       |
| AxLEN[7:0]   | 1-256笔突发传输数据                            |
| AxSIZE[2:0]  | 1，2，4，8，16，32，64，128 bytes per transfer |
| AxBURST[1:0] |                                                |

首先是突发长度，**AWLEN和ARLEN**两个信号指定了每一笔突发传输有多少笔，对于AXI3，支持1-16笔transfer，分别对应000-111，对于AXI4，支持1-256笔transfer，分别对应00000000-11111111。

值得注意的是，对于写而言，该信号很多时候没什么用，**因为WLAST才是真正的最后一笔写传输的标志**，有些设计AWLEN甚至就是默认值，主机那边控制好WLAST即可，这种方法也可以，但不建议，WLAST按理说要和AWLEN对应好！

然后是突发数据大小，**AWSIZE和ARSIZE**两个信号，该信号用来标志传输的数据位宽哪些bit是有效的，一般是从低位开始算，比如ARSIZE为'b001的时候，则代表transfer的size为2Bytes。对应ARDATA[15:0]是有效的。

-------------------------------

**NOTE**：==AxLEN 指的是一个transaction里面有多少个transfer，而AxSIZE指的是每个transfer里的每笔数据DATA里哪些bit有效。==

----------------------------

最后讲一下突发传输类型，**AWBURST和ARBURST**两个信号，AXI中一共支持三种类型，比AHB更简洁。

![img](D:\lqh\Typora\图片\v2-728b77fb609ef4628d4db70082b5a3ec_r.jpg)

**第一种类型为FIXED类型**，顾名思义，地址固定，这个时候可能是普通的传输，即Burst length为1。也有可能是重复访问相同的地址，比如加载或者清空FIFO的情况。
**第二种类型为INCR类型**，这种情况下地址是递增的，支持非对齐访问。增加的地址大小和AxSIZE相关。
**第三种类型为WRAP类型**，这种情况地址也是递增的，但是增加到某个地址以后会回过头去访问，也就是回环，前面我已经讲过了。这种情况实际上是用来写Cacheline的，用在critical word first的模式下，因此它实际上是必须要对齐访问的，因为cache line典型情况是64或者32 byte，但CPU最着急写或者读的是其中的某个word，这种情况下就需要用到WRAP类型，因此它的Length一般也严格限制在2、4、8、16。对应着cache line不同的块。

## 2、AXI其它的一些控制信号

这些控制信号不是必用的，取决于实际应用的场景，AXI提供这些信号作为额外的功能支持。

**AxUSER信号**：扩展信号，提供给用户进行自定义应用，用户可以自定义该信号用来传输想要的信息。

### 2.1、保护功能支持

分别是**AWPROT**和**ARPORT**，该信号为3bit，因此也提供了三种类型的访存保护（其实就是附带一些额外的信息，用来避免非法的transaction操作）。

![img](D:\lqh\Typora\图片\v2-7215cfce2550df711b8adec538cc2bf5_720w.webp)

**AXPROT[0]**用来区分是**普通访存**还是**特权访存**，如下图所示，比如操作系统这一块地址，普通的访存当然是不被允许的，一般是拥有较高的权限，才允许进行访存，体现在硬件上实际就是Normal access还是Privileged Access，这个时候就得借助该信号。![img](D:\lqh\Typora\图片\v2-be11e6e17b6448f80224cf4dc40c233d_720w.webp)

**AXPROT[1]**用来区分是**Secure**还是**Non-Secure**，如下图所示，**Secure Region**一般是内存中**特别机密**的信息，同理这也需要处理器的支持，只有Secure的程序才允许访问Secure Region。![img](D:\lqh\Typora\图片\v2-0c3e62810ddea27b4687adabd326972b_720w.webp)

**AXPROT[2]**用来区分是**Instruction**还是**Data**，这个用于区分CPU是取指令还是取数据，实际上用的很少，很多时候可能取得是指令，但是标明的是Data，ARM的官方手册也有这么一句话：This indication is provided as a hint and is not accurate in all cases. For example, where a transaction contains a mix of instruction and data items. It is recommended that, by default, an access is marked as a data access unless it is specifically known to be an instruction access。
因此对于该比特，大家选择性的使用即可，一般是用不到的。常用的就上面的[0]和[1]bit。

最后关于保护信号做一个总结，如下所示：![img](D:\lqh\Typora\图片\v2-90cf09c0bdab786f6fb730af16ed2e0f_720w.webp)

### 2.2、Cache支持

对于现代的SoC而言，Cache可以存在于SoC的很多地方，比如有以下场景：

- 挨着处理器核（L2 Cache）
- 在Interconnect内部
- 靠近Memory控制器（L3 Cache）

正是因为系统中有了这些Cache，相应的也需要Cache相关的信号，来帮助我们存放数据到想要的位置或者从想要的位置取数据。（看懂这小节需要基本的Cache相关的知识，如果没有请去看计算机组成与设计第五章或同样类型的参考资料）

**AXI对于Cache支持使用了AWCache和ARCache这两个信号**，分别都是4bit，每个比特的含义如下所示。![img](D:\lqh\Typora\图片\v2-de400cd970f70fb4a313657ef32887ba_720w.webp)

**AXCACHE[0]**用来区分是**Bufferable**还是**Non-bufferable**，Bufferable即这笔Transaction是否可以存在于Buffer中，比如我要往某个地址写个数据，我是否必须真正的写在了那个地址，然后收到了response才算传输结束，还是写到Buffer中，收到了Response就算结束。

对于写外设操作而言，一般是不允许Bufferable的，因为你没有写入外设的寄存器，外设没有按照你的预期产生工作状态的改变，你就认为传输结束了，是不符合预期的。只有写一些数据的时候，比如计算的中间结果，这个时候对你写在哪里不太敏感，是允许Bufferable的。**一般该信号只用在写上**，因为读的时候该Buffer可能已经被覆盖掉了，不是想要的值了，当然也不绝对，看你的设计和替换方式是怎样了。

----------------------------------------

**AXCACHE[1]**用来区分是**Cache able**还是**Non-Cache able**，在AXI4中更改为**Modifiable，**这个Modifiable实际上更加好理解**。**官方手册的说法是该比特用于决定实际的传输事务和你原本的传输事务必须是否相等，这么听起来可能有点抽象，以下是具体的实例。

以写为例子，比如你第一次是往地址0写，第二次是往地址1写，如果是Cache able的，系统检查这两个地址的内存属性又是一样的，那么它就可以把你的写Merge到一起。
以读为例子，也就是可以进行指令预取（读一片指令），也可以进行多个transaction合并（比如连续几个连续的DDR空间），**该比特需要和AXCACHE[2] [3]bit一起使用。**
值得注意的是，这个**是否可以Modifiable实际上也决定了是否允许写入Cache**，如果不准Modify，那么就意味着你要么写入Buffer要么写入真正的目的地址，这样自然不涉及Cache了，因此该比特如果为低，代表你的访存操作和Cache没有关系。

------

**AXCACHE[2]和[3]**用来分别表示读分配和写分配，所**谓的分配指的是我们什么情况下应该为数据分配cache line。cache分配策略分为读和写两种情况**。当AXCACHE[2]为高的时候，代表需要读分配。如果读分配为高的话，AXCACHE[1]必须为高，因为你都不准Cache able了，那自然不存在分配了。写分配也是同理。

读分配的含义是，如果是一次读传输，发生了miss，那么应该在cache中给它分配相关的Cacheline，这个过程叫做Cacheline fill(地址对应的cache line不存在，也就是cache miss，需要从外部内存读取，然后填充到对应cache line，这一过程被称为cache line fill)，如下图所示。![img](D:\lqh\Typora\图片\v2-70deabf23cefe971a0e2d7247ea6e066_720w.webp)

**当写Cache Miss的时候，才会考虑写分配策略**。当我们不支持写分配的情况下，写指令只会更新主存数据，然后就结束了。当支持写分配的时候，我们首先从主存中加载数据到cache line中（相当于先做个读分配动作），然后会更新cache line中的数据，如下图所示，当出现写Cache Miss的时候，会先从主存加载数据到Cacheline，然后将要写的那一部分更新到Cacheline相应的位置（图中先Cacheline fill，然后写那个蓝色长方形的灰色格子）。![img](D:\lqh\Typora\图片\v2-b22706876abf2d3f25db9db654e529c0_720w.webp)

以下是该4bit可能的编码组合：![img](D:\lqh\Typora\图片\v2-27d30f0bad26054384dc1b35a58cb26e_720w.webp)

### write through(透写)

![img](D:\lqh\Typora\图片\v2-400b2cf027ea58f4f429c79e5848e901_720w.webp)

在透写（**Write Through**）场景中，数据同时更新到缓存和内存（**simultaneously updated to cache and memory**）。这个过程更简单、更可靠。这用于没有频繁写入缓存的情况（写入操作的次数较少）。

它有助于数据恢复（在断电或系统故障的情况下）。因为我们必须写入两个位置（内存和缓存），数据写入将经历延迟。虽然它解决了不一致的问题，但它的问题是在写操作中使用缓存的优势，因为使用缓存的全部目的是避免对主内存的多次访问。

### write back(回写)

![img](D:\lqh\Typora\图片\v2-bec34c4b9e8b4f7a81cca1f41fb48eb7_720w.webp)

回写（**Write Back**）也被称为延迟写入（**Write Behind / Write Deferred**）。也就是说，最初数据只在缓存中更新，稍后再更新到内存中。对内存的写入动作会被推迟，直到修改的内容在缓存中即将被另一个缓存块替换。

### **写未命中（Write Miss）**

写操作时，cache line中没有相应地址的block块，此时无法对cache进行写，发生Write Miss。由于在写操作时没有数据返回给请求者，所以需要对写未命中做出决定，是否将数据加载到缓存中。这是由以下两种方法定义的：写分配（**Write Allocation**）和无写分配（**No Write Allocate**）。

#### **写分配（Write Allocation）**

![img](D:\lqh\Typora\图片\v2-27c93f5277303393a6b12275b08441e0_720w.webp)

写分配（**Write Allocation**）也被称为写时取（**fetch on write**）。即：未写入位置的数据首先被加载到缓存，然后是写入命中操作。在这种方法中，写入未命中类似于读取未命中。

#### **无写分配（No Write Allocation）**

![img](D:\lqh\Typora\图片\v2-580584751b635b004657fbbeaa7c556c_720w.webp)

无写分配（**No Write Allocation**）也称为 **Write-no-allocate** 或 **write around**。即：未写入位置的数据不加载到缓存，而是直接写入后备内存。在这种方法中，仅在读取未命中时将数据加载到缓存中。这里数据直接写入或者更新到主存而不干扰缓存。当数据无需立即再次使用时，最好使用这种方法。
