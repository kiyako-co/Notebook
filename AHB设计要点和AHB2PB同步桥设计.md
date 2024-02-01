# 1、AHB设计要点

## **1.1、HREADYOUT和HREADY**IN区别

很多人包括我自己在第一次接触AHB的时候，会被它的两个HREADY信号搞混，分别是HREADYOUT和HREADY(IN)信号。

![img](https://pic4.zhimg.com/v2-dae9c811120fb886aa1138bcf702926b_r.jpg)

其中，**HREADYOUT**信号是由Slave发出，送给MUX选择器，用来表示Slave是否已经准备好进真正的数据传输（data phase阶段），例如写操作时，Slave是否已经将数据存下来，本质上是Slave对Master的反压信号。

而**HREADY(IN)**是由MUX输出返回到所有的Slave，通知各个Slave，是否还有别的Slave有未完成的传输。每个Slave在采样地址和控制信号的时候，都需要看这个HREADYIN信号是否为1，如果为0的话则代表别的Slave还有未完成的传输，因此不能采样地址和控制信号。

**首先我们思考一个问题，HREADYOUT默认的复位值应该是多少?**

- 假设都为0，那么HREADY信号自然也为0。这样返回给所有的Slave的HREADYIN信号就为0，则Slave会认为还存在别的transaction还没有结束。则会一直等待，那么整个系统就会不工作了！这也是很多人设计AHB-Slave的时候会犯的错误。

**然后我们再思考一下，如果没有HREADYIN信号会怎么样？**

假设此时有两个Slave，Slave1和Slave2。没有HREADYIN，则对Slave自身而言。没有人去反压它自己，也就意味着只要HSEL选中它的情况下，它都可以接收控制信号。

假设在T0->T1这个时间段，S1接收了一笔控制信号（ADDR PHASE）。按理来说T1->T2它应该要返回数据或者将写数据存下来（DATA PHASE），但如果S1不能接收，在T1->T2这个时间段S1的HREADYOUT为0，同时T1->T2又进入了新的addr phase，假设此时Master选中的是S2，控制信号什么的都发生了变化。那么请问Master在T2->T3这个阶段应该怎么做？

- **如果Master是看S1**，那么S1的HREADYOUT为0，则MASTER会维持住当前的控制信号。但是T2->T3的时候，实际上选中的是S2，那么S2就会接收两笔控制信号。并且还是一模一样的控制信号，如果都是NONSEQ信号，地址还一样，则直接不符合协议报错。
- **如果Master是看S2**，此时S2的HREADYOUT为1，那么Master会变化控制信号和写数据。那么T2->T3的时候写数据变了，则等到S1的hreadyout拉高的时候，它拿不到正确的数据（此时还是针对S1的DATA PHASE）。

![img](https://pic4.zhimg.com/80/v2-cf7e573345524776d5db33ec0e1f93ff_720w.webp)

![LF4`H(F%N0H)ID1H044L8`0](C:\Users\admin\AppData\Roaming\Tencent\QQ\Temp\LF4`H(F%N0H)ID1H044L8`0.png)

其实核心的逻辑在于，如果没有HREADYIN信号，那么HSEL选中的情况下，Slave就可以接收控制信息和数据信息。这个时候Master无论选择哪个Slave的HREADY，都有可能导致其它的Slave接收到错误的信息。还有一点导致这个原因就是AHB具有pipeline传输，当前的CYCLE有可能既是上一个Slave的data pahse也是下一个Slave的addr phase。

因此如果只靠Slave反压master，选择哪个Slave反压Master就成了问题，这样一来，不管选择哪个Slave都会出问题。

## 1.2、Memory Mapping

AHB slave最小是按照1KB对齐的，也就是译码的结果，只会看1KB以上的地址。

如果没有做好1KB对齐，这时候HSEL信号会同时选择Slave 1和Default Slave。这时候自然就不符合预期了。所以应该按照协议的规则，做好1KB对齐。

![img](https://pic2.zhimg.com/80/v2-c59836b4259dbc85d0cd2ed40d4b3541_720w.webp)

Default Slave是用来干嘛的？我们做Memory Map的时候，自然不可能所有的地址都覆盖掉。我们需要考虑有一个Default Slave，这样访问缺省地址的时候，会有一个符合你需求的Response。如果没有相关逻辑的话，就会有问题，总线会卡住。

![img](https://pic1.zhimg.com/v2-3b3445676f0730893f5529efc8f503f4_r.jpg)

# 2、AHB2APB同步桥设计

![img](https://pic2.zhimg.com/80/v2-6f2d7aa5458897fa8d576f53a75e8e8d_720w.webp)

我们再看一下这个典型的基于AMBA的SoC子系统，左右两边分别是AHB和APB子系统。两边子系统的时钟频率通常不一样。所以我们才需要两个子系统，那如何建立二者之间的通信。

这个时候就需要AHB2APB转接桥实现协议的转换，实际上在SoC设计当中，转接桥是非常常见的。转接桥的设计很大程度上决定了系统的工作能力，做好转接桥的设计需要对转接桥两端的协议都有充分的理解，此外当时钟频率不一样的时候，还会涉及跨时钟域的知识。接下来的文章将通过一系列文章给大家讲解AHB2APB转接桥的设计和相关的跨时钟域的知识。

本篇文章给大家讲解**AHB2APB同步桥（或者叫转接桥）**的设计。转接桥在SoC设计中属于比较重要的一环，因为SoC通常使用多种不同的总线协议，这些不同的总线协议之间想要完成通信，就需要转接桥的帮助。

Bridge，顾名思义，桥梁，用于完成两者之间的通信。本篇文章的转接桥是AHB高性能总线到APB总线桥接器，主要完成以下的功能：

- Bridge是APB总线中**唯一主机**；
- APB的时钟和AHB的时钟是**同步时钟**；APB的时钟和AHB时钟的分频关系由PCLKEN信号决定；
- 该模块支持输入输出数据寄存或者不寄存，由模块参数控制；
- 该同步桥支持APB总线字节选通信号，保护控制信号；
- 该同步桥支持APB模块的使能信号；
- 该同步桥只有一个PSEL，其他从设备的PSELx由额外的译码电路根据地址产生；

## 2.1、转接桥接口、时序

![img](https://pic4.zhimg.com/80/v2-d48973dfdba60aaa12f92fd795723d87_720w.webp)

其接口如下图所示，可以看到其完成了AHB协议到APB协议的转换，并且是单对单的，即连接一个AHB主设备、一个APB从设备。

接下来我们看一下设计思路：

**首先是第一种情况**，输入输出数据不寄存且不产生错误的情况下。这种情况下其实和之前讲的APB状态机基本是一样的。大家想想看，AHB的通信和APB实际上本来就是差不多的，不过AHB可以支持流水，APB不支持流水。此外AHB可以突发传输，APB不可以突发传输。对于突发上述模块直接没有相应接口，对于输入是流水的情况，实际上会反压前级，只能按照APB的方式进行传输。因此转接桥和APB协议的状态机一样，总共可以分为三个状态：

- **IDLE**：空闲状态；外设总线默认在此状态；
- **ST_APB_TRNF**：传输建立状态；在此状态，将PSEL置为1，并保持到ST_APB_TRANF2状态（即对应setup phase）
- **ST_APB_TRNF2**：传输状态；在此状态。传输完成时，需要将PENABLE置为1；

![img](https://pic1.zhimg.com/v2-2b7fb391ae2be0d8eb1c6c40c899ba98_r.jpg)

**接下来我们考虑第二种情况**，即输入输出数据需要寄存。同时也没有发生错误。（其实大部分情况下都是这种传输方式，因为APB那边很有可能不能够接收数据，因此就需要先将控制信号寄存好）这种情况下，需要增加两个状态，即输入数据需要寄存，输出数据也需要寄存，总共可以分为以下五种状态：

- IDLE：空闲状态；外设总线默认在此状态；
- ST_APB_WAIT：传输等待状态，等待输入数据寄存1拍；
- ST_APB_TRNF：传输建立状态，在此状态，将PSEL置为1，并且保持到ST_APB_TRNF2状态；
- ST_APB_TRNF2：传输状态；在此状态，传输完成的时候，将PENABLE置为1；
- ST_APB_ENDOK：传输结束状态；因为输出寄存1拍，所以传输完成最后1拍需要继续保持传输状态，给出传输信号。

![img](https://pic3.zhimg.com/v2-1674b0851d4d6ab989932dfaee1c3842_r.jpg)

**<u>NOTE：这个图的状态转移条件8和9反了。</u>**

最后我们考虑第三种情况，即传输错误的情况。因为这种情况下需要通知AHB主机那边，而AHB的错误响应本身就有两拍，因此这种情况下又需要两个状态，用来准备错误异常处理。这里直接贴图：

![img](https://pic2.zhimg.com/v2-e36af69edc47e342a3ff4cf5eab9c2d1_r.jpg)

## **2.2、转接桥代码**

代码为ARM开源代码：cortexm0ds\logical\cmsdk_ahb_to_apb\verilog

首先是参数，这里指定位宽为16bit，对读数据进行缓存，不对写数据进行缓存。

```verilog
module cmsdk_ahb_to_apb #(
  // Parameter to define address width
  // 16 = 2^16 = 64KB APB address space
  parameter     ADDRWIDTH = 16,
  parameter     REGISTER_RDATA = 1, 
  parameter     REGISTER_WDATA = 0)
```

接下来我们看一下状态机，这里和我们前面的讲解一致。

```verilog
   localparam [ST_BITS-1:0] ST_IDLE      = 3'b000; // Idle waiting for transaction
   localparam [ST_BITS-1:0] ST_APB_WAIT  = 3'b001; // Wait APB transfer
   localparam [ST_BITS-1:0] ST_APB_TRNF  = 3'b010; // Start APB transfer
   localparam [ST_BITS-1:0] ST_APB_TRNF2 = 3'b011; // Second APB transfer cycle
   localparam [ST_BITS-1:0] ST_APB_ENDOK = 3'b100; // Ending cycle for OKAY
   localparam [ST_BITS-1:0] ST_APB_ERR1  = 3'b101; // First cycle for Error response
   localparam [ST_BITS-1:0] ST_APB_ERR2  = 3'b110; // Second cycle for Error response
   localparam [ST_BITS-1:0] ST_ILLEGAL   = 3'b111; // Illegal state
```

然后我们看关键的控制信号：

reg_rdata_cfg和reg_wdata_cfg代表读数据或者写数据是否需要寄存;
apb_select代表APB bridge select信号，当HSEL拉高，代表AHB主机要发送数据了，HREADY为高代表没有Slave有未完成的传输，可以采样控制信号，而HTRANS[1]为高代表为NONSEQ或者SEQ传输。
在select信号有效的时候，代表要开始数据传输了，这个时候需要把控制信号给锁存下（注意锁存的都是控制信号，没有锁存wdata，这一拍对应addr cycle）。

```verilog
 // Configuration signal
  assign reg_rdata_cfg = (REGISTER_RDATA==0) ? 1'b0 : 1'b1;
  assign reg_wdata_cfg = (REGISTER_WDATA==0) ? 1'b0 : 1'b1;

  // Generate APB bridge select
  assign apb_select = HSEL & HTRANS[1] & HREADY;
  // Generate APB transfer ended
  assign apb_tran_end = (state_reg==3'b011) & PREADY;

  // Sample control signals
  always @(posedge HCLK or negedge HRESETn)
  begin
  if (~HRESETn)
    begin
    addr_reg  <= {(ADDRWIDTH-2){1'b0}};
    wr_reg    <= 1'b0;
    pprot_reg <= {2{1'b0}};
    pstrb_reg <= {4{1'b0}};
    end
  else if (apb_select) // Capture transfer information at the end of AHB address phase
    begin
    addr_reg  <= HADDR[ADDRWIDTH-1:2];
    wr_reg    <= HWRITE;
    pprot_reg <= pprot_nxt;
    pstrb_reg <= pstrb_nxt;
    end
  end
```

**<u>addr_reg取的是HADDR的[ADDRWIDTH-1 : 2]，因为AHB有HSZIE，低2位可以变化，而APB没有PSIZE，APB的地址只能按word变化,每次加4，然后APB可以根据PSTRB来选择要写的数据，</u>**

**<u>比如AHB的HSIZE=00，则每次AHB的地址+1，写数据是字节，APB的地址是AHB的高ADDRWIDTH-2，AHB地址在+1的时候，对应的APB地址其实是没有变化的，变得是传过来的数据，而AHB的DATA也随着HSIZE和地址变化位置的(看AHB-lite总线.md里的Transfer size小节)，APB根据PSTRB信号选择数据。</u>**

然后我们看一下输出信号是怎么赋值的，我们可以看到。PADDR、PWRITE、PPROT和PSTRB这些控制信号，都是寄存的数据。而PWDATA是直接来自于HWDATA（不使用写数据锁存的方式）。然后PSEL和PENABLE由当前状态决定。

```verilog
  // Connect outputs to top level
  assign PADDR   = {addr_reg, 2'b00}; // from sample register
  assign PWRITE  = wr_reg;            // from sample register
  // From sample register or from HWDATA directly
  assign PWDATA  = (reg_wdata_cfg) ? rwdata_reg : HWDATA;
  assign PSEL    = (state_reg==ST_APB_TRNF) | (state_reg==ST_APB_TRNF2);
  assign PENABLE = (state_reg==ST_APB_TRNF2);
  assign PPROT   = {pprot_reg[1], 1'b0, pprot_reg[0]};
  assign PSTRB   = pstrb_reg[3:0];
```

综上，就完成了AHB2APB转接桥的设计，实际上很简单。因为AHB和APB本身是很像的。这种状态机的转换方式相对效率较低，需要多个周期才能完成。但是可以有效反压，避免丢数据。大家可以仔细研究一下。

