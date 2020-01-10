---
title: 物联网IoT上云协议之Modbus快速入门教程
tags:
  - Modbus
  - 上云协议
  - 物联网
  - OT融合
  - 工业4.0
  - 工业互联网
  - 工业通信
  - 物联网云平台
url: 281.html
id: 281
categories:
  - 物联网IoT
keywords: Modbus,物联网IoT,工业互联网,工业4.0,物联网云平台,上云协议,万物联网
description: '什么是Modbus,物联网IoT上云协议之Modbus零基础快速入门教程'
date: 2020-01-01 20:10:40
---

关注我，获取更多**物联网IoT+云原生**的技术分享。

![](/images/wp/wx.jpg)

[物联网IoT协议之mqtt快速入门教程](https://www.debugself.com/archives/245)

[物联网IoT协议之OPC UA快速入门教程](https://www.debugself.com/archives/262)

[物联网IoT协议之LoRaWAN快速入门教程](https://www.debugself.com/archives/268)

[物联网IoT协议之NB-IoT/CoAP快速入门教程](https://www.debugself.com/archives/276)

# 物联网IoT上云协议之Modbus快速入门教程

## 什么是Modbus
**Modbus和OPC UA、mqtt本质一样，都是为了实现多个设备相互通信的应用层协议**。Modbus于1979年产生于Modicon公司（现被Schneider公司收购），一经面世因其简单开放的通信方式逐渐成为工业系统中流行的标准。Modbus的国际组织主要有Modbus-IDA，负责推广Modbus标准以及对Modbus产品进行认证。Modbus官网见 www.modbus.org，网站主要包括协议文档，Modbus产品和厂商等内容。

之前的[物联网IoT协议之OPC UA快速入门教程](https://www.debugself.com/archives/262)中提到OPC UA是工业领域常用的协议，其实在工业领域中Modbus比OPC/OPC UA更常见，市面上很多数据采集设备(如温湿度采集)都使用Modbus协议；OPC UA大量的专有名词（如节点、服务、引用等）总能把初学者弄得云里雾里，**而Modbus比OPC UA简单太多了**，简单之处体现包括：

- Modbus最开始使用RS232，RS485等串行链路作为底层通信方式，串行总线的接口芯片成本低，而且布线也简单方便；
- Modbus是简单的应用层协议，其信息格式简单易懂，下文会详细讲述协议内容；
- Modbus相关的资料文档有很多。

## 什么是总线/什么是Modbus总线
Modbus中的bus是总线的意思，那么什么是总线呢？**总线是一种网络拓扑结构**，网络拓扑结构有星型、环型、树型、总线型等多种，总线型拓扑结构是其中一种。**无论哪种网络拓扑结构，最终都是为了实现多个设备相互通信**。

两个设备相互通信，用一条线把两个设备连接起来即可。

![网络拓扑结构](/images/wp/modbus_ab.png)

三个设备相互通信，两两相连需要3条线。

![网络拓扑结构](/images/wp/modbus_abc.png)

四个设备相互通信，两两相连需要6条线。

![网络拓扑结构](/images/wp/modbus_abcd.png)

很明显，**随着设备个数的增加，两两相连需要的线按指数级增长，所以两两相连的方式肯定是不合适了**，总线型拓扑结构通过引入总线来解决多个设备连线并通信的问题。

![总线型网络拓扑结构](/images/wp/modbus_bus.png)

多个设备复用同一条总线，减少了布线的复杂度；但是复用同一条总线时会遇到新的问题，**当多个设备同时发送数据，数据会产生冲突**。为了避免数据冲突，有多种解决办法，比如以太网使用的CSMA/CD（即载波侦听多路访问/冲突检测），CSMA协议要求设备在发送数据之前先监听总线/信道，如果总线空闲，设备就可以发送数据，如果总线忙，则设备不能发送数据；而Modbus采用了一种更简单的避免冲突的方式，即主从通信模式。

### Modbus的主从通信模式
Modbus为了避免多个设备数据的冲突，使用了单主机/多从机的模式，即整个系统中只能有一个主机(Master)和多个从机(Slave).

![Modbus的主从通信模式](/images/wp/modbus_ms.png)

Modbus使用请求-应答的模式，且Modbus标准规定，**Modbus请求只能由主机发起**，主机发送请求数据给从机，从机只有在接收到主机的请求数据后，才能发送应答数据到总线上；整个过程中**从机只能被动应答，不能主动发送数据到总线上**。通过主从通信模式，Modbus避免了多个设备数据的冲突；

![Modbus的主从通信模式](/images/wp/modbus_yd.png)

为了避免数据冲突，不允许主机并行发送多个消息到总线上，主机发送请求数据后，必须等到从机应答数据或者超时后，才能发送下一个请求；主机发送信息给从机后，主机会启动一个等待超时计时器，如果主机没有收到从机的应答信息，**只有在等待超时计时器超时后，才可以发送下一个请求信息**。

**每个从机都一个唯一的从机地址，主机发起请求时，请求数据中携带了目标从机地址**；主机把请求数据发送到总线上，**总线上所有从机都会收到该请求，从机会检查该请求中携带的从机地址是否是自己，如果是自己就回复应答数据，如果不是自己就忽略请求**。

## Modbus的业务模型

Modbus的业务模型也比较简单，源码先生把Modbus业务模型简称为“读写数据”模型，即**通过“读”某个（内存）地址的数据，获取设备采集的数据（或者设备的状态）；通过“写”某个（内存）地址的数据，实现对设备的控制（如配置设备的参数）**。

Modbus标准中，把Modbus地址空间划分为4个区域。

![Modbus数据模型](/images/wp/modbus_addr_en.png)

这4个区域，在很多文档中称为4种寄存器。

![Modbus寄存器分类](/images/wp/modbus_addr_zh.jpg)

之所以叫寄存器，是沿用了历史叫法，Modbus最开始是用来操作PLC的寄存器，如今可以把寄存器泛化为内存，**4种寄存器等效为4个内存分区即可**。
至于寄存器的名称，如寄存器0的名称为线圈(coils)状态，也是采用了历史叫法(线圈是继电器上的线圈，继电器是电子开关，给线圈通电，继电器开，给线圈断电，继电器关，所以线圈只有开/关这两种状态），**不明白什么是线圈的话，完全可以忽略线圈这个名字，只要记住寄存器0对应数据只有0/1这两种值**即可，这也是上图中single bit的含义（正好对应了线圈开/关这两种状态，其实是通过线圈这个名称告诉使用者，这个寄存器分区的数据只有0/1），这意味着“读”寄存器0的任何地址的数据，**读到的数据不是0就是1**，读寄存器0的0x0000地址的数据，读到的是0(或者1)，读取寄存器0的0x0001地址的数据，读到的也是0(或者1)。

请忽略寄存器的名称，4种寄存器本质都是“内存”，每中寄存器都包含了一段内存地址空间（每种寄存器的地址取值范围为0x0000-0xFFFF），用户需要做的就是“读写内存”。

4种寄存器按照功能的不同分类，比如寄存器0可读可写，而寄存器1只读；寄存器0的数据是single bit(不是0就是1)，而寄存器3的数据是16-bit world(两个字节)，这意味着读取寄存器3的0x0000地址的数据，读出的是2个字节的数据，读取寄存器3的0x0001地址的数据，读出的也是2个字节的数据。

注意，4种寄存器分别是寄存器0、寄存器1、寄存器3、寄存器4，**没有寄存器2**，至于为什么没有寄存器2，估计也是历史原因造成的吧？！

![都怪我](/images/wp/modbus_lishi.jpg)

## Modbus报文格式/消息帧格式
Modbus有3种的报文：

- Modbus ASCII：使用ASCII编码，底层使用RS458串行链路通信
- Modbus RTU：使用原始二进制，底层使用RS458串行链路通信
- Modbus TCP：使用原始二进制，底层使用TCP通信

这三种报文的报文主体（包括从机地址+功能码+数据域）都相同，三者没有本质区别。由于Modbus ASCII使用ASCII编码，相比Modbus RTU使用的原始二进制，前者占用空间更大，传输效率低，所以Modbus ASCII并不常用，这里主要介绍Modbus RTU和Modbus TCP。

### Modbus RTU
Modbus RTU的报文格式如下图。

![Modbus RTU](/images/wp/modbus_rtu.png)

#### 从机地址(Slave Address)
设备的从机地址用来标识设备的唯一性，用户可以自定义配置设备的从机地址，只要保证总线上每个设备的从机地址唯一，不冲突即可，从机地址范围：1-247，0为广播地址，248-255为预留地址。

#### 功能码(Function Code)
功能码对应了设备提供的某个服务/功能，Modbus标准中把功能码分为3类：

- 公共功能码(Public Function Codes)：Modbus标准定义好的，功能已经明确的功能码，此类功能码可以查询Modbus标准文档( http://www.modbus.org/docs/Modbus_Application_Protocol_V1_1b3.pdf)
- 用户自定义功能码(User-Defined Function Codes)：设备根据需求自定义的功能码，取值范围65-72和100-110；
- 保留功能码(Reserved Function Codes)：目前没有使用，可忽略

虽然Modbus标准中定义了很多个公共功能码，但实际场景中，常用的只有读/写4种寄存器相关的功能码，读写又分为Single读写(一次只读写一个地址的数据)和Multiple读写(一次读写多个地址的数据)：

- 0x01 Multiple Coils 批量读寄存器0
- 0x02 Multiple Discrete Inputs 批量读寄存器1
- 0x03 Multiple Holding Registers 批量读寄存器4
- 0x04 Multiple Input Registers 批量读寄存器3
- 0x05 Single Coil 写单个寄存器0
- 0x06 Single Holding Register 写单个寄存器4
- 0x07 Read Exception Status 读异常状态
- 0x0F Multiple Coils 批量写寄存器0
- 0x10 Multiple Holding Registers 批量写寄存器4

开发Modbus设备时，根据设备需求可自行选取对应的功能码：

- 当设备数据是开关量0/1，读设备数据的可选功能码有0x01和0x02；
- 当设备数据是开关量0/1，同时只允许读不允许写的话，读设备数据的功能码**只能选择0x02**，因为功能码0x01对应的寄存器0允许写；
- 当数据长度大于2字节长度时(比如4字节)，可以选取批量读写的功能码（批量读功能码0x03和0x04，批量写功能码0x10），注意此时会遇到大端/小端的问题（另外，4个字节的数据对应是整型还是浮点数，这取决于应用程序如何解析4个字节的数据，和Modbus标准没有关)

当购买了Modbus设备时，一般需要从设备配套的说明文档中查询设备使用哪些功能码、以及读写数据的含义。

#### 数据域（Data）
数据域用来存放真正要通信的数据，也就是常说的Payload。**数据域以字节为单位，长度可变，数据域的格式也是可变的，具体格式由功能码决定**，对于有些功能码，数据域可以为空，不同的功能码，数据域格式不同，可以通过Modbus标准文档的功能码查询具体的数据域格式，下图是功能码0x03对应的数据域格式。

![Modbus数据域Payload](/images/wp/modbus_payload.png)

0x03是批量读，所以请求报文的数据域包括了要读取“寄存器”的起始数据地址、要读取的寄存器个数。

**数据地址**

Modbus业务模型为“读写数据”模型，而读写时必须要有数据地址，理论上每种寄存器的数据地址的取值范围都为0x0000-0xFFFF，注意**不同种类的寄存器的数据地址是独立**，寄存器可以有0x0000，寄存器2也可以有0x0000，数据地址虽然相同，但是读写数据不会相互影响。
实际情况中，由于历史原因，寄存器的数据地址有多种标法。

![Modbus数据地址表示法](/images/wp/modbus_reg_addr.png)

虽然标法各有异同，不过也比较好区别，记住如下特征即可：

- 地址范围（最大）都是0x0000-0xFFFF，但是分为16进制表示法或者10进制表示法
- 地址最前面，有的有**寄存器序号的前缀**，有的没有，见上图中红色框，寄存器序号对应4种寄存器
- 16进制表示法的地址从0开始，而十进制表示法的地址从1开始

#### CRC校验
对数据的正确性和完整性进行校验，计算方式是固定的，按固定的算法计算出CRC值即可。

### Modbus TCP
虽然RS485串行链路比以太网链路简单，但是鉴于以太网太流行了，Modbus引入TCP/IP的作为通信方式。Modbus是应用层协议，不管底层使用RS485链路还是TCP，应用层变化不大。

![Modbus TCP](/images/wp/modbus_tcp.jpg)

对比Modbus RTU报文，Modbus TCP报文有以下异同：

- 功能码和数据域和Modbus RTU相同
- 去掉了从机地址（即上图中的附加地址），因为TCP是面向连接的，TCP必须先根据IP地址建立一对一的连接，连接建立后，已经不在需要从机地址了；
- 去掉了差错校验，因为TCP是可靠传输，TCP传输层已经做过了数据校验了，Modbus应用层可以不再校验；
- 增加了事务处理标志，这个可以作为自增ID来标识不同的报文，由于Modbus标准中不允许多个消息并行发送，所以事务处理标志填为0也没问题；
- 增加了协议标识符，Modbus协议值固定为0x00；
- 增加了数据长度，即Modbus TCP报文的长度，因为TCP是流式数据，需要使用数据长度来识别两个报文的边界；
- 增加了单元标识符，这里可以填0或者填写从机地址（可用于网关转发请求给从机）；

前面提到Modbus主从通信模式主要是为了避免多个设备的数据冲突，而TCP底层已经使用CSMA/CD解决了数据冲突问题，而且TCP是全双工通信（RS485链路是半双工），这意味着Modbus TCP的从机主动发送报文也不会产生冲突的，但是如果从机主动发送报文，就不符合Modbus标准了。**TCP是面向连接的，基于Modbus TCP实现多主/多从的通信也是可行的，不会产生冲突，但是就不符合Modbus标准了**。

TCP中有客户端（Client）和服务器(Server），对应到Modbus中，Modbus主机(Master)对应了TCP客户端（Client）,Modbus从机(Slave)对应了TCP服务器（Server），使用Modbus TCP时，先使用标准的socket建立客户端到服务器的TCP连接，建立后发送Modbus TCP报文即可。


## Modbus之Hello World
Modbus软件，比较流行是Modbus Poll及Modbus Slave，功能也比较完善，不过是收费软件(可以试用30天）。作为Hello World的演示，源码先生找到了一款免费的Modbus模拟软件EasyModbusTCP,可以通过https://github.com/rossmann-engineering/EasyModbusTCP.NET/releases下载。

### 运行Modbus TCP Server（即从机）
双击EasyModbusServerSimulator.exe启动，程序自动以本机IP建立TCP服务器，端口502。

![Modbus TCP Server](/images/wp/modbus_tcp_server.png)

### 运行Modbus TCP Client（即主机）
双击EasyModbusClientExample.exe，启动客户端。如下图点击connect，连接服务器。

![Modbus TCP Client](/images/wp/modbus_tcp_client.png)

### 读数据
客户端界面，点击“Read Coils -FC1”按钮，可以读取寄存器0的数据。

![Modbus读数据](/images/wp/modbus_tcp_read.png)

上图中，用户可以自定义填写读取数据的起始地址、要读取的寄存器个数；也可以查看读取过程使用的请求报文和应答报文。

### 写数据
下图是写寄存器4的流程，请参考图中的1-5的顺序

![Modbus写数据](/images/wp/modbus_tcp_write.png)

写数据成功后，再次读取写入的数据，正确读取上一步骤写入的110。

![Modbus写数据](/images/wp/modbus_tcp_write3.png)

## 使用Modbus上云
由于Modbus主从通信模式，从机只能被动应答，不能主动上报数据，所以Modbus其实不太适合上云，但是现存大量Modbus设备，如果支持Modbus上云的话，可以快速把现有设备接入到云端，还是非常有必要的。

![Modbus上云网关](/images/wp/modbus_gateway.png)

上图是Modbus标准文档中提供的Modbus通信框图，图中引入了网关的概念，要上云的话，需要使用到网关。网关需要带有Wifi/4G/网口，网关通过TCP/IP和云端通信，具体上云有以下两种方式。

### 方式一：云端作为Modbus主机

![Modbus上云网关](/images/wp/modbus_gateway1.png)

- 云端软件平台作为Modbus主机，云端定时发送请求报文给网关，网关再把请求报文转发Modbus总线上，Modbus总线上的从机收到请求后，返回应答数据到Modbus总线，网关读取应答报文并转发给云端；
- 云端和网关之间使用Modbus TCP报文通信，网关和设备之间可以使用Modbus RTU，也可以使用Modbus TCP报文通信；
- 网关作为转发的角色，在云端和从机设备之间转发数据，网关同时也作为转换协议的角色，比如把Modbus TCP转换为Modbus RTU；
- 云端需要不停的发送请求报文，相当于不停的轮询，当从机设备数量比较多时，云端的轮询压力大，通信带宽要求也比较高。

### 方式二：网关作为Modbus主机

![Modbus上云网关](/images/wp/modbus_gateway2.png)

- 网关作为Modbus主机，网关主动定时发送请求报文到Modbus总线上，然后接收从机的应答报文；网关接收到从机应答报文后，可以使用其他协议（比如mqtt）把应答信息转发给云端。
- 网关除了作为Modbus主机，还需要作为连接云端的客户端（比如Mqtt客户端）把轮询到的数据主动上报给云端；
- 需要把网关如何轮询从机的信息从云端配置给网关，配置稍微麻烦，但是轮询压力分散在网关上，比较符合边缘计算的思路。

## Modbus优缺点
Modbus优点：协议简单，接口芯片成本低；

Modbus缺点：实时性差（从机不能主动上报数据，必须依赖主机轮询），总线利用率低，传输速率低，另外RS485抗干扰较弱，节点错误是会影响整个总线。如果Modbus不能满足自己的需求，可以使用CAN总线。