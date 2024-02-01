# AXI Ordering Model、非对齐访问等

## 1、Ordering Model

在前面的文章我们已经知道了，AXI支持**Outstanding**以及**乱序操作**，这背后都需要ID的支持，所谓的ID其实就对应着一个编号，不同的应用场景我们可以授予ID不同的含义。一般ID的不同对应着不同的Master。对于同一ID的传输，必须保序。对于不同ID之间的传输，可以乱序。

之前已经给大家解释过什么是Outstanding了，这里再重复一遍，如下图所示：写地址通道发出了A地址，此时A地址的写响应通道还没有响应，但仍然可以在此之前发出B地址。这样做的好处，就是可以充分利用带宽。如果等A地址的通信完成再发起B地址的写，中间可能会浪费若干个Cycle，没有充分利用带宽。

![img](D:\lqh\Typora\图片\v2-0ecef9712cced31946f5c1981252a362_720w.webp)

### 1.1、Transfer ID

**接下来给大家讲解Transfer ID相关的信号**，对于AXI3协议而言，每个通道都有对应的ID信号，而在AXI4中，WID进行移除。至于为什么移除，后面再进行讲解，我们先看这五个信号对应的作用。![img](D:\lqh\Typora\图片\v2-24154779b52e5c95b12f7c3b8cd4b03d_720w.webp)

以下是有关ID必须支持的一些点：

- 对于AXI transfer，都有着对应的ID号；
- 对于一次transaction的所有的transfer，使用的都是同一个ID号；
- 对于同一个Master，可以使用不同的ID（同一个Master但是多线程）；
- Slave的ID应该是可配置位宽的；（需要一些额外的信息）

### 1.2、Write Ordering

接下来我们讲解一下**Write Ordering**相关的规则（以下的讲解基于AXI3协议）：

如下图所示：

- 先发出了地址为A，AWID为0的写请求，然后又发出了地址为B，AWID为1的写请求，**这就是前面所讲的Outstanding；（其数量称为Write issuing capability）**
- 可以看到红色的写请求虽然在蓝色的写之后，但实际上先完成了，也就是说**对于不同的ID，整个transaction是可以乱序的；**
- 对于蓝色的写和红色的写，可以看到在蓝色的写中间穿插了红色的写，也就是说对于不同的ID，彼此之间是可以**Interleave**的（其数量称为Write interleave capability，在AXI4中已移除WID，因此也不支持Interleave，移除的原因是因为设计过于复杂，且需要消耗额外的大量面积）；
- 我们再看，无论是蓝色还是红色还是绿色，都是先写的0，再写的1，再写的2，最后再写的Last。也就是说**对于同一ID而言，内部必须是保序的；**

![img](D:\lqh\Typora\图片\v2-f4cb8c8dac4087d4e2352210155528ba_720w.webp)

### 1.3、Read Ordering

**讲完了Write Ordering，我们再来看一下Read Ordering**。实际上是差不多的，但有一点需要注意，就是读的话每一次transfer，Slave都会返回相应的RID，而写的话，只有最后一笔，才会返回相应的BID，对应整个Transaction。

我们看下面的例子：

- 可以看到先发出请求的地址后返回数据回来，也就是**读也支持Outstanding**；
- 红色的读操作先完成，也就是**不同ID之间支持乱序**；
- 红色的读中间包含着蓝色的读，蓝色的读中间也包含红色的读，也就是**读也支持Interleave**（在AXI4中依然保留）
- **同一ID内部之间是保序的（不保序Master怎么知道你返回的是哪个地址的数据？）**；

![img](D:\lqh\Typora\图片\v2-6d6002440dab0b3e565da7ecc9e5f069_720w.webp)

### 1.4、Read/Write Ordering

很多时候我们不是只读或者只写，而是先读一个地址，然后进行一系列的运算再写回去。首先AXI协议它自己是不支持这一机制的，也就是不保证读和写之间的Ordering，如果想实现该机制，应该自己去实现，以下是实现的一些建议：

- Master必须等相应的BRESP返回以后才允许对同一地址的读；
- Master必须等相应的RRESP和RLAST返回以后，才允许对同一地址发起相应的写；

![img](D:\lqh\Typora\图片\v2-fc5098377880123ca12f95e3d5184ca5_720w.webp)

实际上CPU的一些Memory barrier机制，就是通过这种方式实现的，正是因为这种方式不允许Outstanding，不允许乱序，因此带宽利用实际上很差，所以特别的慢。如果跑过相关指令的朋友，应该是可以切身感受比一般的读写，要慢了起码一个数量级。正是如此，大家也应该可以感性的理解AXI的Outstanding对吞吐量的提高有多显著的帮助。

上面这个例子是对于同一个Master而言，如果不同Master之间你想要保序，那就得依靠之前说的原子指令了。

### 1.5、AXI ID uses

前面提到过，Slave的ID应该是可配置位宽的，因为需要一些额外的信息。比如根据ID，怎么返回给对应的Master？（很多时候并不是一个Master就对应着唯一的一个ID，不同的Master使用同样的ID也是有可能的！）

如下面的例子，通过在Slave侧的最低bit，用该bit区分这次transaction是来自Master0还是Master1。![img](D:\lqh\Typora\图片\v2-705e00bcf74245e1ef795c51a2031821_720w.webp)

比如Master1发出了一个访问，Interconnect相应的在最低比特添加了一个1，这样Slave返回的时候，Interconnect就知道这次Transaction是对应于Master1，则不会回复给错误的对象。有个时候甚至是Interconnect连Interconnect，这个时候就需要给ID增加更多的bit了。

此外通过id知道这次transaction是来自哪个Master，还可以做很多额外的事情，比如对于同一ID的保序操作，对于同一ID分配相应的outstanding id，不同ID之间的乱序等等。总而言之：**如果你的设计中Slave需要知道这次Transaction来自于哪个Master**，那就不妨使用该机制。![img](D:\lqh\Typora\图片\v2-86af8886fba3c9690e06858299945634_720w.webp)

对应的也有ID padding，用来保证位宽的统一，如下面的例子中，Master0发出的ID只有2bit，则在相应的高bit添加0，用来做位宽统一，在低bit增加0，说明是来自Master0。![img](D:\lqh\Typora\图片\v2-29116e167b644643fd6d3941118c0977_720w.webp)

## 2、Data Buses

我们之前其实已经接触过Data bus了。实际上就是读数据通道和写数据通道，但之前讲的都是最简单的情况，该通道上还有一些额外的信号帮助我们用在特定的场合，以下展开介绍。

### 2.1、Write Strobe

**首先是Write Strobes信号**，Strobe信号用处非常广泛，只要用到AXI，基本都会实现该机制，即使是自定义总线基本也都会实现该信号。Strobe的英文翻译一般叫做“闸门”，听上去就很直观了，其类似于掩码信号，**当我们只想写特定的字节**，就需要使用该信号。Strobe为高相应的数据可以通过，进行传输，否则则会被堵住。

该信号实际上没有值得多讲的地方，看下面这个图就知道怎么使用了，对应的bit对应相应的Byte。注意WSTRB是可以不连续的，也可以全为0，实际上。![img](D:\lqh\Typora\图片\v2-e9beb00f4b704cfe0a0c3add53c1a7b2_720w.webp)

### 2.2、大小端问题

**AXI实际上没有专门的信号，去表示这次传输是大端模式还是小端模式**，自己设计的时候应该要清楚，使用的是大端模式还是小端模式，另外原则上，同一次transaction，大小端模式是不允许变化的。

此外AXI认为的大端模式是BE-8（Byte Invariance Endianness Scheme）格式。这是什么意思呢？

- 对于32bit，就是颠倒整个顺序；
- 对于16bit，是在**内部之间**进行颠倒，而不是使用另外的16bit，大家看图想必可以明白（字节内部的8bit是没有变化的）；
- 对于8bit，实际上是一样的；

![img](D:\lqh\Typora\图片\v2-8ae69f2f0090a8c679f721f3ba43984b_720w.webp)

## 3、非对齐访问

通常情况下，对于32bit的传输，一般是要满足4-Byte对齐，即访问地址的低两bit应该为00。AHB便是如此，不支持非对齐访问。而AXI非常贴心的加入了非对齐访问机制，可以满足用户相应的需求。

**对于AXI的非对齐访问，非常直观的判断标准就是AxADDR的值是否和AxSIZE相匹配**，如果AxSIZE对应着8Byte，那么AxADDR的低3bit应该就为0，否则就是非对齐访问。

我们看下面的例子：

第一个例子AxADDR为0x01，则对应的bit应该是从8bit开始，也就是只写1、2、3这三个字节，不会写到4这个字节！其机制就是不会跨整个DATA Size去访问，其内部的原理是，**还是从对齐地址开始算，但是会根据是否对齐，判断哪个Byte不用去写**，比如这个例子中第一次transfer就不用去写第 0 Byte。然后第二次transfer写4、5、6、7这四个字节，然后第三次、第四次、第五次。<img src="D:\lqh\Typora\图片\v2-53ce7dc663fd0078c6fd1a8cff72a83a_720w.webp" alt="img" style="zoom:150%;" />

再看第二个例子，其地址为0x03，对应的AXSIZE为16bit(这里指的是写数据有效部分是16bit)。第一次transfer相应的也只会去写3这个字节。然后第二次transfer会去写4、5字节，第三次transfer写6、7字节，第四次transfer写8、9字节，第五次transfer写A、B字节。其实第二次transfer和第三次transfer，你如果是读的话，假如数据位宽是32bit，那么按理说读到的都是整个4、5、6、7这四个字节，但设计中一定要注意好了，该用哪两个字节，别因为重复导致弄错了。（**传输的时候，数据总量肯定是跟BUS宽度一致的，但是多少字节有效，是根据你的SIZE决定的**）<img src="D:\lqh\Typora\图片\v2-5957de75703a1d17ac59ebec82bb9ad4_720w.webp" alt="img" style="zoom:150%;" />

很多IP设计的时候实际上不支持非对齐访问，是否需要非对齐访问取决于你的应用场景，加上非对齐访问的话，会让逻辑变得很复杂，因为要计算对齐地址，取相应的数以后还要进行字节选择，会增加组合逻辑，导致频率受到影响，如果不是硬性需求不用加上对非对齐访问的支持。
