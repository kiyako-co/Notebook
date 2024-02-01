# **APB(Advanced Peripheral Bus)总线**

**连接低速且低效率的外设**，如UART，I2C，SPI等。

**广泛用于配置各种IP的寄存器**（这些IP预留用户控制信号，由软件进行配置，这个时候就可以选择使用APB总线来配置这些寄存器）。

**<u>所有信号必须在时钟上升沿进行传递</u>**

![img](https://pic1.zhimg.com/80/v2-5aa04dc4faf0344d015486a64f416b2c_720w.webp)

**所有的APB外设都是APB的从机, APB桥是唯一主机, APB桥就是将上游的数据转换成APB协议发送出去，并接受读到的数据。**

## APB2

| Signal name | Direction | Description                                                  |
| ----------- | --------- | ------------------------------------------------------------ |
| PCLK        | Global    | 时钟信号，上升沿同步                                         |
| PRESETn     | Global    | APB总线复位信号为低有效并且通常将该信号直接连接到系统总线的复位信号 |
| PADDR       | M-->S     | 地址总线，最多可高达32位                                     |
| PSEL        | M-->S     | 选通信号，当该信号拉高意味着要发起一次传输了                 |
| PENABLE     | M-->S     | 使能信号，用于表示一次APB传输的第二个周期（在阻塞情况下为第二个及以上的周期，该信号的存在完全是历史遗留问题，后面详细讲） |
| PWRITE      | M-->S     | 该信号为高标志这次是写传输，反之则为读传输                   |
| PWDATA      | M-->S     | 写数据总线，由Master在写周期进行持续性驱动（PWRITE为高），最高可达32位。 |
| PRDATA      | S-->M     | 读数据总线，由Slave在读周期进行持续性驱动（PWRITE为低），最高可达32位。 |

### **写操作**

![img](https://pic2.zhimg.com/v2-67f90515052a0ebae33235c7c929e365_r.jpg)

- T1->T2属于IDEL状态, APB总线空闲.

- T2->T3这一个周期
  - PSEL信号拉高，意味着要发起一次新的传输了。
  - PWRITE信号为1，因此此次传输为写操作。
  - PWDATA为要传输的数据，PADDR为要写的地址。二者都应该保持不变，直到此次传输结束。（**这些是APB桥输出到APB外设的信号，也就是说apb桥需要在传输过程中让pwdata和paddr保持不变，apb桥锁存上游传来的写数据和写地址**）
  - 而PSEL拉高的第一个时钟周期，PENABLE应该为0。
- T3->T4这一个周期
  - PSEL信号继续拉高
  - PWRITE、PADDR、PWDATA应该保持不变
  - PENABLE信号拉高，用于代表这已经是写传输的第二个周期了

至此一次传输结束。可以看到，APB对每一笔数据的传输，**<u>需要花费两个时钟周期</u>**。且APB的数据传输不支持流水线操作（即不可以重叠）。因此APB是非常低效的。

理论上写数据完全可以同时给出写地址和写数据，一拍就搞定，为什么APB要大费周章弄成两拍呢？这主要是那个年代的芯片本身的制程工艺以及片上互连线导致的。一个周期可能无法完成从Master向Slave写入数据的整个操作流程。因此采用两拍的方式，第一拍告诉你，我要开始传输啦！（称为setup phase）第二拍才真正的完成数据的传输。（称为access phase）。

### **读操作**

![img](https://pic4.zhimg.com/v2-f276f142aa04997d0134638592158cd7_r.jpg)



- T1->T2属于IDEL状态, APB总线空闲.

- T2->T3这一个周期
  - PSEL信号拉高，意味着要发起一次新的传输了。
  - PWRITE信号为0，因此此次传输为读操作。
  - PADDR为要写的地址。应该保持不变，直到此次传输结束。
  - PSEL拉高的第一个时钟周期，PENABLE应该为0。
- T3->T4这一个周期
  - PSEL信号继续拉高
  - PWRITE、PADDR应该保持不变
  - PENABLE信号拉高，PRDATA读取从机对应地址的数据

### **APB Bridge**

APB桥负责产生上面这些信号，然后传递给APB外设。

- **IDLE**：此时为默认状态。PSEL和PENABLE都为0，没有通信请求。
- **SETUP**：当需要发起传输的时候，会进入该状态，此时PSEL为1，PENABLE为0。PSEL从0到1说明要发起一次传输了，而PENABLE为0表示这是传输的第一个时钟周期。（这个状态也就是setup phase），**<u>APB桥在该阶段锁存上游传来的读写控制，读写地址和写数据</u>**。
- **ENABLE**：这个状态PENABLE需要拉高，进而完成数据的传输。

```verilog
module apb2_bridge(
    // system signal
    input   wire    pclk_i,
    input   wire    prst_n_i,

    // system bus slave interface, to apb 
    input   wire    sys_bus_vld,    
    input   wire    sys_bus_write,  // 1 -> write 0 ->read
    input   wire    [31:0]sys_bus_addr,
    input   reg     [31:0]sys_bus_wdata,
    output  reg     [31:0]sys_bus_rdata,

    // apb interface
    output   reg    [31:0]PADDR,
    output   reg    PSEL,
    output   reg    PENABLE,
    output   reg    PWRITE,
    input    wire  [31 : 0]PRDATA,
    output   reg   [31 : 0]PWDATA
);
    
    // FSM
    parameter IDEL = 2'b00;
    parameter SETUP = 2'b01; //地址阶段
    parameter ENABLE = 2'b11; // 数据阶段

    reg [1:0]current_state;
    reg [1:0]next_state;

    // FSM 
    always @(posedge pclk_i or negedge prst_n_i)begin
        if(!prst_n_i)begin
            current_state <= IDEL;
        end
        else 
            current_state <= next_state;
    end

    always @(*)begin
        case(current_state)
            IDEL:begin
                if(sys_bus_vld)
                    next_state = SETUP;
                else 
                    next_state = IDEL;
            end
            SETUP:begin
                next_state = ENABLE;
            end
            ENABLE:begin
                if(sys_bus_vld)begin
                    next_state = SETUP;
                end
                else next_state = IDEL;
            end
            default: next_state = IDEL;
        endcase
    end
    
    always@(posedge pclk_i or negedge prst_n_i)begin
        if(!prst_n_i)begin
            PSEL <= 0;
            PENABLE <= 0;
            PWRITE <= 0;
            PWDATA <= 0;
        end
        else if(next_state == IDEL)begin
            PSEL <= 0;
            PENABLE <= 0;
        end
        else if(next_state == SETUP)begin
            PADDR <= sys_bus_addr;
            PWDATA <= sys_bus_wdata;
            PWRITE <= sys_bus_write;
            PSEL <= 1;
            PENABLE <= 0;
        end
        else if(next_state == ENABLE)begin
            PENABLE <= 1;
            if(!PWRITE)
                sys_bus_rdata <= PRDATA;
        end
    end

endmodule

```

<u>**对于一些纯组合逻辑的电路，PENABLE很有用。可以作为读写的控制信号。**</u>

## **APB3**

比APB2多了2个信号。

| Signal name | Direction | Description                                                  |
| ----------- | --------- | ------------------------------------------------------------ |
| PREADY      | S-->M     | Ready. The slave uses this signal to extend an APB transfer.从机反压主机，PREADY为0时说明从机还没准备好读写数据，此时主机需要等待。 |
| PRESETn     | S-->M     | This signal indicates a transfer failure. APB peripherals are not required to support the PSLVERR pin. This is true for both existing and new APB peripheral designs. Where a peripheral does not include this pin then the appropriate input to the APB bridge is tied LOW. |

### **写操作**

![img](https://pic3.zhimg.com/v2-16fe5885776da6a3880fd321c9d5c2ea_r.jpg)

**整个写传输过程需要在PSEL & PWRITE & PENABLE & PREADY才算完成**

![img](https://pic1.zhimg.com/v2-9d8d20efbe83c0bc82f96e1992d0ba08_r.jpg)

如果PREADY没有准备好，PSEL、 PWRITE 、 PENABLE、PADDR、PWDATA需要保持不变 ，也就是保持在ENABLE状态。

### **读操作**

![img](https://pic2.zhimg.com/v2-711b12415185ced16b0f7179295a3719_r.jpg)

直到PREADY为高时，PRDATA接受从机数据。

### **Error response**

![img](https://pic3.zhimg.com/v2-016185479683f844f052f8255766d146_r.jpg)

以写为例，实际上就是在真正发起传输的那一拍（也就是PREADY拉高的那一拍）顺势拉高PSLVERR，用于标志这次传输失败。PSLVERR信号是从机产生的，具体怎样产生由从机决定，如果从机不支持判断传输错误，那么该信号拉低即可。

### **APB Bridge**

![image-20240103105537001](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240103105537001.png)

## **APB4**

比APB3多2个信号。

| Signal name | Direction | Description                                                  |
| ----------- | --------- | ------------------------------------------------------------ |
| PPROT       | M-->S     | Protection type. This signal indicates the normal, privileged, or secure protection level of the transaction and whether the transaction is a data access or an instruction access. |
| PSTRB       | M-->S     | Write strobes. This signal indicates which byte lanes to update during a write transfer. There is one write strobe for each eight bits of the write data bus. Therefore, PSTRB[n] corresponds to PWDATA[(8n + 7):(8n)]. Write strobes must not be active during a read transfer. |

![image-20240131190117659](F:\笔记\assets\image-20240131190117659.png)

![image-20240131190141161](F:\笔记\assets\image-20240131190141161.png)

## APB slave  mux

![img](F:\笔记\assets\v2-7f9596fde8c938ec2964b02527d38045_r.jpg)

- 根据PSEL是否有效以及DECODE4BIT的值，完成16选1,PSEL0~PSEL15有一个或者0个拉高。
- PREADYm默认为1，当PSEL为1的时候，根据译码结果选择相应的PREADY信号（当端口没有使能的时候`en[x] == 0`， 对应的`PREADYx`信号不会被选择）。
- PSLVERR和PRDATA，选中谁就取谁的。

```verilog
assign PSEL0   = PSEL & dec[ 0] & en[ 0];
assign PSEL1   = PSEL & dec[ 1] & en[ 1];
assign PSEL2   = PSEL & dec[ 2] & en[ 2];
//省略3~15
assign PREADY  = ~PSEL |
                   ( dec[ 0]  & (PREADY0  | ~en[ 0]) ) |
                   ( dec[ 1]  & (PREADY1  | ~en[ 1]) ) |
                   ( dec[ 2]  & (PREADY2  | ~en[ 2]) ) |
//省略3~15
assign PSLVERR = ( PSEL0  & PSLVERR0  ) |
                   ( PSEL1  & PSLVERR1  ) |
                   ( PSEL2  & PSLVERR2  ) |
//省略3~15
assign PRDATA  = ( {32{PSEL0 }} & PRDATA0  ) |
                   ( {32{PSEL1 }} & PRDATA1  ) |
                   ( {32{PSEL2 }} & PRDATA2  ) |
//省略3~15
```

这套代码的缺点或者说优点是，它是用组合逻辑做的，逻辑非常的简单。实际上就是多选1，一般来说APB的时钟频率很低，所以增加了一定的组合逻辑级数也不会出现时钟违例。这样做还可以节省一个时钟周期，大家也可以用时序逻辑去做，思路是类似的。还有一个问题就是这个模块没有PENABLE信号，这个其实挺致命，大家可以手动加上，跟PSEL的逻辑基本是一模一样的，非常简单。

## **APB interconnect**

![img](https://pic4.zhimg.com/v2-00ca6a99d8df444a5e4feeea13581393_r.jpg)

对于Master-->Slave的信号而言，**PENABLE、PWRITE、PWDATA、PADDR**信号直接由Master给所有的Slave。而PSELx信号有SLave数量这么多组，其逻辑应该是

```verilog
PSELx = PSEL & dec[ x] & en[ x]
```

译码器根据PADDR选择拉高某个SLAVE的dec信号，也就是最多选中其中的某一个Slave。此外下面这个图中有一个默认slave，当没有任何slave被选择的话，则会选中默认的slave，用来应对地址越界的错误情况。该Slave默认的PRDATA默认为0，PSLVERR默认为1。

对于Slave-->Master的信号而言，PREADY、PRDATA、PSLVERR由MUX进行选择，从指定的Slave传给Master。

![img](https://pic3.zhimg.com/v2-57ee952ebc71309cc23ced7d37b6029a_r.jpg)

译码器模块对每一笔传输进行地址译码，给每一个Slave相应的PSEL信号。Decode逻辑非常简单，就是根据当前的PADDR选中某一个Slave，如下图所示。这个模块功能更加丰富，地址映射可以静态的配置（工作的时候不能配置），实际上大部分的SoC设计中，地址映射应该是完全固定死的，无法更改的。这个是FPGA提供的IP，所以相对更灵活一点。

![img](https://pic2.zhimg.com/v2-580489d5771d06486b6cadc9718944a9_r.jpg)

该模块基于MUX提供的PSELX信号，从多个Slave中选择合适的PRDATA、PREADY、PSLVERR信号，设计很简单，就不多讲解了，直接看图：

![img](https://pic2.zhimg.com/v2-f275e9dfbb629725d1f3ae0c00bc4c31_r.jpg)

## **多主多从的APB interconnect**

![img](https://pic4.zhimg.com/v2-6f2442d3ee9629378a5ebfd86c099f57_r.jpg)

Arbiter+MUX。Master to Slave Multiplexor对多个PSEL进行仲裁，然后选择其中的一个PSELX，基于这个PSELX，选择合适的PADDR、PWRITE、PSELX、PENABLE给Single Master Interconnect模块。

相应的，Slave to Master Multiplexor将PRDATA、PREADY、PSLVERR路由到相对应的Master。从而完成整个传输流程。下图这些向右的箭头实际上是双向的。Master和Slave互相交互，完成传输过程。

![img](https://pic4.zhimg.com/v2-01565b549020c085bdd8b97f9185469f_r.jpg)