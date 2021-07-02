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

  





































