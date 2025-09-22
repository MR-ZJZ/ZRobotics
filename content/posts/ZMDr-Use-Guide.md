---
title: "驱动使用指南"
date : 2025-09-12T03:25:53+08:00
categories : ["电机驱动"]
tags : ["用户指南"]
---

### 简介
电机驱动是机器人技术中软件硬件桥梁。项目针对永磁同步电机（PMSM）伺服控制，从实现**国产化替代**，**降低使用门槛**，**提高安全性**等角度设计完整的伺服软硬件系统。
<br>

### 驱动硬件
#### 版本
|     Power      |     Joint     |    General    |
| :------------: | :-----------: | :-----------: |
| **48V / 120A** | **48V / 60A** | **48V / 30A** |

Power采用多叠层设计，功率层为铝基板，具备较大功率容量。Joint为关机电机设计，搭配可离轴编码器和双编码控制方式以及刹车控制器。General为基础版本，适用于大多数场合和学习。
<br>

#### 接口
|  USB   |   CAN    |   TTL    |   IO   |  SPI   |
| :----: | :------: | :------: | :----: | :----: |
| 上位机 | 运行控制 | 简易通信 | 自定义 | 编码器 |

虚拟串口需要用于与上位机通信，进行调试和校准。CAN通信为主要通信方式，**具备最高的优先级**，运用于
轨迹跟踪、力矩控制等要求较高的场合。另外提供UART接口，SPI接口。

### 软件参数
#### 用户参数
用户参数需要用户给定，包含电机的特性，校准的阈值、控制增益、控制阈值等，可保存至Flash。
| 参数名称 | UART CMD ID | CAN CMD ID |     范围     |                                       含义                                       |
| :------: | :---------: | :--------: | :----------: | :------------------------------------------------------------------------------: |
| 无感控制 |     ses     |    0x01    |  [0,1] int   |                             0表示禁用无感，反之启用                              |
|  编码器  |     enc     |    0x02    |  [0,6] int   |     0:as5047p, 1:mt6816, 2:mt6825, 3:mt6835, 4:ma600, 5:icmu150, 6:admt4000      |
| 线性补偿 |     enl     |    0x03    |  [0,1] int   |                           0 表示禁用线性补偿，反之启用                           |
| Node ID  |     cid     |    0x04    | [1,0xF] int  |                                  CAN通信节点ID                                   |
| CAN频率  |     cra     |    0x05    |  [0,2] int   |                            0:1MHz, 1:0.5MHz, 2:0.1MHz                            |
| 心跳间隔 |     cbe     |    0x06    | [0,1000] int |                              0表示无心跳，单位为ms                               |
| 电压下限 |     vbd     |    0x07    |   [10,55]    |                       电压低于该值将产生低电压报错，单位V                        |
| 位置上限 |     pou     |    0x08    |  [-1e4,1e4]  |                        位置大于该值将产生越位报错，单位圈                        |
| 位置下限 |     pod     |    0x09    |  [-1e4,1e4]  |                        位置小于该值将产生越位报错，单位圈                        |
| 速度限制 |     vli     |    0x0A    |  [0.01,1e4]  |          运行速度受该值制，速度的绝对值大于该值将产生超速报错，单位rps           |
|  极对数  |     pol     |    0x0B    |  [1,50] int  |          电机极对数，用于编码器校准核验，错误填写将导致极对数不匹配报错          |
| 电流限制 |     ili     |    0x0C    |  [0.1,i_m]   |  运行电流受该值制，电流的绝对值大于该值将产生过电流报错，i_m依据版本而定，单位A  |
| 校准电流 |     icl     |    0x0D    |   [1,120]    |                    电机校准电流、尽量取额定电流的50％，单位A                     |
| 启动模式 |     ste     |    0x0E    |  [0,5] int   |                    启动后自动进入的模式,非0时无需手动使能电机                    |
| 响应模式 |     acm     |    0x0F    |  [0,3] int   |    1: 无反馈立即执行, 2:有反馈立即执行, 3: 无反馈队列执行, 4: 有反馈队列执行     |
| 滤波系数 |     fik     |    0x10    |  [0.001,1]   | 滤波系数，对编码器有效，降低滤波系数可以获得更平滑的速度，同时更容易产生控制震荡 |
| 容忍系数 |     tok     |    0x11    |  [1.1,3.0]   |            允许速度和电流短时间超过限制的裕度，大于该裕度立即产生报错            |
|  位置P   |     p_p     |    0x12    |    [0,50]    |                            位置比例增益，应小于速度环                            |
|  位置I   |     p_i     |    0x13    |   [0,100]    |                            位置积分增益，应小于速度环                            |
|  速度P   |     v_p     |    0x14    |    [0,50]    |                          速度比例增益，依据响应刚性给定                          |
|  速度I   |     v_i     |    0x15    |   [0,100]    |                            速度积分增益，消除静态误差                            |
|  加速度  |     acu     |    0x16    |   [0,1e4]    |           速度和位置运行的的加速度，0表示无穷大，单位圈/s<sup>2</sup>            |
|  减速度  |     acd     |    0x17    |   [0,1e4]    |          速度和位置运行的的减去速度，0表示无穷大，单位圈/s<sup>2</sup>           |
| 队列间隔 |     pvt     |    0x18    |  [-1e3,1e3]  |                       速度位置控制下的每帧间隔时间，单位ms                       |

#### 校准参数
校准参数是驱动器通过自身校准获取的参数，无需用户给定，禁止用户写入(**read-only**)，可保存至Flash。
| 参数名称 | UART CMD ID | CAN CMD ID |   范围    |                                                                                       含义                                                                                       |
| :------: | :---------: | :--------: | :-------: | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| 相间电阻 |     rsi     |    0x19    | read-only |                                                              电机两相间的电阻，用于构建电流，合法范围为[10,12000]mR                                                              |
| 相间电感 |     ind     |    0x1A    | read-only |                                                              电机两相间的电感，用于构建电流，合法范围为[10,10000]uH                                                              |
|   KV值   |     k_v     |    0x1B    | read-only |                                                           电机每伏电压可以达到的转速，用于无感控，合法范围为[10,5000]                                                            |
| 编码方向 |     dir     |    0x1C    | read-only |                                                               编码方向和电机正方向是否相同，1表示相同，-1表示相反                                                                |
| 编码偏移 |     eof     |    0x1D    | read-only |                                                                 实现编码角度和电机电气角度的对齐，范围为 [0,2pi]                                                                 |
| 校准状态 |     cla     |    0x1E    | read-only | 电机当前的校准状态，0: 未校准，1: 仅校准编码器偏移，2: 仅校准KV，3: 已校准编码器偏移和补偿值，4: 已校准编码器偏移和KV，5: 完成所有校准，未完成校准进入对应控制模式将报错非法模式 |

#### 状态参数
状态参数反映电机运行参数，方便用户调试，其不会被保存到Flash，包含用户设置的目标、实际运行值等。
|  参数名称  | UART  CMD ID | CAN CMD ID |    范围    |                                                                                            含义                                                                                            |
| :--------: | :----------: | :--------: | :--------: | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|  当前模式  |     mod      |    0x1F    |   [0,5]    | 0:失能, 1:电流模式, 2:速度模式, 3:位置模式, 4:测试模式, 5:电阻电感校准, 6:编码器线性补偿,7:编码器偏移校准, 8:VK校准, 9:保存配置,10:擦除配置, 11:清除错误, 12: 清除警告, 13: 刹车, 14: 重启 |
|  当前警告  |     war      |    0x20    | read-only  |                                                b0:电压波动, b1:电流满载, b2:油门满载, b3:规划超限制, b4:温度较高, b5:初位变动, b6:非法输入                                                 |
|  当前错误  |     err      |    0x21    | read-only  |            0:无, 1:低电压, 2:过电压, 3:电流不稳, 4:过电流, 5:超速超位, 6:电阻过大, 7:电感过大,  8:编码器错误, 9:极对数不匹配, 10:KV校准失败, 11:非法模式, 12:参数错误, 13:高温             |
|  电流输入  |     iin      |    0x22    | [-120,120] |                                                                                 电流模式下的目标值，单位A                                                                                  |
|  速度输入  |     vin      |    0x23    | [-1e4,1e4] |                                                                                速度模式下的目标值，单位RPS                                                                                 |
|  位置输入  |     pin      |    0x24    | [pou,pod]  |                                                                                 位置模式下的目标值，单位圈                                                                                 |
|  队列控制  |     lif      |    0x25    | [0,16000]  |                                                      万位为启动标志，1表示启动，0表示停止执行。千位以下为队列点个数，可以重置队列点数                                                      |
|  A相电流   |     i_a      |    0x26    | read-only  |                                                                                     电机A相电流，单位A                                                                                     |
|  B相电流   |     i_b      |    0x27    | read-only  |                                                                                     电机B相电流，单位A                                                                                     |
|  C相电流   |     i_c      |    0x28    | read-only  |                                                                                     电机C相电流，单位A                                                                                     |
|   滞后角   |     via      |    0x29    | read-only  |                                                                           电流对电压的滞后角度，范围为[0,0.5pi]                                                                            |
| 输出占空比 |     v_m      |    0x2A    | read-only  |                                                                            输出电压与总电压的比值，范围为[0,1]                                                                             |
|  电流幅值  |     i_m      |    0x2B    | read-only  |                                                                                   电机电流的大小，单位A                                                                                    |
|  编码器值  |     ecr      |    0x2C    | read-only  |                                                                        编码器原始值，范围为[0,cpr-1]，cpr为有效位数                                                                        |
|  当前转速  |     vel      |    0x2D    | read-only  |                                                                                   电机实时转速，单位rps                                                                                    |
|  当前位置  |     pur      |    0x2E    | [-1e4,1e4] |                                                                     电机当前的多圈位置，可写当前位置以重置零点，单位圈                                                                     |
|  磁链估计  |     flk      |    0x2F    | read-only  |                                                                                磁链观测器的估计磁链，单位Wb                                                                                |
|  电源电压  |     vbs      |    0x30    | read-only  |                                                                          供电电压，用于系统控制和过压保护，单位V                                                                           |
|  输入功率  |     pow      |    0x31    | read-only  |                                                                                      供电功率，单位W                                                                                       |
| 驱动器温度 |     tep      |    0x32    | read-only  |                                                                                 驱动器功率管温度，极限100℃                                                                                 |
|  自定接口  |     pwu      |    0x33    | [-1e4,1e4] |                                                                           暂时开放的自定义接口，需定制，默认无效                                                                           |

### 控制要点
#### 部署过程要点
* 无感与有感控制<br>
无感适用与无位置传感器的场合，具备**速度波动小，噪声少**等优势,但无法在零速范围内保持稳定。有感则具备全速范围适应性，但是其存在编码器噪声、非线性安装误差等因素带来的控制精度影响。控制可以根据需求合理选择，可以**同时开启有感和无感**，则驱动器会在合适时机切换位置参考，发挥两者优势。
* 编码非线性补偿<br> 
对于编码器安**装精度不高**或使用不具备**离轴**非线性校准的编码器时，推荐打开线性补偿。但是对于**齿槽转矩**较大的电机，编码补偿可能受到齿槽转矩谐波的影响，导致精度下降。应根据情况合理选择。
* CAN通信设置<br>
若总线其它协议设备时，注意配置合适的节点ID，**避免协议冲突**。最多支持15个节点，节点0为广播帧，所有设备均可通信。可适当降低波特率来降低误码率。可设置CAN心跳反馈频率获取实时位置速度和电流。
* 电机参数设置<br> 
首先确定有无感控制模式、极对数和编码器,设置校准电流和最大电流，**推荐校准电流为额定的一半**。即可以进行参数校准。校准过程中可能产生各类错误，依据错误类型调整。如果无错误，则表面系统不存在问题。可以进入使能控制。
* 控制参数调节<br> 
主要包含增益设置、滤波和容忍设置。增益调节的基本方法为，**首先调整P**，达到预期的响应性能，在适当增大I提高稳态性能，**先调节速度环**，再调节位置环。如电机有嗡嗡噪声，则需降低滤波系数（1表示不滤波），过低的滤波系数可能导致**震荡**，应配合增益合理调节。在极端工况下，需要适当提高容忍系数以避免**冲击导致的误报错**。
#### 运动控制要点
* 力矩控制<br>
力矩模式工作在MIT模式下，因此在扭矩模式给定设定电流的条件下不会持续旋转，若要持续加速，需要设置位置和速度增益为0。扭矩模式的最大速度受到最大转速参数的限制。
* 点动控制<br>
对于速度和位置的简单点动（从一个目标到另一个目标）运动控制，请设置合适加减速，驱动将自动完成规划，过大的加减速将对系统产生冲击。
* 轨迹跟踪<br>
对于位置跟随要求较高的控制场合，有两种方式。其一是将加减速设置为0，同时上层控制器保持较高位置发送频率，实现简单跟随。对于频率受限场合，推荐使用PVT轨迹规划控制。在位置模式下，将队列间隔（pvt）设为为控制帧间隔时间，即启动PVT模式，PVT模式有专用CAN帧格式。
* 队列控制<br>
为降低系统对上层控制器实时性要求，驱动器提供队列缓冲，通过响应模式（acm）设置。轨迹执行时间间隔为队列间隔（pvt）的绝对值，对于速度模式而言，时间间隔的正负无影响。对于位置模式，时间间隔为正将执行轨迹跟踪，为负或0将执行点动。轨迹容量为6000点，超过容量将直接产生参数错误报错。可以通过队列控制（lif）实现启动和队列截取。例如

|       lif=0        |     lif=10000      |      lif=11000       |
| :----------------: | :----------------: | :------------------: |
| 清空队列并停止执行 | 清空队列并开始执行 | 保留1000点并开始执行 |

### 通信协议格式
#### USB串口协议
USB协议采用ASCII通信，格式为
```yaml
                             读写指令 + UART CMD ID + 数据 + 结束符
```
读写段含读和写两种字符指令，分别为“r_”（read）和“w_”（write），数据段为对应写入参数的值，前加“=”，读指令无数据段；结束符为“\n”。
如写极对数为14，
```c
                                    w_pol=14\n
```
为了保证与上位机通信效率，写入不作反馈。可以通过读取确认参数写入是否正确。
```c
                                     r_pol\n
```
驱动器反馈
```c
                                    14.000000\n
```
所有数据应遵循参数表格所给出的参数范围，禁止写入的数据写入指令无效，参数超过范围或者指令格式不正确，指令无效。

#### CAN协议
为充分利用CAN帧格式资源，采用类似CANOpen的CANID段拆分的方式进行通信。通信采用标准帧格式，CAN帧ID段如下描述

|  ID b10~4  |   ID b3~0   |
| :--------: | :---------: |
| CAN CMD ID | CAN NODE ID |

CAN帧DATA段即通信的数据，写入指令DATA段为4字节。如，驱动器节点ID为1，写极对数14，则CAN发送

| ID b10~4 | ID b3~0 | DATA Byte0~3 |
| :------: | :-----: | :----------: |
|   0x0B   |   0x1   |    14.0f     |

读取指令,将DATA段长度设置为0，如读取电机极对数，则CAN发送

| ID b10~4 | ID b3~0 |
| :------: | :-----: |
|   0x0B   |   0x1   |

驱动器反馈

| ID b10~4 | ID b3~0 | DATA Byte0~3 |
| :------: | :-----: | :----------: |
|   0x0B   |   0x1   |    14.0f     |

若通信格式不合法或参数范围错误，指令无效，同时反馈一帧err。
CAN通信具备以下几类特殊格式：

* 参数归一化<br>
为充分利用有限的数据空间，采用目标数据范围与无符号整型数据空间值一一对应的方式通信，称为归一。参数归一化范围如下

|   位置    |    速度    |    电流    | 位置增益 | 速度增益 |
| :-------: | :--------: | :--------: | :------: | :------: |
| [pod,pou] | [-vli,vli] | [-ili,ili] |  [0,50]  |  [0,50]  |

以位置为例，假设位置区间为[-10,100]，位置归一空间为两个字节。则当位置值小于等于-10时，归一位置为0，当位置值大于等于100时，归一位置为0xffff，位置为20时，对应的归一位置为
```yaml
                        (20+10)/(100+10)*0xffff=17873
```
类似的，可以依据归一值计算真值，假设归一值为2000，则真值为
```yaml
                        2000/0xffff*(100+10)-10=-6.643
```

* MIT力矩模式帧格式(**小端在右**)
  
| ID b10~4 | ID b3~0 |  DATA b0~16  | DATA b16~27  | DATA b28~39  | DATA b40~51  | DATA b52~63  |
| :------: | :-----: | :----------: | :----------: | :----------: | :----------: | :----------: |
|   0x38   | NODE ID | 归一目标位置 | 归一目标速度 | 归一目标电流 | 归一位置增益 | 归一速度增益 |

```yaml
         实际电流 = 位置P * (目标位置-实际位置) + 速度P * (目标速度-实际速度) + 目标电流
```

* PVT轨迹跟踪帧格式(**小端在左**)
  
| ID b10~4 | ID b3~0 | DATA Byte0~4 | DATA Byte5~7 |
| :------: | :-----: | :----------: | :----------: |
|   0x39   | NODE ID | 归一目标速度 | 归一目标位置 |

```yaml
                         在队列间隔（pvt）时间后，以目标速度达到目标位置
```

* 心跳的内容(**小端在左**)
  
| ID b10~4 | ID b3~0 | DATA Byte0~3 | DATA Byte4~5 | DATA Byte6~7 |
| :------: | :-----: | :----------: | :----------: | :----------: |
|   0x3A   | NODE ID |   归一位置   |   归一速度   |   归一电流   |

### 上位机
上位机为用户调试电机提供更简单的交互方式，可以方便完成对驱动器的参数设定，校准，状态参数的监测和基本运动控制。上位机软件最大存储约30s的数据，通信频率1Khz。
#### 分区功能
![ZMDr-Tool.png](https://i.postimg.cc/cCTYvH3r/ZMDr-Tool.png)
* 1区：设备选择和参数一键操作 <br>
  下拉选择驱动器设备，**连接到错误设备将会自动断开**。连接成功后会自动更新当前的模式和错误新信息。下方是对参数的一键操作，实现对用户参数的**一键读取、写入、保存、擦除**等。
* 2区：用户参数区<br>
  所有用户需要给定的参数，用户根据自己需求调节，部分参数是点击**下拉窗口**，通过后方三角按钮单独发送该数据，也可以通过一键写入发送全部数据，**不发送则不会修改驱动器数据**，发送后如果需要永久保存需要点击保存参数以保存到FLASH，鼠标放在2区内**滚动滚轮**可以拉出下方隐藏的部分非常参数。
* 3区：模式和状态功能区<br>
  可以设置当前模式，显示和操作当前的模式和错误、警告。“测试模式”用于检测硬件是否异常，非不要不进入。“参数校准”用于自动获取电机参数。“释放电机”将对电机失能。“刹车急停”将迅速锁定当前位置，其将造成冲击，非必要不使用。对于校准数据，功能区也提供了查询接口，也可**写入当前位置和操作队列控制位**。
* 4区：绘图区<br>
  包含三个通道，可以电机通道内的UART CMD ID，在上拉窗口选择对应示波参数，单击三角按钮开始示波，再次单击停止。点击暂停示波后，鼠标放在该区域内滚动滚轮缩放图像，以图像中间为中心缩放，按住滚轮拖动鼠标调整中心的位置，总数据为窗口数据的三倍。
* 5区：运动区<br>
  可以拖动滑条设置目标参数，鼠标在本区内拖动才有效。也可直接在右侧输入框输入后单击按钮发送。滑条的最大最小值可以用户更改，为了避免误操作，**更改后回车才会有效**。

#### 基本操作流程

* 第一步：电机选择设备，选择正确的设备后，点击后面连接图标，其变绿且出现状态信息时表示成功。选择了错误设备将出现“端口不匹配”的提示。

* 第二步：点击“一键读取”，读取设备信息，更新上位机内用户参数。同样可以一键写入将上位机内的用户参数写入驱动器。

* 第三步：根据控制对象设置用户参数的前8项内容，校准电流不宜过低，输出占空比低于3%将无法通过校准。点击3区的参数校准，电机开始运行自动完成校准程序。推荐在校准过程中打开电流、位置等监视信息。如校准过程产生错误，将在3区显示，若无错误，则表示校准完成。

* 第四步：点击“选择模式”后面的下拉窗口选择测试运行的模式，点击后三角按钮实现模式的进入，当前模式将被更新。如在速度模式，则可打开速度监视，在5区中拖动滑条或直接输入目标数据，电机将开始运行。调节适当的PID和滤波条件，达到需求效果。

* 第五步：测试运行完成，点击“保存参数”完成对所有参数的锁存。驱动器自动重启，上位机断开。设备大概需要0.1s时间完成重启，重启完成后CAN会发送一帧自身节点ID消息。

#### Vofa兼容
上位机浮点协议兼容**Vofa**，其具备更强大的数据存储功能，如果需要长时间高速数据监测，可以采用[vofa](https://www.vofa.plus/)，
驱动器可以同时以约1khz的速度打印四个数据。控制方式为
```yaml
                           w_（通道）=（CAN CMD ID）\n
```
例如，通道a打印速度，
```yaml
                                   w_pwa=89\n
```
三个通道分别为pwa、pwb、pwc，pwd，89为0x59（vel）的十进制格式。关闭通道的方式为令通道can id为0。


### 控制举例

#### python-can
利用SLCAN协议，可以使用[Canable](https://canable.io/)工具，将驱动器组网并接入主机系统。
```python
bus = can.interface.Bus(interface='slcan', channel='COM1', bitrate=1000000) # 设置端口

def can_write(node_id, cmd, data):

    msg = can.Message(arbitration_id=(node_id | (cmd << 4)),
                      data=struct.pack('f', data), is_extended_id=False)
    bus.send(msg)
    time.sleep(0.0001)  # 延迟避免阻塞


def can_read(node_id, cmd):

    msg = can.Message(arbitration_id=(node_id | (cmd << 4)), is_extended_id=False)
    bus.send(msg)
    time.sleep(0.0001)  # 延迟避免阻塞


def can_pvt(node_id, v, p, vli, pod, pou):
    
    pos_mod = int((p - pod) / (pou - pod) * 0xffffffff)
    vel_mod = int((v + vli) / (2 * vli) * 0xffffffff)
    msg = can.Message(arbitration_id=(node_id | (0x39 << 4)),
                      data=struct.pack('I', vel_mod) + struct.pack('I', pos_mod),
                      is_extended_id=False)
    bus.send(msg)
    time.sleep(0.0001)  # 延迟避免阻塞


def can_mit(node_id, p, v, i, kp, kv, pod, pou, vli, ili):
    
    pos_mod = int((p - pod) / (pou - pod) * 0xffff)
    vel_mod = int((v + vli) / (2 * vli) * 0xfff)
    cur_mod = int((i + ili) / (2 * ili) * 0xfff)
    kp_mod = int(kp / 50 * 0xfff)
    kv_mod = int(kv / 50 * 0xfff)

    msg = can.Message(arbitration_id=(node_id | (0x38 << 4)),
                      data=struct.pack('B', pos_mod >> 8)
                           + struct.pack('B', pos_mod & 0xff)
                           + struct.pack('B', vel_mod >> 4)
                           + struct.pack('B', ((vel_mod & 0xf) << 4) | (cur_mod >> 8))
                           + struct.pack('B', cur_mod & 0xff)
                           + struct.pack('B', kp_mod >> 4)
                           + struct.pack('B', ((kp_mod & 0xf) << 4) | (kv_mod >> 8))
                           + struct.pack('B', kv_mod & 0xff), is_extended_id=False)
    bus.send(msg)
    time.sleep(0.0001)  # 延迟避免阻塞


def go_position():  # Press the key "space" to actio

    global pos
    pos = 0
    can_write(0xf, mod_id, 11)  # 清除错误
    can_write(0xf, pur_id, 0)  # 设定当前位置为0
    # 可以自定再添加用户设定，比如加减速度、电流限制等等
    can_write(0xf, mod_id, 3)  # 进入位置模式
    time.sleep(0.02)
    print(1)
    while True:
        if pos < 2 * math.pi:
            # 对多个节点输入正弦目标位置
            for i in id:
                can_write(i, pin_id, math.sin(pos))
        else:
            # 释放电机
            can_write(0, mod_id, 0)

            # 读参数验证,id0为广播帧
            for i in range(10):
                can_read(0xf, i + 1)
                time.sleep(0.2)

        pos += 0.004
        time.sleep(0.002)


def motor_pvt():

    delta_t = 0.002  # 控制时间间隔
    num = 3.14 / delta_t + 1  # 轨迹点数
    num_data = num
    tt = 0  # 轨迹时间统计参数
    can_write(0xf, mod_id, 11)  # 清除错误
    can_write(0xf, pur_id, 0)  # 设定当前位置为0
    can_write(0xf, pvt_id, delta_t)  # 设置时间间隔
    can_write(0xf, acm_id, 2)  # 设置为无反馈队列模式
    can_write(0xf, lif_id, 0)  # 停止队列执行并清空队列
    can_write(0xf, mod_id, 3)  # 进入位置模式

    while True:
        if num_data > 0:
            # 在队列存储20个时启动队列，相当于给你20个缓冲点，队列边进边出
            if num_data == num - 20:
                can_write(0xf, lif_id, 10000 + 19)
            # 轨迹方程
            v = 0.5 * (math.sin(2 * tt)) * 2
            p = -0.5 * (math.cos(2 * tt) - 1)
            # 发送指令，注意归一范围与预存参数对应，自行设定目标节点
            for i in id:
                can_pvt(i, v, p, vli, pod, pou)
            # 队列模式指令发送无需过多等待，若为直接执行模式，此处延迟必须为delta_t
            time.sleep(0.001)
            # 轨迹统计更新
            tt = tt + delta_t
            num_data = num_data - 1

        elif num_data == 0:
            # 等待轨迹响应完成
            time.sleep(2.5)
            # 读参数验证,id0为广播帧
            for i in range(10):
                can_read(0xf, 2 * i + 1)
                time.sleep(0.2)
            # 重置轨迹
            tt = 0
            num_data = num


def motor_mit():

    tt = 0  # 轨迹时间统计参数
    cur = 1  # 目标电流为10A
    delta_t = 0.01  # 控制时间间隔
    num = 3.14 / delta_t + 1  # 轨迹点数
    num_data = num
    can_write(0xf, mod_id, 11)  # 清除错误
    can_write(0xf, pur_id, 0)  # 设定当前位置为0
    can_write(0xf, mod_id, 1)  # 进入扭矩模式

    while True:
        if num_data > 0:
            # 轨迹方程
            v = 0.5 * (math.sin(2 * tt)) * 2
            p = -0.5 * (math.cos(2 * tt) - 1)
            # 发送指令，注意归一范围与预存参数对应，自行设定目标节点
            for i in id:
                can_mit(i, p, v, cur, 5, 2, pod, pou, vli, ili)
            # 轨迹统计更新
            time.sleep(delta_t)
            tt = tt + delta_t
            num_data = num_data - 1

        elif num_data == 0:
            # 读参数验证,id0为广播帧
            for i in range(10):
                can_read(0xf, 2 * i + 1)
                time.sleep(0.2)
            # 重置轨迹
            tt = 0
            num_data = num
            time.sleep(delta_t)


def can_receive():

    while True:
        time.sleep(0.001)
        msg = bus.recv()
        if msg is not None and not msg.is_extended_id:  # 收到标准帧消息
            if msg.dlc == 4:  # 参数读取反馈信息
                print("节点：", hex(msg.arbitration_id & 0xf),
                      "指令：", msg.arbitration_id >> 4,
                      "数据：", struct.unpack("f", msg.data[0:])[0])
            elif msg.dlc == 8:  # 心跳反馈信息
                print("位置：", struct.unpack("I", msg.data[0:4])[0] / float(0xffffffff) * (pou - pod) + pod,
                      "速度：", struct.unpack("H", msg.data[4:6])[0] / float(0xffff) * (2 * vli) - vli,
                      "电流：", struct.unpack("H", msg.data[6:8])[0] / float(0xffff) * (2 * ili) - ili, )

```

#### stm32-hal
针对常见的单片机控制系统，以STM32为例。
```c
//请先自定义CAN发送和接收函数接口
uint8_t can_SendPacket(FDCAN_HandleTypeDef *can,uint32_t id, uint8_t *_DataBuf, uint8_t _Len);

void can_write(uint8_t node_id, uint8_t cmd, float data) {
    uint16_t can_id = node_id | (cmd << 4);
    can_SendPacket(&hfdcan1, can_id, (uint8_t *) &data, 4);
    osDelay(1);// 延迟0.1ms避免阻塞
}

void can_read(uint8_t node_id, uint8_t cmd) {
    uint16_t can_id = node_id | (cmd << 4);
    can_SendPacket(&hfdcan1, can_id, NULL, 0);
    osDelay(1);// 延迟0.1ms避免阻塞
}

void can_pvt(uint8_t node_id, float v, float p, float vli, float pod, float pou) {
    uint32_t pos_mod = (uint32_t) ((p - pod) / (pou - pod) * (float) 0xffffffff);
    uint32_t vel_mod = (uint32_t) ((v + vli) / (2 * vli) * (float) 0xffffffff);
    uint8_t pv_mod[8] = {0};
    memcpy(pv_mod, &vel_mod, 4);
    memcpy(pv_mod + 4, &pos_mod, 4);

    uint16_t can_id = node_id | (0x39 << 4);
    can_SendPacket(&hfdcan1, can_id, (uint8_t *) &pv_mod, 8);
    osDelay(1);// 延迟0.1ms避免阻塞
}

void can_mit(uint8_t node_id, float p, float v, float i, float kp, float kv, float pod, float pou, float vli, float ili) {
    uint16_t pos_mod = (uint16_t) ((p - pod) / (pou - pod) * (float) 0xffff);
    uint16_t vel_mod = (uint16_t) ((v + vli) / (2 * vli) * (float) 0xfff);
    uint16_t cur_mod = (uint16_t) ((i + ili) / (2 * ili) * (float) 0xfff);
    uint16_t kp_mod = (uint16_t) ((kp / 50) * (float) 0xfff);
    uint16_t kv_mod = (uint16_t) ((kv / 50) * (float) 0xfff);

    uint8_t mit_mod[8] = {pos_mod >> 8, pos_mod & 0xff,
                          vel_mod >> 4, ((vel_mod & 0xf) << 4) | (cur_mod >> 8), cur_mod & 0xff,
                          kp_mod >> 4, ((kp_mod & 0xf) << 4) | (kv_mod >> 8), kv_mod & 0xff};

    uint16_t can_id = node_id | (0x38 << 4);
    can_SendPacket(&hfdcan1, can_id, (uint8_t *) &mit_mod, 8);
    osDelay(1);// 延迟0.1ms避免阻塞
}

void go_position(void) {
    float pos = 0;
    can_write(0xf, mod_id, 11);  // 清除错误
    can_write(0xf, pur_id, 0);  // 设定当前位置为0
    //可以自定再添加用户设定，比如加减速度、电流限制等等
    can_write(0xf, mod_id, 3);  // 进入位置模式
    while (1) {
        if (pos < 2 * 3.14159f) {
            // 对多个节点输入正弦目标位置
            for (int i = 0; i < sizeof(id); i++) {
                can_write(id[i], pin_id, arm_sin_f32(pos));
            }
        } else {
            // 释放电机,id0为广播帧
            can_write(0xf, mod_id, 0);
            // 读参数验证
            for (int i = 0; i < 10; i++) {
                can_read(0xf, 2 * i + 1);
                osDelay(1000);
            }
        }
        pos += 0.004f;
        //以RTOS为例
        osDelay(20);
    }
}

void motor_pvt(void) {
    float delta_t = 0.002f;  // 控制时间间隔
    float num = 3.14f / delta_t + 1;  // 轨迹点数
    float num_data = num;
    float tt = 0;  // 轨迹时间统计参数
    can_write(0xf, mod_id, 11); // 清除错误
    can_write(0xf, pur_id, 0);  // 设定当前位置为0
    can_write(0xf, pvt_id, delta_t);  // 设置时间间隔
    can_write(0xf, acm_id, 2);  // 设置为无反馈队列模式
    can_write(0xf, lif_id, 0);  // 停止队列执行并清空队列
    can_write(0xf, mod_id, 3);  // 进入位置模式

    while (1) {
        if (num_data > 0) {
            // 在队列存储20个时启动队列，相当于给你20个缓冲点，队列边进边出
            if (num_data == num - 20)
                can_write(0xf, lif_id, 10000 + 19);
            // 轨迹方程
            float v = 0.5f * (arm_sin_f32(2 * tt)) * 2;
            float p = -0.5f * (arm_cos_f32(2 * tt) - 1);
            // 发送指令，注意归一范围与预存参数对应，自行设定目标节点
            for (int i = 0; i < sizeof(id); i++) {
                can_pvt(id[i], v, p, vli, pod, pou);
            }
            // 队列模式指令发送无需过多等待，若为直接执行模式，此处延迟必须为delta_t
            osDelay(10);
            // 轨迹统计更新
            tt = tt + delta_t;
            num_data = num_data - 1;
        } else if (num_data == 0) {
            // 等待轨迹响应完成
            osDelay(25000);
            // 读参数验证
            for (int i = 0; i < 10; i++) {
                can_read(0xf, 2 * i + 1);
                osDelay(1000);
            }
            // 重置轨迹
            tt = 0;
            num_data = num;
        }
    }
}

void motor_mit(void) {
    float tt = 0;  // 轨迹时间统计参数
    float cur = 1;  // 目标电流为10A
    float delta_t = 0.01f;  // 控制时间间隔
    int num = (int)(3.14f * 2 / delta_t + 1);  // 轨迹点数
    int num_data = num;
    can_write(0xf, mod_id, 11);  // 清除错误
    can_write(0xf, pur_id, 0);  // 设定当前位置为0
    can_write(0xf, mod_id, 1);  // 进入扭矩模式

    while (1) {
        if (num_data > 0) {
            // 轨迹方程
            float v = 0.5f * (arm_sin_f32(2 * tt)) * 2;
            float p = -0.5f * (arm_cos_f32(2 * tt) - 1);
            // 发送指令，注意归一范围与预存参数对应，自行设定目标节点
            for (int i = 0; i < sizeof(id); i++) {
                can_mit(id[i], p, v, cur, 5, 2, pod, pou, vli, ili);
            }
            // 轨迹统计更新
            osDelay((int) (delta_t * 10000));
            tt = tt + delta_t;
            num_data = num_data - 1;
        } else if (num_data == 0) {
            // 读参数验证
            for (int i = 0; i < 10; i++) {
                can_read(0xf, 2 * i + 1);
                osDelay(1000);
            }
            // 重置轨迹
            tt = 0;
            num_data = num;
            osDelay((int) (delta_t * 10000));
        }
    }
}

//置于接收中断中
void can_receive(void) {
    FDCAN_RxHeaderTypeDef FDCAN1_RxHeader;
    uint8_t can_buf[8] = {0};
    // 接收数据并再次开启中断触发
    HAL_FDCAN_GetRxMessage(&hfdcan1, FDCAN_RX_FIFO0, &FDCAN1_RxHeader, can_buf);
    HAL_FDCAN_ActivateNotification(&hfdcan1, FDCAN_IT_RX_FIFO0_NEW_MESSAGE, 0);

    if (FDCAN1_RxHeader.IdType == FDCAN_STANDARD_ID) {
        if (FDCAN1_RxHeader.DataLength >> 16 == 4) {  // 参数读取反馈信息
            printf("节点：%lx, 指令：%lx, 数据：%f\n",
                   FDCAN1_RxHeader.Identifier & 0xf,
                   FDCAN1_RxHeader.Identifier >> 4,
                   *(float *) can_buf);
        }
        if (FDCAN1_RxHeader.DataLength >> 16 == 8) {  // 心跳反馈信息
            printf("位置：%f, 速度：%f, 电流：%f\n",
                   (float) *(uint32_t *) can_buf / (float) 0xffffffff * (pou - pod) + pod,
                   (float) *(uint16_t *) (can_buf + 4) / (float) 0xffff * (2 * vli) - vli,
                   (float) *(uint16_t *) (can_buf + 6) / (float) 0xffff * (2 * ili) - ili);
        }
    }
}
```

### 致谢
启蒙参考：ODrive，VESC<br>
硬件设计：矛盾聚合体、滚筒洗衣机<br>
软件开发：Turing、AMO、Jdhfusk<br>