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

#### :five:SD卡初始化流程

![image-20210704113444241](https://gitee.com/LucasXm/img/raw/master/img//image-20210704113444241.png)

* 卡一共分为4类

  * SD2.0高容量卡
  * SD2.0标准容量卡
  * SD1.x卡
  * MMC卡
  * 

* 初始化步骤

  * 卡上电（SDIO_POWER[1:0]=11）

  * 发送CMD0命令，进行软件复位

  * 发送CMD8命令，用于区分SD卡2.0（只有2.0及以后的卡才支持CMD8命令）

    ![image-20210704125309538](https://gitee.com/LucasXm/img/raw/master/img//image-20210704125309538.png)

    * 通道CMD8带的参数我们可以修改VHS位，即告诉SD卡主机供电情况

      ![image-20210704125353604](https://gitee.com/LucasXm/img/raw/master/img//image-20210704125353604.png)

    * 设置参数位0x1AA，VHS[19:16]=0001b,即使用2.7V~3.6V供电

    * 如果SD卡支持CMD8且支持该电压范围，则会通过CMD8的响应R7将参数部分返回给主机

    * 如果不支持，则不响应

  * 发送ACMD41（**发送相关命令ACMD之前，必须发送通用命令CMD55**）进一步确认卡的操作电压范围，并通过HCS位来告诉SD卡主机是否支持高容量卡

    * ACMD41命令格式如下

      ![image-20210704165311296](https://gitee.com/LucasXm/img/raw/master/img//image-20210704165311296.png)

    * ACMD41响应（R3）包含SD卡OCR寄存器内容，OCR寄存器内容如下图

      ![image-20210704165519990](https://gitee.com/LucasXm/img/raw/master/img//image-20210704165519990.png)

    *  对于支持CMD8指令的卡（SD卡2.0及以后版本的卡）

      * 主机设置ACMD41的HCS位为1，表示主机支持SDHC卡
      * 主机设置ACMD41的HCS位为0，表示主机不支持SDHC卡
      * 如果SDHC卡接收到HCS为0，则不会返回卡就绪状态
      * 对于不支持CMD8的卡，HCS设置为0即可

    * SD卡在接收到ACMD41指令后，将返回OCR寄存器内容

      * 如果是2.0的卡，主机可以通过判断OCR的CCS位来判断是SDHC还是SDSC
      * 如果是1.x的卡，忽略CCS位
      * OCR寄存器的最后一位，告诉主机SD卡上电是否完成，如果上电完成，该位将会置1

  * 发送CMD2，获取CID寄存器的数据

    * CID寄存器数据各位定义如下表

      ![image-20210704171117175](https://gitee.com/LucasXm/img/raw/master/img//image-20210704171117175.png)

    * SD卡，收到CMD2后，将返回R2长响应（136位）

      * 包含128位有效数据（CID寄存器内容）

      * 存放在SDIO_RESP1~4寄存器里面

  * 发送CMD3，用于获取卡相对地址（RCA，必须为非0）

    * 对于SD卡（非MMC卡），再收到CMD3指令后，会返回一个新的RCA给主机，方便主机寻址
      * RCA的存在允许一个SDIO接口挂多个SD卡
    * 对于MMC卡，则不是自动返回RCA，而是由主机主动设置RCA
      * 通过CMD3带参数（高16位用于设置RCA），实现RCA的设置
      * MMC卡也支持一个SDIO接口挂多个MMC

  * 在获得卡 RCA 之后，我们便可以发送 CMD9（带 RCA 参数），获得 SD卡的CSD寄存器内容，从CSD寄存器，我们可以得到SD 卡的容量和扇区大小等内容

  * 最后通过CMD7命令，选中我们要操作的SD卡，即可开始对SD卡的读写操作

### :star:FATFS文件系统

#### :one:FATFS简介

* FATFS源码获取地址 [传送门](http://elm-chan.org/fsw/ff/00index_e.html)

* 层次结构图

  ![image-20210704220913464](https://gitee.com/LucasXm/img/raw/master/img//image-20210704220913464.png)

* 移植FATFS，只需要对ffconf.h文件和diskio.c进行修改
  * ffconf.h文件介绍

    | 配置选项      | 作用                                                         |
    | ------------- | ------------------------------------------------------------ |
    | _FS_TINY      | 0：正常模式<br>1：精简模式，每个文件对象减少_MAX_SS个字节    |
    | _FS_READONLY  | 0：读写模式<br>1：只读模式                                   |
    | _USE_STRFUNC  | 0：不支持使用字符串操作<br>1或2：支持使用字符串操作          |
    | _USE_MKFS     | 0：禁止格式化<br>1：使能格式化                               |
    | _USE_FASTSEEK | 0：禁止快速查找<br/>1：使能快速查找                          |
    | _USE_LABEL    | 0：不支持磁盘盘符读取与设置<br/>1：支持磁盘盘符读取与设置    |
    | _CODE_PAGE    | 设置语言类型<br>设置为936，即简体中文（GBK码，需要c936.c文件支持） |
    | _USE_LFN      | 0：不支持长文件名<br>1~3：支持长文件名，但是存储地方不一样。我们选择使用3，通过ff_memalloc 函数来动态分配长文件名的存储区域 |
    | _VOLUMES      | 用于设置 FATFS 支持的逻辑设备数目                            |
    | _MAX_SS       | 扇区缓冲的最大值，一般设置为 512                             |

#### :two:移植步骤

* 在integer.h文件中定义好数据类型

* 配置ffconf.h文件来配置FATFS相关功能

* 对diskio.c进行底层驱动编写

  ![image-20210704230745851](https://gitee.com/LucasXm/img/raw/master/img//image-20210704230745851.png)

  * disk_initialize函数

    ![image-20210704231221243](https://gitee.com/LucasXm/img/raw/master/img//image-20210704231221243.png)

  * disk_status函数

    ![image-20210704231520299](https://gitee.com/LucasXm/img/raw/master/img//image-20210704231520299.png)

  * disk_read函数

    ![image-20210704231618058](https://gitee.com/LucasXm/img/raw/master/img//image-20210704231618058.png)

  * disk_write函数

    ![image-20210704231909703](https://gitee.com/LucasXm/img/raw/master/img//image-20210704231909703.png)

  * disk_ioctl函数

    ![image-20210704231943345](https://gitee.com/LucasXm/img/raw/master/img//image-20210704231943345.png)

  * get_fattime函数

    ![image-20210704232043828](https://gitee.com/LucasXm/img/raw/master/img//image-20210704232043828.png)

#### :three:注意事项

* Fatfs指定的内存地址并不总是4字节对齐的，对于SD卡，必须使用函数进行中间处理
* 在使用 FATFS的时候，必须先通过 f_mount函数注册一个工作区，才能开始后续 API 的使用

## :hammer:测试内容



















