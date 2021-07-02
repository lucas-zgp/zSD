# zSD

## :bell:基础知识

### :star:SDIO

#### :one:stm32f4的SDIO控制框图

![image-20210702224749934](https://gitee.com/LucasXm/img/raw/master/img//image-20210702224749934.png)

* SDIOCLK是适配器时钟，用于驱动SDIO适配器，一般为48Mhz，并用于产生SDIO_CK
* SDIO_CK是卡时钟，每个时钟周期传输一位命令或数据
  * SDIO_CK = SDIOCLK/(2+CLKDIV)
    * 时钟分频器不旁路时，可以使用上述公式计算卡时钟
    * 时钟分频器旁路时，SDIO_CK直接等于SDIOCLK
* SDIO_CMD是命令线
* SDIO_D[7:0]是数据线
* :exclamation:在SD卡在初始化的时候，SDIO_CK不能超过400Khz，初始化以后就不受限制了

#### :two:SDIO的命令与响应

![image-20210702231938898](https://gitee.com/LucasXm/img/raw/master/img//image-20210702231938898.png)

* SDIO的命令分为了相关命令（ACMD）和通用命令（CMD），应用命令发送之前，必须先发送通用命令CMD55。

* SDIO所有的命令和响应都是通过SDIO_CMD引脚发送的，任何命令的长度固定为48bit

  * 起始位、传输位、crc7、结束位由SDIO硬件控制
  * 命令索引由SDIO_CMD寄存器设置，命令参数由SDIO_ARG寄存器设置

* 除CMD0之外，SD卡接收到命令之后，都会响应。

  * 短响应（48bit）格式如下

    ![image-20210702232725787](https://gitee.com/LucasXm/img/raw/master/img//image-20210702232725787.png)

    * 硬件自动过滤起始位、传输位、crc7、结束位
    * 命令索引存储在SDIO_RESPCMD寄存器中
    * 参数存储在SDIO_RESP1寄存器中

  * 长响应（136bit）格式如下

    ![image-20210702232857044](https://gitee.com/LucasXm/img/raw/master/img//image-20210702232857044.png)

    * 硬件自动过滤后，仅保留cid/csd位域
    * 存放在SDIO_RESP1~SDIO_RESP4四个寄存器中

* 关于命令与响应的具体信息，可以查看参考文档《STM32F4xx中文参考手册》.

#### :three:SDIO控制器与SD卡之间的传输

* 对于SDID存储器，数据是以数据块的形式传输的

* 对于MMC卡，数据是以数据块或数据流的形式传输的

* SDIO（多）数据块读操作

  ![image-20210702234317399](https://gitee.com/LucasXm/img/raw/master/img//image-20210702234317399.png)

  * 主机发送相关命令给从机，从机将发送响应给主机
  * 接受到相关指令后，从机开始将数据块发送给从机
    * 单个数据块读取的时候，无需发送停止命令（CMD12），也可以停止
    * 多个数据块读取的时候，从机将移植发送数据块给主机，直到主机发送了停止命令

* SDIO（多）数据块写操作

  ![image-20210702234751791](https://gitee.com/LucasXm/img/raw/master/img//image-20210702234751791.png)

  * 通数据块读操作
  * 写数据块之前必须判断sd卡是否繁忙，只有在非繁忙的情况下才能写入
  * 繁忙信号由sd卡拉低SDIO_D0表示，硬件自动控制

#### :four:SDIO相关寄存器

 * SDIO电源控制寄存器（SDIO_POWER）

   ![image-20210703000855854](https://gitee.com/LucasXm/img/raw/master/img//image-20210703000855854.png)

   * 设置最后两位为1，打开电源。

 * SDIO时钟控制寄存器（SDIO_CLKER）

   ![image-20210703000818647](https://gitee.com/LucasXm/img/raw/master/img//image-20210703000818647.png)

   * WIDBUS用于设置SDIO总线位宽，正常使用的时候设置为1，即四位位宽
   * BYPASS用于设置分频器是否旁路
   * CLKEN用于设置是否使能SDIO_CK，我设置为1，即使能SDIO_CK
   * CLKDIV用于设置SDIO_CK的频率。一般设置为0，即得到24Mhz的SDIO_CK频率

* SDIO参数寄存器（SDIO_ARG）

  ![image-20210703000818647](https://gitee.com/LucasXm/img/raw/master/img//image-20210703001327660.png)

  作为命令参数的一部分，如果发送的命令具有参数，在将命令写入命令及寄存器之前，需要将参数加载到此寄存器

* SDIO命令响应寄存器（SDIO_RESPCMD）

  ![image-20210703001903873](https://gitee.com/LucasXm/img/raw/master/img//image-20210703001903873.png)

  低六位有效，用于存储最后收到的命令响应的命令索引。如果传输的命令响应不包含命令索引，则该寄存器的内容不可预知。

* SDIO响应1~4寄存器（SDIO_RESPx）

  ![image-20210703002258809](https://gitee.com/LucasXm/img/raw/master/img//image-20210703002258809.png)

  * 如果收到短响应，则数据存储在SDIO_RESP1寄存器
  * 如果收到长响应，则数据存储在SDIO_RESP1~SDIO_RESP4寄存器

* SDIO 命令寄存器 (SDIO_CMD)

  ![image-20210703002712691](https://gitee.com/LucasXm/img/raw/master/img//image-20210703002712691.png)

  ![image-20210703002727840](https://gitee.com/LucasXm/img/raw/master/img//image-20210703002727840.png)

  * CMDINDEX为命令索引号，即发送CMD1，其值为1，索引就设置为1
  * WAITRESP用于设置等待响应位

* SDIO 数据定时器寄存器 (SDIO_DTIMER)

  ![image-20210703003201817](https://gitee.com/LucasXm/img/raw/master/img//image-20210703003201817.png)

* SDIO 数据长度寄存器 (SDIO_DLEN)

  ![image-20210703003324407](https://gitee.com/LucasXm/img/raw/master/img//image-20210703003324407.png)

  * 用于设置需要传输的数据字节长度
  * 对于数据块的传输，改寄存器的值，必须是数据块长度的倍数

* SDIO 数据控制寄存器 (SDIO_DCTRL)

  ![image-20210703003522642](https://gitee.com/LucasXm/img/raw/master/img//image-20210703003522642.png)

  ![image-20210703003544905](https://gitee.com/LucasXm/img/raw/master/img//image-20210703003544905.png)

* 状态寄存器（SDIO_STA）、清除中断寄存器（SDIO_ICR）和中断屏蔽寄存器（SDIO_MASK）

  ![image-20210703003723651](https://gitee.com/LucasXm/img/raw/master/img//image-20210703003723651.png)

* SDIO 数据 FIFO 寄存器 (SDIO_FIFO)

  ![image-20210703003905388](https://gitee.com/LucasXm/img/raw/master/img//image-20210703003905388.png)

  * 数据FIFO寄存器接受和发送FIFO
  * 由一组连续的32个地址上32个寄存器组成，CPU可以使用FIFO读取多个操作数、
  * SDIO 将这 32 个地址分为 16 个一组，发送接收各占一半。而我们每次读写的时候，最多就是读取发送 FIFO 或写入接收 FIFO 的一半大小的数据，也就是 8 个字（32 个字节）
  * 必须4字节对齐

  

















