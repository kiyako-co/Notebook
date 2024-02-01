# 1、AHB Bus Matrix

前面我们的内容，主要讲的都是Master和Slave。可是Master和Slave之间如何连接呢？通过点对点的方式当然可以实现连接，但是如果有多个Master多个Slave呢？普通的点对点或者slave mux的方式可能无法满足我们的要求，这个时候我们就需要新工具了。

**AHB Bus Matrix，即总线矩阵**，其实际上就是一个**互连（Interconnect）**。用于连接满足该总线协议的外设，包括Master和Slave。基于该模块，我们可以快速的完成“连连看”工作。将设计好的IP封装成AHB协议，然后挂载上去即可。这样就完成了简单的SoC集成工作。在集创赛ARM杯中，很多项目其实都是基于这种方式工作。**将设计好的IP挂载成Slave，然后我们通过上位机给CPU编程，CPU随机会通过总线下发命令给相应的IP，完成工作以后产生中断，然后去拿运算完成的结果进行进一步的运算即可**。很多低端的MCU都是这种工作方式，一般Master主要有两种，一个CPU Core，还有一个DMA用来搬数。

![img](D:\lqh\Typora\图片\v2-30f5abd857ec988abdbb30f14510ba5c_720w.webp)

这个AHB Bus Matrix大家可以手写，但是这样的话工作量较大，并且AHB协议实际上写起来，还是很容易出错的，ARM官方实际上提供了一套工具，可以编写XML文件，通过脚本即可生成我们需要的矩阵。具体的我就不介绍了，大家可以看下面这个视频，讲的很好，并且足够详细了。我当时就是看这个视频，学会了如何生成我需要的总线矩阵，然后将我自己设计的IP挂载了上去，实现了一个简单的SoC。

[Cortex-M3的FPGA实现 - 使用CMSDK搭建Cortex-M3_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV127411P7W9/?p=2&vd_source=aa30a4e19bdd942dc58f16b3be44358e)

## 1.1、使用官方脚本生成AHB Bus Matrix

文件结构：

**cmsdk_ahb_busmatrix**

- bin/BuildBusMatrix.pl	AHB Bus Matrix生成脚本
- verilog/src            AHB Bus Matrix源文件，不要直接使用这些文件
- verilog/built         配置完成后，AHB Bus Matrix.v生成在这个文件夹
- xml      脚本控制文件，在该文件夹下有2个问价，一个提供了稀疏连接矩阵的范例，另一个则是全连接的矩阵。修改文件中的连接关系可以控制生成的AHB Bus Matrxi。

**cmsdk_mcu_mtx4x2**

- 官方提供的2主机4从机稀疏矩阵示例连接。

## 1.2、设计流程

首先，画出AHB Bus的连接关系。

![image-20240115092639788](D:\lqh\Typora\图片\image-20240115092639788.png)

-------------**NOTE**--------------

**S0~S4指的是主机的从机，比如CPU，DMA这些Master的从机，所以有几个Sx就代表有多少个主机。**

**M0~M4指的是外设的从机，有几个Mx就有多少个外设。**

**实际的指向关系是S->M。也可以把S看成是Master端的输出，M看出是Slave端的输入。**

----------------------------------



然后修改.xml文件。

![image-20240115100917721](D:\lqh\Typora\图片\image-20240115100917721.png)

Arbitration scheme:

- **burst** 	 - Fixed priority;	 Master 0 has highest priority, does not break defined length bursts  
- **fixed** 	  - Fixed priority; 	Master 0 hashighest priority. 
- **round**	 - Round robin priority; 	priority goes to next available master.

![image-20240115102317799](D:\lqh\Typora\图片\image-20240115102317799.png)

从机连接关系：指定与从机相连的主机，并设置从机在主机中的映射地址。

![image-20240115102521141](D:\lqh\Typora\图片\image-20240115102521141.png)

定义外设的主机。

使用perl脚本生成最终的AHB Bus Matrix，最终文件在build文件夹。

```shell
bin/BuildBusMatrix.pl -xmldir ../cmsdk_mcu_mtx4x2/xml/ -cfg cmsdk_mcu_4x2.xml -over -verbose
```

![image-20240115102119309](D:\lqh\Typora\图片\image-20240115102119309.png)

## **1.3、代码示例**

```verilog
reg  [1:0] addr_in_port_next; // 组合逻辑结果
reg  [1:0] iaddr_in_port;    // 寄存器输出
reg        no_port_next;     // 组合逻辑结果
// 下面代码是fixed priority格式的仲裁，也就是req_port 0的优先级最高，然后依次降低。如果没有输入请求，则保持当前的port，如果始终没有port请求，则拉高no_port。
always @ (
             req_port0 or
             req_port2 or
             req_port3 or
             HSELM or HTRANSM or HMASTLOCKM or iaddr_in_port
           )

  begin : p_sel_port_comb
    // Default values are used for addr_in_port_next and no_port_next
    no_port_next     = 1'b0;
    addr_in_port_next = iaddr_in_port;

      if (HMASTLOCKM) //如果是原子操作，则当前主机传输时，不允许其他主机打断传输
      	addr_in_port_next = iaddr_in_port;
      else if ( req_port0 | ( (iaddr_in_port == 2'b00) & HSELM & (HTRANSM != 2'b00) ) ) //没有新的传输插手，就保持当前的port
      	addr_in_port_next = 2'b00;
      else if ( req_port2 | ( (iaddr_in_port == 2'b10) & HSELM & (HTRANSM != 2'b00) ) )
      	addr_in_port_next = 2'b10;
      else if ( req_port3 | ( (iaddr_in_port == 2'b11) & HSELM & (HTRANSM != 2'b00) ) )
      	addr_in_port_next = 2'b11;
      else if (HSELM)
      	addr_in_port_next = iaddr_in_port;
      else
      	no_port_next = 1'b1;
  end // block: p_sel_port_comb
```

# 2、AHB的局限性

实际上现在的SoC设计，AHB已经很少使用了，除非是很低端的MCU，考虑到节省功耗和面积等因素，才会去使用AHB总线。目前AXI已经使用的越来越多了。

我们看一下下面这幅图，和之前讲过的典型AMBA系统的SoC基本是一模一样的，只不过把AHB换成了AXI，这样做有什么好处呢？下面就给大家进行一个简单的分析。

![img](D:\lqh\Typora\图片\v2-59524099e524d54ba16279b41be81a38_720w.webp)

AHB协议本身是ARM公司在AMBA2这一代推出的协议，距今已经有20多年了，该协议本身存在着很多历史局限性，用今天的视角来看，AHB属于高不成低不就的一代协议。在低频低性能要求的情况下，论功耗面积等开销比不过APB协议。在高频高性能要求下，能跑到的主频和带宽又比不过AXI。所以AHB目前一般只用在IP本身比较旧，用的还是AHB协议的Wrapper，然后又不想改的情况下，才会考虑使用。不然一般会直接上AXI协议。

此外AHB协议本身限制要求较高，比如command和data必须是1Cycle的延迟，error response，HREADYOUT和HREADYIN等机制都很容易导致设计出错，这是从设计层面去考虑AHB存在的问题。

我们分析一下AHB的带宽，在理想情况下，AHB的最大带宽为 
$$
BW = Freq * DW
$$
虽然AHB存在WDATA和RDATA，但是它的控制信号只有一路，并且COMMAND和DATA必须满足1T的延迟，**所以WDATA和RDATA是无法同时工作的。等于数据信号的利用率最多只有50%**。可以认为它近似于一个**半双工**的BUS。

![img](D:\lqh\Typora\图片\v2-f8ad33686429d7453f5ade6e5bf8e476_720w.webp)

上述还不是最坏的情况，因为我们考虑数据是一拍回复，那当数据不是一拍回复，并且当我们需要改变传输方向的时候呢？比如我们先读后写。在读数据回来之前我们是不准写的，这个前面讲AHB协议的时候有说到，因为HREADY会反压，一直到读数据回来的时候才准写。**因此当回数据需要很多拍的时候，并且频繁地发生读写转换的时候，AHB的效率是非常低的**！

![img](D:\lqh\Typora\图片\v2-111031497f881636e1bd9f506da4128c_720w.webp)

AXI相比AHB**最重要的改变**就是**读写通道分离**，相当于我们有两路可以传输了（全双工），此外AXI还支持Outstanding（现在简单理解为在路上即可）。同样考虑开始开车的例子，同样是读写频繁转换，传输延迟很大为100s，在理想的情况下可以是2car/1s！。因为AXI不需要等读回数据响应再发送下一次传输，它支持“在路上”。

![img](D:\lqh\Typora\图片\v2-85b7cb35b5dab6eee0d180d26e4c5845_720w.webp)

除此之外，AXI引入了握手的机制，不需要data必须在command的1T以后，其实多少T以后都可以。因此我们可以跑到很高的频率，频率不满足要求加周期就行了。这也是AXI的一个优势所在，在普通频率或者很高的频率，都有它的用武之处。