# **AHB-lite**

## **前言**

文档介绍的是**AMBA3**的AHB-lite协议，最新的AMBA5中增加了很多信号，具体的ARM处理器使用的AMBA协议版本可以参考[AHB和ARM处理器](#jump9)。

## 1、一个典型的基于AHB总线的微控制器架构

AHB和APB之间通过转接桥Bridge进行连接，这个转接桥实际上就完成了AHB和APB之间的协议转换。通过这种模块化的划分方式，可以让系统工作在不同的时钟域下，低速外设不会影响高速外设，进而高速模块可以跑到较高的时钟频率。可以这么说，AHB的设计目的就是解决APB通信速率过慢的问题，从而提高系统的时钟频率和带宽。

![img](https://pic1.zhimg.com/v2-f2c730245302ebf6a591735c40be775c_r.jpg)

## 2、AHB的发展历史

ARM公司在1999年推出AMBA2，首次提出了AHB协议，将其用于高性能模块之间的连接，可以连接多个Master和多个Slave。

其架构图如下图所示，可以看到通过仲裁器对多个Master进行选择，对于同一时刻，**让总线上只有一个Master在工作**。译码器对Slave进行选择，以完成完整的通信过程。此外Slave必须1KB边界对齐。

<img src="https://pic3.zhimg.com/80/v2-1a53898d2fd5f3f7e93d63c0a5697ff6_720w.webp" alt="img" style="zoom:80%;" />

ARM公司又在2006年推出了AMBA3，其中包括了AHB-lite协议，其针对的是<u>**单个Master和多个Slave**之间的连接</u>。它可以看做AHB协议的简化版本，因为在很多场景中实际上只有一个AHB的Master，并不需要像AHB那样复杂的硬件架构，采用AHB-lite可以简化逻辑设计。

2015年又推出了AMBA5，其对AHB增加了独占传输、扩展存储类型、安全传输、原子性访问等功能，这些功能在AXI协议中也有体现（AXI协议部分讲解）。

## **<span id = 'jump'>3、AHB-lite协议硬件架构</span>**

**AHB-lite硬件架构实际上跟一对多的APB架构非常的像**，区别只是信号不一样，其硬件连接非常非常相似。这一代协议也是目前用的最多的AHB协议。

可以看到其硬件架构如下图所示，由于只有一个Master，所以不需要仲裁器，只需要译码器和那个Slave到Master的MUX即可。

![img](https://pic1.zhimg.com/80/v2-8b9a6d54054df5948040d4c6fca3c3ac_720w.webp)

![img](https://pic4.zhimg.com/v2-dae9c811120fb886aa1138bcf702926b_r.jpg)

- 首先AHB-lite可以看做是AHB的简化版本，也可以进一步的认为就是AHB的子集。
- AHB-lite由于**<u>只有一个Master</u>**，因此也就不需要Arbiter仲裁逻辑。同时由于省略了一些信号，AHB-lite的Slave设计也相对简单。
- AHB-lite实际上也可以连接多个Master，即采用完整的AHB Interconnect，同时采用多层的架构。让每一个主设备认为是专属于自己所在的层。
- 如下图AHB-lite Master 1位于layer 1上，而AHB-lite Master 2位于Layer 2上。这样它们彼此是感受不到别的Master的存在的，因此也就实现了多个Master多个Slave。对于Master 1,在它看来它连接了三个slave，而对于Master 2，在它看来连接了5个slave。从宏观看，slave 4和slave 5是Master 2的私有外设。
  总而言之，没有绝对的一对多的连接，只要你想肯定有办法实现多对多。当然这都是特殊的情况才会使用的方法，对于初学者而言暂时不需要掌握这种情况。

![img](https://pic1.zhimg.com/80/v2-f1aafec1fece5bd3f14f44b509d774ac_720w.webp)

## 4、AHB-lite信号（总览）

AHB的信号都是以H开头，此外AHB-lite的硬件架构，可以分为四部分，分别是Master、Slave、Decoder以及MUX，因此官方协议将其信号列表也分为以下几部分：（信号名只有结合传输过程一起看才有意义，因此读者看下面的信号名，留个印象即可，重点应该放在后续的AHB-lite传输流程上）

### **4.1、全局信号**

| Name        | Destination | Description                    |
| ----------- | ----------- | ------------------------------ |
| **HCLK**    | Global      | 时钟信号，上升沿同步           |
| **HRESETn** | Global      | AHB-lite总线复位信号，为低有效 |

### **4.2、Master信号**

**<u>以下信号由Master产生</u>**：

| Name             | Destination    | Description                                              |
| ---------------- | -------------- | -------------------------------------------------------- |
| **HADDR**[31:0]  | slave和decoder | 32bit的地址总线（不是严格限制为32bit）                   |
| **HBURST**[2:0]  | slave          | 突发传输类型                                             |
| **HMASTERLOCK**  | slave          | 用来实现原子操作的                                       |
| **HPROT**[3:0]   | slave          | 保护控制信号                                             |
| **HSIZE**[2:0]   | slave          | 指示传输的大小，通常为字节、半字或者字。最大可到达1024位 |
| **HTRANS**[1:0]  | slave          | 指示传输的类型                                           |
| **HWDATA**[31:0] | slave          | 写数据，一般建议最小为32位，最大可以为1024位             |
| **HWRITE**       | slave          | 为高代表写传输，为低代表读传输                           |

### **4.3、Slave信号**

**<u>以下信号由Slave产生</u>**：[结合硬件架构图](#jump)

| Name             | Destination | Description                                                  |
| ---------------- | ----------- | ------------------------------------------------------------ |
| **HRDATA[31:0]** | Mux         | 在读操作期间，读数据总线即该信号，将数据从所选从设备传送到多路复用器。然后，多路复用器将数据传输到主机。 |
| **HREADYOUT**    | Mux         | 当为高电平时，HREADYOUT信号表示总线上的传输已完成。该信号可以被驱动为低电平以延长传输（即为0，可以实现反压） |
| **HRESP**        | Mux         | 传输响应信号，当低电平时，HRESP信号指示传输状态为正常。当为高电平时，HRESP信号指示传输状态为错误。配合HREADY可以传递更多的信息，具体的后面说 |

可以看到Slave产生的信号都是送给MUX的，由MUX选择其中一个，送到Master。

### **4.4、Decode信号**

**<u>以下信号由Decode产生</u>**：

对地址译码，当满足条件的时候，将多组HSELx的其中一个拉高，已选中需要选中的Slave。

此外这个信号不光要给Slave，通常还要给MUX，以方便Mux从多个Slave中选中一个，返回给Master。这个信号可能是HSELX本身，也可以是HSELX导出的信号。

| Name      | Destination | Description                                                  |
| --------- | ----------- | ------------------------------------------------------------ |
| **HSELx** | Slave       | 每个从机都有自己的HSELx信号，这个信号指示当前传输指向被选中的从机。当从机被开始选中时，它必须监视**HREADY**来确保之前的总线传输已经完成。 |

![img](https://pic1.zhimg.com/v2-4ede9d2876da1faf95fe39810926557c_r.jpg)

### **4.5、MUX信号**

**<u>以下信号由Mux产生</u>**：

| Name             | Destination      | Description                                               |
| ---------------- | ---------------- | --------------------------------------------------------- |
| **HRDATA**[31:0] | Master           | MUX根据地址译码结果选择对应的从机的HRDATA，然后传递给主机 |
| **HREADY**       | Master and Slave | 高电平时，告诉主机和所有从机之前的传输已经完成            |
| **HRESP**        | Mater            | 传输响应，根据地址译码结果选择                            |

HREADY和HREADYOUT不一样，**HREADYOUT**是由Slave产生，用来指示当前被选中传输的从机是否准备好进行真正的数据传输，和APB中的PREADY类似。**HREADY**是由MUX产生，用来指示主机和所有从机，整个总线上是否有未完成的数据传输。

## 5、AHB传输流程

### 5.1、Basic Transfers

AHB-lite和APB类似，也将传输分成了两个阶段，**地址阶段和数据阶段**（APB中是setup phase和access phase）。

- **Address**：地址阶段，通常持续一个周期，除非是上一次传输的数据阶段一直没有结束。
- **Data**：数据阶段，可能会持续很多个周期，受到HREADY的控制。

**HWRITE**用来控制数据传输的方向：

- 当**HWRITE**为**高**的时候，代表这是一个**写传输**，主机在写数据总线上**广播**数据，HWRITE和HWDATA等信号会给所有的Slave，所以叫做广播。但是HSELx会根据译码结果，只拉高一个，从而选中特定的Slave。
- **HWRITE**为**低**的时候，代表这是一次读传输，从设备必须在读数据总线上生成数据

在没有**wait states**的情况下，读传输如下所示。可以看到确实是分成了两个阶段，地址阶段和数据阶段。第一个阶段实际上就是**Decoder**在工作，根据地址选中相应的Slave！然后在第二个阶段Slave和MUX完成剩下的工作。此外大家有没有发现，两个阶段的地址是不一样的？也**就是说第一次传输的数据阶段和第二次传输的地址阶段，是可以重叠的，这个机制非常的好啊，可以流水线读或者流水线写，大大节省了时间。**考虑给10个不同的地址写10次数据，AHB-lite只需要11个周期，而APB需要20个周期，差距非常大。

![img](https://pic4.zhimg.com/v2-fc894335941f5dc1d91679a0ed55bb7f_r.jpg)

![S6Z67I$SQ{QJ}YOLJ~AVO~6](C:\Users\admin\AppData\Roaming\Tencent\QQ\Temp\S6Z67I$SQ{QJ}YOLJ~AVO~6.png)

### **5.2、带Wait states的传输**

首先看读，在第一个时钟周期，主机发送地址信息和控制信息。而从机暂时无法回复相应数据，因此将HREADY拉低，从而实现反压，主机因此就不能改变它的信号（**注意此时HADDR和HWRITE都已经变了，也就是第二次传输的地址阶段了，因此即使阻塞住，也不影响两次传输的流水线操作！**）。当从机可以返回数据了，就拉高HREADY，改变相应的HRDATA，主机在下一个时钟周期获取这些信号，并且能进入到第二次传输的数据阶段。

![img](https://pic1.zhimg.com/v2-5a5df68f83007ed79cbc82608c6e837c_r.jpg)

写的话其实更简单，重点就是从机在反压主机，主机的信号不能变。(APB总线反压是在ENABLE阶段，PREADY=0时，保持在ENABLE状态，PSEL、PENABLE、PWRITE、PADDR、PWDATA保持不变)

![img](https://pic1.zhimg.com/v2-d75dbfa78f0f01274d20d22f38237c58_r.jpg)

此处反压时，需要保持第二个传输的HADDR和HWRITE保持不变，避免进入下一次传输。

## **6、AHB-lite控制信号**

![img](https://pic4.zhimg.com/80/v2-ecd8fbd12494c2700f27142779c1271f_720w.webp)

总线协议，本质上就是完成主机和从机之间的通信传输，因此我们基于上面的硬件架构图去讲解，大家阅读下文的时候要有意识的去思考这个硬件架构图。

### **6.1、Transfer Type**

**HTRANS**[1:0]信号用于指示当前的传输类型，一共有四种传输类型：

- **IDLE**
  - 没有数据传输,其它的控制信号和地址信号因此也就不起作用。

- **BUSY**
  - 没有数据需要传输。
  - 这个信号可以在[突发传输](#jump1)中（**什么是突发传输后面讲，因为官方文档也是这个顺序，如果这一部分看不懂的，先硬着头皮看完，然后等看完后面的突发传输再回过头来再看一遍**），用来插入空闲的CYCLE，表示主机在忙，也就是说传输还在继续，但是处于暂停状态。
  - 在不指定突发长度的情况下，在最后一拍用BUSY去传输，来表明这是Burst的最后一笔，这一鸡肋的机制被AXI的LAST信号完美替代。
  - **实际上BUSY这个状态很少见**，在CORTEX-M系列基本上不会有这个BUSY状态，在大部分情况下都是使用NONSEQ和SEQ传输类型，因此可以暂时不去掌握该类型

- **NONSEQ**
  - 需要传输数据。
  - 可能是发一笔数据（Single传输），也可能是Burst传输的第一笔transfer（在这里transfer指的是突发传输的一次读或者一次写，或者就是Single传输）。
  - 地址和其它控制信息和之前的传输没有关系！只取决于这次transfer本身。
- **SEQ**
  - 需要传输数据。
  - 用在突发传输中，代表连续的传输。
  - 地址需要增加（除非上一笔transfer是BUSY transfer）。
  - 其它的控制信号保持不变。

![img](https://pic2.zhimg.com/v2-0b9ab72cd500fdf9036a514b32178ded_r.jpg)							<span id = "jump2">4 byte 突发读</span>

由上图可以知道，这是一次突发读操作，实际上是4拍有效读：

- T0->T1：突发传输的**第一笔Transfer**，因此HTRANS应该为NONSEQ，HADDR为0x20代表起始地址.
- T1->T2：突发传输的**第二笔Transfer（伪）**，因为HTRANS为BUSY，可能是因为主机忙碌没有空去读，所以插入了这样一拍，HADDR增加0x4，HRDATA返回了0x20地址的数据，这一拍没有发生数据传输。
- T2->T3：突发传输的**第二笔Transfer（真）**，由于上一拍是BUSY，所以HADDR的地址不需要增加。HTRANS应该是SEQ。
- T3->T4：和上一拍差不多，第三笔Transfer。
- T4->T5：最后一拍读，由于Slave不能完成数据传输，因此将HREADY拉低，这一拍相当于wait state。
- T5->T6：最后一拍读，HREADY拉高，说明这拍主机可以将控制信号等信息发给从机。
- T6->T7：从机返回最后一拍读的数据。

### **6.2、Locked Transfers**

如果主机想发起一次带锁的访问，那么就需要**HMASTERLOCK**信号。至于什么是“锁”，请自行学习**操作系统**相关的知识（想做SoC的朋友一定要认真学操作系统和体系结构这两门课啊！）。

简单理解就是在**多主机**可以访问一个从机的情况下，需要确保这个时间段只有该主机对其进行访问，避免数据出现错误，如图3所示。（想象一下，主机2去写个数据，还没有写完主机1就去读同样的地址的数据，就拿到了旧的值，这样就不符合预期了，很多汇编指令有原子操作指令，从底层来看就需要硬件的机制来帮忙）。

![img](https://pic1.zhimg.com/v2-a541422706b58d7fb84a32eded603d00_r.jpg)

同样的，我们看传输协议，如图3所示。可以看到是一个读操作一个写操作，**HMASTERLOCK**为高说明这是带锁的操作，也就是希望这两个操作之间不希望被BUS的Interconnect打断！

![img](https://pic2.zhimg.com/v2-1a13b120a1e181170c1990dccce8858d_r.jpg)

以上图为例，slave 3可以被多个Master访问，假设Master 0发起了读又发起了写，没有原子操作的支持的话，假设Master 1也发起了读，我们本质上是希望Master 1读到Master 0写到slave 3的数据，但是Master 1的读很可能位于Master 0的写之前，这样就不符合我们的预期了。流程如下：

- Master 0读Slave 0
- Master 1读Slave 0（读到的还是之前的值，不是Master 0新写的值！）
- Master 0写Slave 0

因此就需要**HMASTERLOCK**的支持。他保证了这两笔是原子的，中间不会有任何的操作。这样就非常的赛高啊，完美实现了我们的需求。流程如下：

- Master 0读Slave 0紧接着Master 0写Slave 0（这是一个整体，不能被打断）
- Master 1读Slave 0

此外在**原子操作**（不可被中断的一个或一系列操作）的下一拍，通常插入一个IDLE，代表原子操作结束了。如图2的第三拍所示。

![img](https://pic2.zhimg.com/v2-0a15cbb7b8e9b1899c63fc5522aec609_r.jpg)

**<u>注意</u>**: 大部分Slave实际上不需要实现**HMASTERLOCK**信号，因为大部分Slave不会被多个Master所访问。只有能够被多个Master访问的Slave，才需要**HMASTERLOCK**信号，如多端口Memory Controller，就必须要实现**HMASTERLOCK**信号的机制。

最后再补充一下Locked Transfer和ARM处理器的知识：

- ARM9和ARM11处理器在使用**SWP指令**的时候发起的传输会是Locked Transfers（这两种处理器已经没人用了）
- Cortex-M3使用Locked Transfers，用在Bit-Band write上。（专门划分了一个位操作的区域，对这一区域的访问会自动转换成Locked Transfers，所以比较慢）
- Locked Transfer实际上用的很少了，SWP指令也被抛弃了（思考一下为什么），因此这一部分大家还是侧重于学习思想即可。

### **6.3、Transfer size**

HSIZE[2:0]用于标志数据传输的位宽，共有下图这几种组合。

![img](https://pic3.zhimg.com/v2-a88ebe186fe5d87614bcfd2c52ef3166_r.jpg)

----------------------------------------**NOTE**--------------------------------------

**The transfer size set by HSIZE must be less than or equal to the width of the data bus. For example, with a 32-bit  data bus, HSIZE must only use the values 0b000, 0b001, or 0b010.**

The HSIZE signals have the same timing as the address bus. However, they must remain constant throughout a burst  transfer. 

HSIZE in conjunction with HBURST determines the address boundary for wrapping bursts.

----------------------------



这个信号其实非常简单，对于信号自身而言没有任何难以理解的地方。

有一点需要注意的是它其实表明了在突发写或者突发读中**HADDR每一次需要增加多少**。我们最常见的是32bit，此时HSIZE=3'b010。因此HADDR应该每一次增加0x4（不要问我为什么地址加0x4对应32bit...）。因为存储器内部一个地址对应一个字节，要读写32位数据，地址就要一次加4。

![image-20240110110718453](D:\lqh\Typora\图片\image-20240110110718453.png)

HSIZE决定了地址偏移量(Address offset)，不同的地址偏移决定了WDATA里面用到的字节。

### <span id = 'jump1'>**6.4、Burst operation(突发传输)非常重要**</span>！

**突发(burst)**将**多个传输（Transfer）**作为一个单元（有时候可以叫Transaction）进行执行，而不是独立地处理每次Transfer，核心思想是将多次Transfer看做一个整体。对于主机而言，一次下发，即可实现连续的写或者连续的读。

我们重新看一下[图 4 byte突发传输](#jump2)，这就是一次突发读。共有4个beat(4次Transfer）。对于主机而言，只要下发一次命令，给个起始地址，给个突发传输的类型，给个Size。就不用管后续的东西了，剩下的Interconnect会帮助主机完成。

**HBURST[2:0]**

突发传输本质上有四种类型：

- SINGLE
  - 突发长度为1，也可以理解成不是突发传输。
  - **HTRANS**可以是IDLE或者NONSEQ。
- INCR
  - 非定长。
  - 不可以跨越1 K Byte的边界（因为AHB规定不同的Slave最小粒度为1 KB，因此跨越1 KB可能访问到别的Slave去）。
- INCRx
  - 定长，可以是**4、8、16笔transfer。**
  - 不可以跨越1 K Byte的边界。
- WRAPx
  - 回环（主要用于cacheline，具体的后面说），可以是**4、8、16笔transfer。**
  - Cacheline访问，critical word first（关键词优先）。

**首先看一下INCR4的例子**，如下图所示：

- 总共是4笔Transfer（因为HBURST为INCR4）
- HADDR每次增加0x4（因为HSZIE为Word，即代表32Bit）
- HTRANS，第一笔为NONSEQ，后面的为SEQ

可以看到有了突发传输，主机只需要给个地址，给个HBURST类型，给个HSIZE，就可以坐等4组数据的返回了！并且突发读是流水线的，理论上最快只需要N+1个Cycle（N为transfer次数）。

再思考一下：突发读或者突发写，和流水线形式的SINGLE TRANSFER是一样快的嘛？为什么？

![img](https://pic1.zhimg.com/v2-e464483cb0d9712c3cf86e0b5aea9b30_r.jpg)

再看一下WRAP8的例子，如下图所示：

- 总共是8次Transfer（因为HBURST是WRAP8）
- 地址每次增加4（因为HSIZE为Word，即代表32Bit）
- 地址不总是增加，从0x90开始，也就是critcal-word，增加到0x9c以后跳回了0x80
  - 为什么这么算呢？这是由WRAP8和HSIZE共同决定的，HSIZE决定了每次地址增加0x4，而8个0x4就是0x20，也就是0x20为一组。0x90则属于第五组（0x80~0x9C之间），因此地址是这样变化的（看一下图9，从中间开始增加，然后返回起点继续写）。

![img](https://pic4.zhimg.com/v2-cb2635b4e28061be7bf076de71a92a07_r.jpg)

![img](https://pic4.zhimg.com/80/v2-023a40d7240b5fd404d2e417e8ea59db_720w.webp)

最后我们回顾一下开始的思考题：突发读或者突发写，和流水线形式的SINGLE TRANSFER是一样快的嘛？为什么？

表面上来看，没什么区别。理论上都是N+1个Cycle。但是这是针对一拍可以回数的紧耦合SRAM而言的，如果访问的是DDR呢？

- 如果是突发传输，你只需要下发一次命令，DDR Controller可以帮你计算好，你总共需要读多少数据，比如每一拍是32Bit，突发长度是4，DDR控制器只要对DDR发一次命令即可，一次性读回128Bit的数据
- 如果是Single transfer，DDR Controller可不知道你下一笔传输的地址和这笔传输的地址只差了0x4，DDR Controller完全可能把你的第一次single transfer下发出去，然后又下发第二次，然后又下发第三次，然后...由于DDR的读取时序没有那么简单，不是完全的流水式的，因此这中间就可以阻塞很多个周期。
- 所以！突发传输和流水线的Single transfer是不一样的！能用突发传输就用突发传输，不要用多次Single Transfer。

## **7、Slave Response Signaling **

在Master发起一次传输以后，剩余的传输过程实际上是由Slave在控制。一旦传输开始，Master不能在中途取消这次传输。在之前的文章中，我们都是从Master的视角去看传输的，这篇文章引入Slave响应的视角，来建立完整的传输流程。

AHB-lite实际上通过HRESP和HREADY这两个信号的组合来控制传输响应，**HRESP**只有下面两个值：

- **OKAY**
  - 传输完成（**HREADY**为**High**）。
  - Slave额外需要传输周期，Transfer处于**Pending**状态（**HREADY**为**Low**）。
- **ERROR**
  - 在传输过程中发生了错误（通常情况下是写了只读区域造成的）。
  - 需要**两个HCLK Cycles**去完成。
  - Master可以选择继续发该笔Transfer或者不再发起这次传输。

![image-20240108094513305](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240108094513305.png)

- 当**HRESP**为**0**，**HREADY**为**1**，代表本次传输成功。

- 当**HRESP**为**0**，**HREADY**为**0**。则代表Slave正在反压主机，可能从机此时不能够及时的响应，主机此时应该维持住控制信号。当HREADY拉高，并且HRESP仍然为0，则代表本次传输成功结束。（需要注意的是，对于正常的传输，也就是不涉及错误情况的时候，HRESP应该始终为0）。

  <u>**Note：通常而言，每个Slave应该提前规定好最大的Wait states周期数。此外建议Slave不要插入多于16个周期的等待时间，否则的话每次访问可能花费大量的时间去等待。**</u>

- **当HRESP为1，则代表传输出现错误了。**大概率是写了不让写的地方导致的。对于之前的正常传输而言，只需要维持住一个周期的响应，但是ERROR response需要两个时钟周期。（这是由于总线的流水线特性决定的，当slave发出ERROR响应时，下一个传输的地址已经被广播到总线上了。two-cycle响应给master提供了足够的时间取消下一次访问并将HTRANS [1:0]驱动到IDLE。）

![img](https://pic2.zhimg.com/v2-0949a051de4e97f2ac58ec939f4b3771_r.jpg)

Master终止当前的突发传输的情况：

- T1->T2：Slave插入一个Wait state，即HRADY为0并且将HRESP设置为OKAY。（Slave发现突发写出现错误，因此要准备返回ERROR response，首先要插入一个wait states来准备后续的操作，当然也可以插入多个wait states来准备后续的操作）
- T2->T3：Slave返回一个ERROR response，这是ERROR response的第一个时钟周期，此时HREADY为0。
- T3->T4：Slave返回一个ERROR response，这是ERROR response的第二个时钟周期，此时HREADY为1。
  - Master因此就有足够多的时间去决定下笔是继续发这个transfer还是终止这次transfer，这也就是为什么需要两个时钟周期去维持这个ERROR response，如果没有两个时钟周期的ERROR response，那么T2->T3的HREADY就应该为高，HTRANS应该迅速变成IDLE，来完成这次传输，如果逻辑复杂的话，组合逻辑链路会过长，为了保证流水线特性，让Master有足够多的时间去感知本次传输出错了，则需要维持两拍的ERROR response，此外这也是保证和其他的正常传输一致。回顾一下没有错误传输情况的突发传输，二者都具有流水线特性）。
  - 在这个例子中，Master将HTRANS变成IDLE，代表放弃了这次对地址B的传输。
- T4->T5：Slave返回OKAY response。这次突发传输到此结束。

当然也可以继续传输，如下图所示：虽然突发写出错了。但是Master决定继续突发传输下去。

![img](https://pic3.zhimg.com/80/v2-1dc14e9f657d8062d245c84b00dad346_720w.webp)

If a Subordinate provides an **ERROR** response, then the Manager can cancel the remaining transfers in the burst.  However, this is not a strict requirement and it is acceptable for the Manager to continue the remaining transfers in  the burst. 

A Manager, which receives an **ERROR** response to a **read transfer** might still use the data. A Subordinate  cannot rely on the **ERROR** response to prevent the reading of a value on **HRDATA**. **It is recommended that  HRDATA is driven to zero when an ERROR response is given to a read transfer.**

## **8、其他控制信号**

### 8.1、Protection Control

保护控制信号HPROT[3:0]，每一位代表不同的含义。该信号用于提供一些保护信息。

- HPROT[0]，代表是取数据还是取指令。
- HPROT[1]，代表是特权还是非特权模式访问。比如操作系统的地址区域，就应该是特权访问，并不是所有的写都有权限去写这段地址空间，不然就出大事了。
- HPROT[2]，代表Bufferable。这个Bufferable指的是写的数据是可以放在Buffer里面还是一定要到达最后的目标地址空间。（实际上最终还是会写回最终目标地址空间的，只不过返回HRESP的时候，大概率是还没有写到最终的目标地址空间）对于写具有严格的先后顺序之分的地址空间，一般是Non-cacheable和Non-bufferable，否则可能会乱序。
- HPROT[3]，代表Cacheable，表示是否可以存放在Cache中，是否更新最终的memory。对于外设寄存器这种，当然是不可以Cacheable的。我们本意就是希望写目标寄存器，达到某些功能。写到Cache里面，那就没法达到预期了。

![img](https://pic4.zhimg.com/80/v2-ca4a88c562a4e6fc1db60516d7da04df_720w.webp)

***-------------------------------------NOTE---------------------------------------*-**

**Many Managers are not capable of generating accurate protection information. If a Manager is not capable of  generating accurate protection information, it is recommended that: **

- **The Manager sets HPROT to 4'b0011 to correspond to a Non-cacheable, Non-bufferable, privileged, data  access. **
- **Subordinates do not use HPROT unless necessary.**

-------------------------

除此之外，AHB5（AMBA5的AHB协议）增加了很多额外的控制信号，这些信号我暂时不打算讲。因为后续还会花几篇文章专门介绍完整的AHB协议，这几篇文章大家掌握**AMBA3**的AHB-lite控制信号即可。

## <span id = jump9>**9、AHB和ARM处理器**</span>

AHB本身就是由ARM公司推出的，因此它最早就是用在了ARM系列处理器中，如下图所示：

- AHB-lite总线用在了Cortex-M0，M0+，M3，M4，M7处理器中。用于取指令和数据访问，还用在CoreSight的调试接口上。可以看到这些处理器都是一些低功耗的处理器。AHB-lite协议简单，信号变量较少，对性能要求不高，适用于这些场景。
- AHB2用在了一些早期的ARM处理器中，比如ARM7，实际上这种处理器内核已经没有人使用了，AHB2协议也基本上淘汰了，现在要么是使用AMBA5的AHB5或者AHB-lite或者AMBA3的AHB-lite。（AMBA5和AMBA3的AHB-lite区别很小）。
- AHB5主要用在一些相对性能高的M系列处理器中，比如Cortex-M23，Cortex-M33。这些处理器对保护特性、带锁访问、排他访问等具有要求，因此AHB5的一些信号适用于该场景。

![img](https://pic3.zhimg.com/80/v2-33a6984f5207b57d3ff1fc60aecc047a_720w.webp)

接下来再介绍一下对于某个特定的ARM处理器，它对应的AHB总线协议。

- Cortex-M0/M0+：没用到突发传输，都是Single Transfer。这两款处理器都是非常轻量级的，Buffer也比较少。所以也用不到Burst，不需要源源不断的去取数据。

- Cortex-M3/M4：没有Wrap transaction。只有Single Transfer或者基于INCR的Burst传输。（因为没有Cache，所以没有必要用Wrap transaction）。
  除此之外，还增加了sideband signal用于支持Exclusive Transfer（前面已经提到，这两款处理器使用的是AHB-lite协议，AHB-lite协议里面本身没有Exclusive相关的信号。但是有时候要跑实时操作系统的时候，需要semaphore（信号量）这种机制，设计者可以自行增加这些额外信号）。

- Cortex-M7：有三个AHB-lite接口：AHBS、AHB、AHBP

  ![img](D:\lqh\Typora\图片\v2-01a9165db690715fa96a797910b89360_720w.webp)

- AHBS：AHB-lite Slave。用于让DMA去初始化TCM。TCM全称Tightly Coupled Memory。紧耦合内存，可以理解为On-chip Memory。用于存储一些关键的指令和数据。

- AHBD：AHB-lite Debug。用来给Coresight调试用的。

- AHBP：AHB-lite Peripheral。用来访问外设的，只支持Single Transfer。

- Cortex-M23/M33：
  - M23和M0+很像，增加了TrustZone和整数除法功能，和M0+一样，只支持Single Transfer。
  - M33是M23和M4的缝合产品，支持TrustZone。对于突发传输，它只支持INCR Transfer（依旧没有Cache）。
