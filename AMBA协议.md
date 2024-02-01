# AMBA协议

## **介绍**

**总线**（Bus）是指[计算机组件](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/w/index.php%3Ftitle%3D%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BB%84%E4%BB%B6%26action%3Dedit%26redlink%3D1)间规范化的交换[数据](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE)（data）的方式，即以一种通用的方式为各组件提供数据传送和[控制逻辑](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/w/index.php%3Ftitle%3D%E6%8E%A7%E5%88%B6%E9%80%BB%E8%BE%91%26action%3Dedit%26redlink%3D1)。（总线只是提供数据转移的一种方式，不能决定数据本身）

**片外总线**：一般指两颗芯片或两个设备之间的数据交换传输方式，包括UART、I2C、CAN、SPI等

**片上总线：**同一个芯片上不同模块之间的一种规范化交互数据的方式。

- **总线带宽**：指的是单位时间内总线上传送的数据量；其大小为总线位宽*工作频率（单位为bit，但通常用Byte表示，此时需要除以8）。
- **总线位宽**：指的是总线有多少比特，即通常所说的32位，64位总线。
- **总线时钟工作频率**：以MHz或者GHz为单位。
- **总线延迟**：一笔传输从发起到结束的时间。在突发传输中，通常指的是第一笔的发起和结束时间之间的延迟。

**AMBA(Advanced Microcontroller Bus Architecture)**

AMBA是一系列协议，其定义了SoC各个模块之间是如何互联，如何通信。

![img](https://pic1.zhimg.com/v2-d24f3b861c1c2e610104758586ed43d4_r.jpg)

- AMBA“X”代表了第多少代协议。
- AMBA1已经不用掌握了，因为市场已经抛弃了这代协议。
- AMBA2推出了AHB协议，该协议沿用至今；而AMBA2中的APB2协议，也已经被市场抛弃了。
- AMBA3推出了**APB3、AXI3、AHB-lite协议**（ATB协议是用作CPU的trace的）。需要注意的是，AXI3代表的含义是，第三代AMBA总线中的AXI协议。这也是AXI协议的首次提出。**因此也没有AXI1和AXI2协议！**APB3和AHB-lite目前仍然广泛使用。
- AMBA4推出了ACE协议，以及AXI的兄弟协议，AXI-lite和AXI-stream。
- AMBA5对AXI、AHB和ACE协议进行了优化，此外还推出了CHI协议。
- APB1.0一般指的是2003年发布的AMBA3中的APB版本，也就是APB3。而APB2.0一般指的是2010年发布的AMBA4中的APB的2.0版，也就是APB4。

![img](https://pic4.zhimg.com/v2-c30659932235f1aec416b0b2166aec4f_r.jpg)

image/image-20240131195421966.png
