# AXI2SRAM设计

## 1、AXI2MEM简介

本模块**是ETH开源项目PULP**的一个小模块，其完整代码链接如下：[axi_mem_if/src/axi2mem.sv at master · pulp-platform/axi_mem_if · GitHub](https://github.com/pulp-platform/axi_mem_if/blob/master/src/axi2mem.sv)

该模块的作用，是将AXI协议转换为SRAM的读写时序，就比如在下图这个例子中，Native接口的SRAM是无法直接和AXI的接口直接相连的，我们必须借助这种协议转换模块，完成两种协议之间的通信。![img](D:\lqh\Typora\图片\v2-d6afda33fd4584d8611e3e92776875aa_720w.webp)

我们设计的模块接口非常简单，一边是AXI矩阵传来的AXI接口，另外一边是SRAM的native接口，如下图所示。![img](D:\lqh\Typora\图片\v2-d98187be0081a543338bc0b5bf789dc9_720w.webp)

## 2、AXI2MEM设计思路及代码讲解

我们设计的AHB2MEM规格说明如下：

- 不支持Outstanding，阻塞型读写
- 我们连接的SRAM是单端口SRAM，因此我们的AXI2MEM不支持也不必支持同时读写
- 支持突发读写、不跨4KB边界
- 对于wrapping burst，burst length必须是2、4、8、16
- 一旦突发读写开始，不允许中途停止

### 2.1、READ操作

读操作有读地址通道和读数据通道，先对读地址通道进行握手，关键信号如下：

```verilog
  // - ARVALID : M -> S  ARREADY : S -> M
  // - RVALID : S -> M   RREADY : M -> S

```

首先是IDLE状态，AXI主机发送地址VALID信号，AXI2SRAM模块检测到AXI的ARVALID信号后，拉高ARREADY信号表示从机准备好接收地址信息，此时对控制信号进行采样锁存，同时跳转到READ状态，进行读数据通道握手。（xxx_d是组合逻辑中采到的值，xxx_q是对应的寄存器输出值）

```verilog
case (state_q)

    IDLE: begin
        // Wait for a read or write
        // ------------
        // Read
        // ------------
        if (slave.ar_valid) begin
            slave.ar_ready = 1'b1;
            // sample ax
            ax_req_d       = {slave.ar_id, slave.ar_addr, slave.ar_len, slave.ar_size, slave.ar_burst};
            state_d        = READ;
            //  we can request the first address, this saves us time
            req_o          = 1'b1;
            addr_o         = slave.ar_addr;
            // save the address
            req_addr_d     = slave.ar_addr;
            // save the ar_len
            cnt_d          = 1;
```

然后是READ状态。该状态下，先利用前面采到的控制信号判断突发传输模式，是FIXED还是INCR还是WRAP模式。如果是FIXED模式AXLEN=0，说明burst传输中只有一个transfer，如果是INCR和WRAP模式，那么只需要给出一个地址，然后地址递增。地址计算方法如下：

```verilog
localparam LOG_NR_BYTES = $clog2(AXI_DATA_WIDTH/8);

aligned_address = {ax_req_q.addr[AXI_ADDR_WIDTH-1:LOG_NR_BYTES], {{LOG_NR_BYTES}{1'b0}}};

wrap_boundary = get_wrap_boundary(ax_req_q.addr, ax_req_q.len);

// this will overflow
upper_wrap_boundary = wrap_boundary + ((ax_req_q.len + 1) << LOG_NR_BYTES);

// calculate consecutive address
cons_addr = aligned_address + (cnt_q << LOG_NR_BYTES);
```

INCR模式下，先对AXI传过来的地址进行对齐操作，对齐后的地址基础上每transfer增加LOG_NR_BYTES，cnt_q对负责对transfer进行计数，计满到AXLEN时说明传输到最后一笔transfer，cnt_q << LOG_NR_BYTES就是乘LOG_NR_BYTES，比如32bit的数据，那么地址每次增加4，cntq=2也就是第二次传输，此时在对齐的地址基础上加8即为当前transfer的地址。

WRAP模式下，也是先对AXI传过来的地址进行对齐操作，同时计算WRAP的边界，上边界等于对齐的地址 + （1+AXLEN) * (1 << AXSIZE)，其中AXLEN说明burst有多少笔传输，WRAP模式下burst规定必须是2，4，8，16。AXSIZE说明单个transfer下有多个字节有效，所以可以计算出burst的操作边界，对应这cacheline中的一个块。如果地址到了WRAP的上边界，但是计数没有到AXLEN，也就是burst还没传完，此时地址回到下边界wrap_boundary继续传输。

```verilog
case (ax_req_q.burst)
    FIXED, INCR: addr_o = cons_addr;
    WRAP:  begin
        // check if the address reached warp boundary
        if (cons_addr == upper_wrap_boundary) begin
            addr_o = wrap_boundary;
            // address warped beyond boundary
        end else if (cons_addr > upper_wrap_boundary) begin
            addr_o = ax_req_q.addr + ((cnt_q - ax_req_q.len) << LOG_NR_BYTES);
            // we are still in the incremental regime
        end else begin
            addr_o = cons_addr;
        end
    end
endcase
```

$$
WRAP\_Boundray = (INT(Start\_address / (Number\_bytes * Burst\_Length))) / (Number\_bytes * Burst\_Length)
$$

WRAP的下边界是对brust的整个字节数进行对齐，也就是单个transfer的字节数 x burst的transfer数量。==**<u>因为余数一定小于除数，所以对齐操作可以转变为对低位置0。</u>**== 

除数的二进制位宽等于多少，那么最低几位就直接取0，只取前面高位，此时得到的地址就是对齐的地址。

```verilog
function automatic logic [AXI_ADDR_WIDTH-1:0] get_wrap_boundary (input logic [AXI_ADDR_WIDTH-1:0] unaligned_address, input logic [7:0] len);
    logic [AXI_ADDR_WIDTH-1:0] warp_address = '0;
    //  for wrapping transfers ax_len can only be of size 1, 3, 7 or 15
    if (len == 4'b1)
        warp_address[AXI_ADDR_WIDTH-1:1+LOG_NR_BYTES] = unaligned_address[AXI_ADDR_WIDTH-1:1+LOG_NR_BYTES];
    else if (len == 4'b11)
        warp_address[AXI_ADDR_WIDTH-1:2+LOG_NR_BYTES] = unaligned_address[AXI_ADDR_WIDTH-1:2+LOG_NR_BYTES];
    else if (len == 4'b111)
        warp_address[AXI_ADDR_WIDTH-1:3+LOG_NR_BYTES] = unaligned_address[AXI_ADDR_WIDTH-3:2+LOG_NR_BYTES];
    else if (len == 4'b1111)
        warp_address[AXI_ADDR_WIDTH-1:4+LOG_NR_BYTES] = unaligned_address[AXI_ADDR_WIDTH-3:4+LOG_NR_BYTES];
 
    return warp_address;
endfunction
```

## 3、突发传输模式中的地址计算

![image-20240122145914670](D:\lqh\Typora\图片\image-20240122145914670.png)

首先定义上述地址。

在一个burst中的每个transfer有如下关系式：

- Start_Address = AxADDR
- Number_Bytes = 2 ^ AxSIZE (在程序中可以使用1 << AxSIZE代替指数运算符)	
- Burst_Length = AxLEN + 1
- Aligned_Address = (INT(Start_Address / Number_bytes)) x Number_Bytes     

![image-20240122150630394](D:\lqh\Typora\图片\image-20240122150630394.png)

```verilog
number_byte = 1 << r_size_q; //ARSIZE决定了每个transfer里DATA的有效字节数量，因此也决定了地址的增量
//例如，ARSIZE='b010，则DATA中4个字节有效，地址每次+4,左移r_size_q得到的就是2**r_size_q

aligned_address = {r_addr_q[AXI4_ADDRESS_WIDTH - 1 : r_size_q], r_size_q{1'b0}}; //以number_byte为除数对齐
```

r_size_q是2，那么number_byte就是'b100，[2:0]

取r_addr_q[AXI4_ADDRESS_WIDTH - 1 : 2]，后[1:0]两位忽略，视为0

所以可以看到，r_size位忽略，高AXI4_ADDRESS_WIDTH-r_size位保留。

```verilog
BURST_RD,WRAP_RD:begin
    arready_o = 1'b0;
    read_req = 1'b1;
    rvalid_o = 1'b1; //valid一定不能放在if(RREADY_i)里，不然就valid依靠readyl了(禁止)

    // number_byte = 1 << r_size_q; //ARSIZE决定了每个transfer里DATA的有效字节数量，因此也决定了地址的增量
    //例如，ARSIZE='b010，则DATA中4个字节有效，地址每次+4,左移r_size_q得到的就是2**r_size_q
    //Aligned_Address = (INT(Start_Address / Number_Bytes) ) x Number_Bytes.
    aligned_address = {r_addr_q[AXI4_ADDRESS_WIDTH - 1 : r_size_q], r_size_q{1'b0}}; //以number_byte为除数对齐

    // WRAP模式下burst_length必须是2，4，8，16，对应的AXLEN是'b1,'b11,'b111,'b1111
    //wrap_boundray = (INT(Start_Address / (Number_Bytes × Burst_Length))) × (Number_Bytes × Burst_Length).
    //根据余数一定小于除数定理，直接将被除数的后M位置0，即为商(对齐的地址)，M为除数的位宽，number_byte的有效位宽是1+r_size位
    if(r_len_q == 4'b1)begin
        wrap_boundary = {r_addr_q[AXI4_ADDRESS_WIDTH - 1 : 1 + r_size_q], {(1 + r_size_q){1'b0}}}; 
        // 除数为number_byte * (1 + r_len_q)，number_byte的有效位宽是1 + r_size，也就是[r_size : 0],
        // 乘Burst_length相当于再左移r_len_q的有效位宽,比如r_len_q为4’b111，则再左移3位
    end
    else if(r_len_q == 4'b11)begin
        wrap_boundary = {r_addr_q[AXI4_ADDRESS_WIDTH - 1 : 2 + r_size_q], {(1 + r_size_q){1'b0}}}; 
    end
    else if(r_len_q == 4'b111)begin
        wrap_boundary = {r_addr_q[AXI4_ADDRESS_WIDTH - 1 : 3 + r_size_q], {(1 + r_size_q){1'b0}}}; 
    end
    else if(r_len_q == 4'b1111)begin
        wrap_boundary = {r_addr_q[AXI4_ADDRESS_WIDTH - 1 : 4 + r_size_q], {(1 + r_size_q){1'b0}}}; 
    end
    else begin
        wrap_boundary = {AXI4_ADDRESS_WIDTH{1'bx}};
    end
    //回环上边界=first address的对齐地址 + transfer个数 * 单个transfer中的字节数量
    upper_wrap_boundary = wrap_boundary + (r_len_q + 1) << (r_size_q); // 一个数乘以2的指数，可以转换成这个数左移指数项
```

