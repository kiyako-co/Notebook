# AHB2SRAM设计

本篇文章给大家讲解**AHB2SRAM的**设计，它的作用是**通过AHB接口去访问SRAM**，实际上我们可以将它理解成AHB协议到SRAM的native接口的协议转换，和之前讲的**apb2reg**是差不多的。

设计一个东西之前首先思考一下，为什么需要这个东西？我们直接用SRAM不可以吗？还真的不可以，因为前面讲过，我们设计的SoC主体其实是基于AHB和APB总线。普通的SRAM无法直接连在AHB总线上，必须通过ahb2sram这个模块完成协议转换，才可以连接SRAM。其实SoC设计很多时候都是做这种工作，给IP做一个AHB/AXI/APB的wrapper，然后像连连看一样，把对应的信号连接即可。

![img](D:\lqh\Typora\图片\v2-b05d3eb382019f78eaba9a134647d9bb_r.jpg)

## 1、AHB2SRAM模块简介

我们要设计的模块为紧耦合SRAM([Tightly coupled memory，TCM](#jump))，不考虑等待延迟，默认一拍回数据，不考虑报错。且只支持32bit位宽的SRAM。支持按字节写，该模块除了用在IC设计中，也可以用在FPGA设计中，因为FPGA提供的MIG(Memory Interface Generator, Xlinx中的IP用于提供DDR3、DDR4等多种存储器接口)，只支持AXI或者native接口，这种情况下还是需要AHB2SRAM这个模块来完成协议转换。

我们看一下接口，非常直观，一边是AHB接口，一边是SRAM的native接口。该模块实际就是完成协议的转换。

![img](D:\lqh\Typora\图片\v2-cd93e3f341ff8420f8ab00042c9d8e9a_720w.webp)

我们要设计的AHB2SRAM的规格说明如下：

- 完成AHB接口与SRAM接口的时序转换；
- 支持字节、半字、字的读写操作；
- 支持**先写后读**（不支持同时读写，单端口SRAM），并且读写地址相同的时候，读数据的拼接以及更新（比如对addr1进行写操作，写数据data1，下一拍马上就需要读，这个时候需要靠一个buffer，把上一拍写的数据data1作为这一拍读回的数据，因为data1有可能还没有写进SRAM）；

## 2、AHB读写时序和SRAM读写时序

我们分别看一下两端的读写时序，这样就清楚怎么做相应的协议转换了（做任何协议的转换都是如此，要弄清楚两边各自的协议）。

### 2.1、AHB读时序

![img](D:\lqh\Typora\图片\v2-6f20f24e0c8aac8d859a647cd85c5f43_720w.webp)

- 第一拍给出读写控制以及地址信号；（CTRL包括了HTRANS和其它的一些控制信号，这里简化了，大家理解意思就行）
- 当没有HREADY反压的时候，第二拍得到读数据信号；
- 当有HREADY反压的时候，第2拍以后的某一拍HREADY为高的时候，说明此时HRDATA有效，得到读数据信号；

### **2.2、AHB写时序**

![img](D:\lqh\Typora\图片\v2-7475bd0c6da11e016b93faa2d0f6312f_720w.webp)

- 第一拍给出读写控制以及地址信号；
- 第二拍给出写数据信号；

### 2.3、SRAM读时序

![img](D:\lqh\Typora\图片\v2-ab3694caa229325d188d5a94421c2299_720w.webp)

- SRAMCS是片选信号，chip select，只有为高的时候SRAM才会真正的工作；
- 第一拍给出读写控制，地址信号；
- 第二拍给出读数据信号；（不考虑反压和错误的情况下，和AHB一模一样）

### 2.4、SRAM写时序

![img](D:\lqh\Typora\图片\v2-e72b6261e047f1ebdad7c2dcdc5910ad_720w.webp)

- 第一拍同时给出写控制，地址，数据信号；（和AHB不一样！）

## 3、AHB2SRAM时序转换

可以分为以下的情况：

- 全部是读的情况：这种情况非常简单，因为本身二者就相同，第一拍给地址和控制信号，第二拍回数据，大家看下波形肯定可以理解，就不详细讲了。![img](D:\lqh\Typora\图片\v2-e20708c36762f75cf7160d685861fb0a_r.jpg)

- 全部是写的情况：**由于SRAM的写操作命令地址和数据是在一拍完成的**，而AHB写数据要在第二拍才能写入，所以需要将**AHB中的地址延迟一拍**再与SRAM中的地址进行同步，这样地址数据就满足了sram接口要求。![img](D:\lqh\Typora\图片\v2-36dcbd6050a2e8b0e1e852af12aef479_720w.webp)

- 先读后写：先读后写实际上不涉及数据更新，就是写跟在读后面而已，下面时序稍微有点问题，对于读这一部分ADDR和DATA。AHB和SRAM两侧应该是同一拍的。![img](D:\lqh\Typora\图片\v2-eb74b664a0ae0e8bb0940f7d26c2c427_720w.webp)

- 先写后读：相对来说最复杂的一种情况，当读写地址相同的时候，需要完成读数据的拼接以及更新。

  - **首先我们要判断是否是先写后读**，**我们需要把写的控制信号打一拍**。如果这一拍是读，上一拍是写同时满足，则说明出现了先写后读。buf_pend代表buffer write data valid。为什么需要这个呢？可能是先写后读再读，这个时候可能也还没有写进去，但是我们也认为是先写后读。当然也包括先写后读后读读读。。。过会看代就一目了然了

  - **然后我们判断是否读写地址相同**，这个就相对简单了。上一拍的地址和这一拍的地址相同，则代表读写地址相同。![img](D:\lqh\Typora\图片\v2-ff1aaa8ee6f422a6eff755ef4612347f_720w.webp)

  - 接下来我们就可以对读数据更新，可以看到如果buf_hit并且buf_we有效，则代表上一拍确实写了这个字节，**这种情况下我们就应该把这一字节替换成上一拍写的值**，而不是用SRAM读出来的值。

    ![img](D:\lqh\Typora\图片\v2-1a6a249c49bf74d45fa2401ec35620e5_720w.webp)

接下来我们看先写后读的时序情况，可以看到addr2这个地址是先写后读。可以看到buf_data_en和~HWRITE都为1，则代表出现了先写后读。可以将buf_data_en拉高。并且地址也相同，因此buf_hit也会拉高。这个时候HRDATA返回的数据就应该是完成替换以后的值。这个时序图应该还是比较清晰的，大家好好理解一下。

需要注意的是，addr2写操作不会在下一拍就写进去SRAM，因为出现了先写后读，这个写SRAM的操作会被暂时的pending住！当addr3来的时候，这个时候addr对应的data2才能够写进去。大家看后面的代码就知道了。![img](D:\lqh\Typora\图片\v2-6b4740b1ac0dd721a95fc959a2e35427_720w.webp)

## 4、AHB2SRAM代码

```verilog
wire        ahb_access   = HTRANS[1] & HSEL & HREADY; //HTRANS为1表示连续或者非连续传输，HSEL拉高代表发起传输
wire        ahb_write    = ahb_access &  HWRITE;  //写控制信号
wire        ahb_read     = ahb_access & (~HWRITE);//读控制信号

// Stored write data in pending state if new transfer is read
//   buf_data_en indicate new write (data phase)
//   ahb_read    indicate new read  (address phase)
//   buf_pend    is registered version of buf_pend_nxt
//发生了先写后读的情况，注意buf_pend或buf_data_en
//意味着如果是先写后读再读，也会拉高buf_pend_nxt，当然再读也是一样的会拉高。
wire        buf_pend_nxt = (buf_pend | buf_data_en) & ahb_read;

// RAM write happens when
// - write pending (buf_pend), or
// - new AHB write seen (buf_data_en) at data phase,
// - and not reading (address phase)
//注意，buf_data_en是ahb_write打了一拍，按照之前的逻辑设计。我们写操作控制信号本身就需要打一拍，并且新来的一拍不能是读
//如果新来的一拍是读，上一拍是写，那么就是先写后读，这个时候会一直不写，直到新来的也是写，这个才能真正的写进去SRAM
//所以也叫buf_pend，意思为暂停，因此是buf_pend和buf_data_en相或。
//为什么要这么设计呢？我也不知道，我觉得有可能是写的这个数据本身就已经存好了，暂时代替了SRAM的作用，因此没必要着急直接写
//进去SRAM，可以节省一点逻辑资源。大家对其进行修改当然也是可以的。
wire        ram_write    = (buf_pend | buf_data_en)  & (~ahb_read); // ahb_write

// RAM WE is the buffered WE
assign      SRAMWEN  = {4{ram_write}} & buf_we[3:0];

// RAM address is the buffered address for RAM write otherwise HADDR
assign      SRAMADDR = ahb_read ? HADDR[AW-1:2] : buf_addr;

// RAM chip select during read or write
assign      SRAMCS   = ahb_read | ram_write;
```

然后我们看一下读回数据的情况，这个实际上和上面的电路图保持一致。当buffer_hit即读写地址相同，并且上一拍真的写进去了。就用写的数据代替从SRAM回来的数据作为读回数据即可。

```verilog
wire [ 3:0] merge1  = {4{buf_hit}} & buf_we; // data phase, buf_we indicates data is valid

assign HRDATA =
{ merge1[3] ? buf_data[31:24] : SRAMRDATA[31:24],
 merge1[2] ? buf_data[23:16] : SRAMRDATA[23:16],
 merge1[1] ? buf_data[15: 8] : SRAMRDATA[15: 8],
 merge1[0] ? buf_data[ 7: 0] : SRAMRDATA[ 7: 0] };
```

## <span id = 'jump'>5、紧耦合内存(TCM)</span>

TCM is designed to provide low-latency memory that can be used by the processor. This memory can be used to hold critical routines, such as interrupt handling routines or real-time tasks where the indeterminacy of a cache would be highly undesirable. In addition, you can use TCM to hold ordinary variables, data types whose locality properties are not well suited to caching, and critical data structures such as interrupt stacks. A TCM is physically located very close to the processor core. **Accesses to the TCM will typically be configured to capture or return data in a single cycle**. By storing time-critical routines such as exception handlers in the TCM, the processor can have immediate access to the subroutine rather than having to wait for an initial code fetch from external memory.

TCM is expected to be used as part of the physical memory map of the system, and is not expected to be backed by a level of external memory with the same physical addresses. Particular memory locations must be contained either in the TCM or in the cache. In particular, no coherency mechanisms are supported between the TCM and the cache. This means that it is important when allocating the base address of the TCM to ensure that the same address ranges are not contained in the cache.

由于是高速缓存，所以这两块内存区域被当做特殊的用途。比如某些对时间要求非常严格的代码，就可以被放到ITCM中执行。这可以有效地提高运行速度。某些需要频繁存取的数据，也可以放到DTCM中以节省存取时间。
怎么样把代码放到ITCM中？有两种方法。一种是使用gcc特有的“属性标签”，将指定代码赋予“ITCM”属性，此时该代码会被载入ITCM中执行。还有一种方法是直接将.c源文件改成.itcm.c，此时源文件会被直接编译成在ITCM中运行的目标文件。
而DTCM就方便得多了。虽然两个TCM都是可映射的，也就是说，它们的地址并非固定，但是一般会将其分别映射到固定地址。既然已经有了固定地址，那么就可以很轻松地访问了。不过，正如刚才所说的，这两块内存空间都是有特殊用途的，所以不建议直接访问。相比于ITCM来说，DTCM更加重要。因为在这块内存中，存在着一个非常重要的对象——栈。局部变量和函数调用的参数，就是靠栈进行传递的。由于DMA无法访问TCM，所以也就无法访问栈。又由于局部变量是被开辟到栈中，所以DMA也无法对局部变量进行传递。