# 板卡硬件与引脚参考

> 适用对象：基于 Xilinx Kintex-7 `XC7K325T-2FFG900I` 的 EDABOX/FPGA 数字孪生平台板卡。  
> 主要用途：为后续 AI、硬件工程师和驱动开发人员提供可检索的板级引脚、电气限制以及 HC-SR04、GY-302/BH1750、四针 I²C OLED 接线依据。  
> 页码规则：`PDF p.x` 表示从 PDF 首页开始计算的实际页码。  
> 结论边界：资料没有明确给出的电压、地址、控制器型号和电流能力均标记为“待确认”，不凭经验补全。

## 1. 板卡概述

| 项目 | 已确认信息 | 来源 |
|---|---|---|
| FPGA | Xilinx Kintex-7 `XC7K325T-2FFG900I`，FFG900 封装，速度等级 -2，工业级 | [S1, 多页]；[S2, PDF p.2-p.7] |
| 板卡/工程标识 | 原理图 PDF 元数据为 `edabox2.4`；FunctionPin 报告页眉中的 Design Name 指向 `EDABOX2.2/EDABOX2.1.brd` | [S1, PDF p.1]；[S2, PDF 元数据] |
| 处理器 | FPGA 本身没有硬核 ARM；CPU/SoC 由 bitstream 中的软核和总线系统实现 | 器件型号与工程性质 |
| 主时钟 | 200 MHz 差分时钟，`SYSCLK_P=AD12`、`SYSCLK_N=AD11`；振荡器标识 `SiT9102AC-241N33E-200MHz` | [S1, PDF p.3]；[S2, PDF p.5] |
| 配置 Flash | `S25FL256SAGNFI00`，四线 SPI 配置 Flash | [S2, PDF p.2] |
| DDR | 两组 DDR3 颗粒构成 64 bit 数据总线；Bank 32/33/34，VCCO=1.5 V | [S1, PDF p.1-p.9]；[S2, PDF p.5、p.9-p.10] |
| 用户扩展口 | J7、J8、J9、J10，共 40 根 `DEBUG_1..40` FPGA I/O | [S1, PDF p.12-p.15]；[S2, PDF p.17] |
| 板载接口 | JTAG、QSPI、DDR、千兆以太网、CP2104 USB-UART、HDMI、VGA、TF 卡、4 路 USB Host | [S2, PDF p.2、p.8-p.15] |
| 板载人机资源 | 64 位拨码开关、8 个按键、32 个 LED、4 组双位数码管、蜂鸣器 | [S1, PDF p.9-p.30]；[S2, PDF p.13、p.16] |

### 1.1 当前已有外设模块

| 模块 | 图片可确认信息 | 待确认信息 |
|---|---|---|
| HC-SR04 | `VCC/TRIG/ECHO/GND`；4.5-5.5 V；工作电流约 5 mA；TRIG 脉冲至少 10 us；TRIG/ECHO 各有 10 kΩ 内部上拉；2-450 cm | 不同批次 ECHO 输出结构和实际误差 [I1] |
| GY-302/BH1750 | I²C；`VCC/GND/SCL/SDA/ADDR`；ADDR 低为 `0x23`，高为 `0x5C` | 模块允许供电范围、板载上拉连接方式 [I3]、[I4] |
| 四针 OLED | 0.96 英寸、蓝色、I²C；丝印 `GND/VCC/SCL/SDA` | 控制器型号、分辨率、I²C 地址、供电范围 [I2] |

## 2. 电气特性与使用限制

### 2.1 FPGA I/O Bank 电压

| I/O Bank | FunctionPin 分组 | 板上 VCCO | 主要用途 | 来源 |
|---:|---|---|---|---|
| 12 | F42 | VADJ2 | USB OTG ULPI | [S1, PDF p.31]；[S2, PDF p.7、p.15] |
| 13 | F41 | VADJ1 | LED3/LED4 数码管、USB reset 等 | [S1, PDF p.29-p.31]；[S2, PDF p.7、p.13] |
| 14 | F40 | 3.3 V | SW_1..41、配置 Flash | [S1, PDF p.23-p.29]；[S2, PDF p.2、p.7、p.16] |
| 15 | F39 | VADJ1 | SW_42..64、VGA | [S1, PDF p.20-p.23]；[S2, PDF p.7、p.14、p.16] |
| 16 | F38 | VADJ1 | LED_1..32、HDMI、TF 卡 | [S1, PDF p.15-p.20]；[S2, PDF p.7-p.8、p.13] |
| 17 | F37 | VADJ1 | `DEBUG_1..40`、CP2104、部分 Ethernet | [S1, PDF p.12-p.15]；[S2, PDF p.7、p.11-p.12、p.17] |
| 18 | F36 | 3.3 V | KEY1..8、LED1/LED2 数码管、部分 Ethernet | [S1, PDF p.9-p.12]；[S2, PDF p.7、p.11、p.13、p.16] |
| 32/33/34 | F35/F34/F33 | 1.5 V | DDR3 | [S1, PDF p.1-p.9]；[S2, PDF p.5、p.7、p.9-p.10] |
| 配置 Bank 0 | F32 | 3.3 V | JTAG、配置 Flash、DONE/PROGRAM_B | [S1, PDF p.1]；[S2, PDF p.2、p.7] |

### 2.2 DEBUG 扩展口的关键限制

- J7-J10 的 pin 20 明确接固定 `+3.3V`，pin 11-19 接 GND。[S2, PDF p.17]
- `DEBUG_1..40` 全部来自 FPGA Bank 17，而 Bank 17 的 VCCO 是 `VADJ1`，不是原理图上直接标明的固定 3.3 V。[S2, PDF p.7]
- 电源页只给出 VADJ1 稳压器设计能力 3 A，没有在文档中写明 VADJ1 的实际设定电压。[S2, PDF p.18]
- 因此，**不得仅因为 J7-J10 pin 20 是 3.3 V，就直接认定 DEBUG 信号一定是 LVCMOS33**。上板前应测量 VADJ1 测试点 TP5 对 GND 的电压，并核对当前 XDC 的 IOSTANDARD。
- 每根 DEBUG 信号串联 100 Ω，并连接 GSOT15C-HG3-08 TVS 到地。这有助于限流、阻尼和 ESD 防护，但不是 5 V 到 3.3 V 电平转换器。[S2, PDF p.17]
- 文档没有给出 DEBUG 单脚最大驱动电流、单个扩展口电源限流或允许热插拔条件。不得用 FPGA I/O 直接驱动电机、继电器或大功率负载。

### 2.3 板上电源

| 电源网 | 原理图标注能力/用途 | 是否在 DEBUG 排针提供 | 注意事项 | 来源 |
|---|---|---|---|---|
| DC12V_IN/VCC12V | 主电源入口；经电源开关和 3 A 保险路径 | 否 | 文档未给允许输入范围，只能确认标称 12 V | [S2, PDF p.18] |
| +5V/VCC5V | 稳压器标注 5.0 V、6 A；供 USB 等 | 否 | 不是 J7-J10 用户电源脚；HC-SR04 需另找经确认的 5 V 接点或外部 5 V | [S2, PDF p.15、p.18] |
| +3.3V/VCC3V3 | 稳压器标注 3.3 V、3 A；J7-J10 pin 20 | 是 | 3 A 是整条电源轨设计标注，不等于单个排针可持续输出 3 A | [S2, PDF p.17-p.18] |
| VADJ1 | Bank 13/15/16/17；稳压器标注 3 A | 不作为独立电源针脚引出 | 精确电压待测；DEBUG I/O 电平由它决定 | [S2, PDF p.7、p.18] |
| VADJ2 | Bank 12；稳压器标注 3 A | 否 | 精确电压待确认 | [S2, PDF p.7、p.18] |
| 1.0/1.2/1.35/1.5/1.8/2.0 V | FPGA 内核、GTX、DDR、辅助电源 | 否 | 不用于普通传感器供电 | [S2, PDF p.1、p.7、p.18] |

### 2.4 ADC 与模拟信号

- 两份资料没有给出可供用户接线的 XADC 专用模拟排针。
- FPGA 的 `VP_0/VN_0`、`VREFP_0/VREFN_0` 出现在配置/模拟专用引脚块中，但没有连接到用户连接器。[S2, PDF p.2]
- Bank 15 有多组带 `ADxP/ADxN` 能力的辅助模拟管脚，但板上已接 `SW_43..64` 拨码开关，且使用 3.3 V 上拉，没有模拟输入调理电路。[S1, PDF p.20-p.23]；[S2, PDF p.16]
- 因此本板没有经文档确认、可直接接模拟光照模块 AO 的安全 ADC 入口。模拟传感器应增加外部 I²C/SPI ADC 模块，或在完成原理图、电压范围和 XADC 保护设计后再使用。

## 3. 接口与连接器位置说明

| 连接器 | 类型/用途 | 主要信号 | 是否适合普通传感器 | 来源 |
|---|---|---|---|---|
| J7 | 2x10 DEBUG 扩展口 | DEBUG_1..10、GND、3.3 V | 是，先确认 VADJ1 | [S2, PDF p.17] |
| J8 | 2x10 DEBUG 扩展口 | DEBUG_11..20、GND、3.3 V | 是，先确认 VADJ1 | [S2, PDF p.17] |
| J9 | 2x10 DEBUG 扩展口 | DEBUG_21..30、GND、3.3 V | 是，先确认 VADJ1 | [S2, PDF p.17] |
| J10 | 2x10 DEBUG 扩展口 | DEBUG_31..40、GND、3.3 V | 是；当前传感器推荐口 | [S2, PDF p.17] |
| J1 | 2x5 JTAG | TCK/TDO/TMS/TDI、3.3 V、GND | 否，下载调试专用 | [S2, PDF p.2] |
| J2 | HDMI Type-A | TMDS、CEC、DDC、HPD、5 V | 否，HDMI 专用 | [S2, PDF p.8] |
| J4 | Micro-USB | CP2104 USB-UART | 否，电脑串口专用 | [S2, PDF p.12] |
| J5 | TF/SD 卡座 | DAT0-3、CMD、CLK、CD | 否，存储卡专用 | [S2, PDF p.13] |
| J6 | DB15 VGA | RGB、HS、VS、GND | 否，显示专用 | [S2, PDF p.14] |
| J11/J12 | 双层 USB-A | 4 路 USB Host | 否，USB 专用 | [S2, PDF p.15] |
| CN1 | DC Jack | DC12V_IN、GND | 仅板卡供电 | [S2, PDF p.18] |
| J13 | 2x5 电源调试口 | DC12V_IN、GND、STP_debug_1..6 | 否，电源调试/控制专用 | [S2, PDF p.18] |
| H1 | 3 pin 风扇口 | VCC12V、GND、另一路待确认 | 否，风扇专用 | [S2, PDF p.13] |

## 4. 全部引脚总表

### 4.1 J7：DEBUG_1..10

| 接口 | 物理引脚 | GPIO/网络名 | FPGA 封装脚 | 默认功能 | 复用功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| J7 | 1 | DEBUG_1 | G17 | 通用扩展 I/O | GPIO/I²C/SPI/UART/PWM/定时器/中断 | VADJ1 | 100 Ω + TVS | bitstream/XDC 定义 | [S1, PDF p.13]；[S2, PDF p.17] |
| J7 | 2 | DEBUG_2 | G18 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.12]；[S2, PDF p.17] |
| J7 | 3 | DEBUG_3 | F17 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J7 | 4 | DEBUG_4 | H20 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J7 | 5 | DEBUG_5 | F18 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.12]；[S2, PDF p.17] |
| J7 | 6 | DEBUG_6 | A16 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J7 | 7 | DEBUG_7 | E18 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.12]；[S2, PDF p.17] |
| J7 | 8 | DEBUG_8 | A17 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J7 | 9 | DEBUG_9 | G19 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.12]；[S2, PDF p.17] |
| J7 | 10 | DEBUG_10 | G20 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J7 | 11-19 | GND | - | 地 | - | 0 V | 9 根地线 | 可任选一根共地 | [S2, PDF p.17] |
| J7 | 20 | +3.3V | - | 电源 | - | 3.3 V | 板上 3.3 V 电源轨 | 单口允许电流待确认 | [S2, PDF p.17-p.18] |

### 4.2 J8：DEBUG_11..20

| 接口 | 物理引脚 | GPIO/网络名 | FPGA 封装脚 | 默认功能 | 复用功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| J8 | 1 | DEBUG_11 | C19 | 通用扩展 I/O | GPIO/I²C/SPI/UART/PWM/定时器/中断 | VADJ1 | 100 Ω + TVS | bitstream/XDC 定义 | [S1, PDF p.14]；[S2, PDF p.17] |
| J8 | 2 | DEBUG_12 | B19 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J8 | 3 | DEBUG_13 | B18 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J8 | 4 | DEBUG_14 | A18 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J8 | 5 | DEBUG_15 | A20 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J8 | 6 | DEBUG_16 | H21 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.15]；[S2, PDF p.17] |
| J8 | 7 | DEBUG_17 | C20 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J8 | 8 | DEBUG_18 | H22 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.15]；[S2, PDF p.17] |
| J8 | 9 | DEBUG_19 | B20 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J8 | 10 | DEBUG_20 | B22 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J8 | 11-19 | GND | - | 地 | - | 0 V | 9 根地线 | 可任选一根共地 | [S2, PDF p.17] |
| J8 | 20 | +3.3V | - | 电源 | - | 3.3 V | 板上 3.3 V 电源轨 | 单口允许电流待确认 | [S2, PDF p.17-p.18] |

### 4.3 J9：DEBUG_21..30

| 接口 | 物理引脚 | GPIO/网络名 | FPGA 封装脚 | 默认功能 | 复用功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| J9 | 1 | DEBUG_21 | K20 | 通用扩展 I/O | GPIO/I²C/SPI/UART/PWM/定时器/中断 | VADJ1 | 100 Ω + TVS | bitstream/XDC 定义 | [S1, PDF p.15]；[S2, PDF p.17] |
| J9 | 2 | DEBUG_22 | K19 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.15]；[S2, PDF p.17] |
| J9 | 3 | DEBUG_23 | L18 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J9 | 4 | DEBUG_24 | L17 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.15]；[S2, PDF p.17] |
| J9 | 5 | DEBUG_25 | K18 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J9 | 6 | DEBUG_26 | J18 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.13]；[S2, PDF p.17] |
| J9 | 7 | DEBUG_27 | J17 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J9 | 8 | DEBUG_28 | J19 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J9 | 9 | DEBUG_29 | H17 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J9 | 10 | DEBUG_30 | H19 | 同上 | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.14]；[S2, PDF p.17] |
| J9 | 11-19 | GND | - | 地 | - | 0 V | 9 根地线 | 可任选一根共地 | [S2, PDF p.17] |
| J9 | 20 | +3.3V | - | 电源 | - | 3.3 V | 板上 3.3 V 电源轨 | 单口允许电流待确认 | [S2, PDF p.17-p.18] |

### 4.4 J10：DEBUG_31..40

| 接口 | 物理引脚 | GPIO/网络名 | FPGA 封装脚 | 当前推荐功能 | 复用功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| J10 | 1 | DEBUG_31 | A22 | HC-SR04 TRIG 前端控制 | GPIO/PWM/定时器/中断 | VADJ1 | 100 Ω + TVS | 必须隔离 5 V 上拉 | [S1, PDF p.14]；[S2, PDF p.17] |
| J10 | 2 | DEBUG_32 | A21 | HC-SR04 ECHO 输入 | GPIO/定时器捕获/中断 | VADJ1 | 100 Ω + TVS | ECHO 先降压 | [S1, PDF p.13]；[S2, PDF p.17] |
| J10 | 3 | DEBUG_33 | C21 | 备用 GPIO | GPIO/I²C/SPI/UART/PWM/中断 | VADJ1 | 100 Ω + TVS | bitstream/XDC 定义 | [S1, PDF p.15]；[S2, PDF p.17] |
| J10 | 4 | DEBUG_34 | D21 | 备用 GPIO | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.15]；[S2, PDF p.17] |
| J10 | 5 | DEBUG_35 | C22 | 备用 GPIO | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.12]；[S2, PDF p.17] |
| J10 | 6 | DEBUG_36 | E21 | 备用 GPIO | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.12]；[S2, PDF p.17] |
| J10 | 7 | DEBUG_37 | D22 | 备用 GPIO | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.12]；[S2, PDF p.17] |
| J10 | 8 | DEBUG_38 | F21 | 备用 GPIO | 同上 | VADJ1 | 100 Ω + TVS | 同上 | [S1, PDF p.12]；[S2, PDF p.17] |
| J10 | 9 | DEBUG_39 | F22 | 当前传感器 I²C SCL | I²C/GPIO | VADJ1 | 100 Ω + TVS | 开漏；外部上拉电压须匹配 VADJ1 | [S1, PDF p.15]；[S2, PDF p.17] |
| J10 | 10 | DEBUG_40 | G22 | 当前传感器 I²C SDA | I²C/GPIO | VADJ1 | 100 Ω + TVS | 开漏；外部上拉电压须匹配 VADJ1 | [S1, PDF p.15]；[S2, PDF p.17] |
| J10 | 11-19 | GND | - | 传感器地 | - | 0 V | 9 根地线 | HC-SR04 外部 5 V 也必须共地 | [S2, PDF p.17] |
| J10 | 20 | +3.3V | - | OLED/GY-302 电源 | - | 3.3 V | 板上 3.3 V 电源轨 | 不用于 HC-SR04 VCC | [S2, PDF p.17-p.18] |

### 4.5 JTAG、配置与串口

| 接口 | 物理引脚/网络 | FPGA 封装脚 | 默认功能 | 电压/限制 | 来源 |
|---|---|---|---|---|---|
| J1 | pin 1 JTAG_TCK | E10 | JTAG 时钟 | 3.3 V，33 Ω 串联 | [S1, PDF p.1]；[S2, PDF p.2] |
| J1 | pin 3 JTAG_TDO | G10 | JTAG 数据输出 | 3.3 V，33 Ω 串联 | 同上 |
| J1 | pin 5 JTAG_TMS | F10 | JTAG 模式 | 3.3 V，33 Ω 串联 | 同上 |
| J1 | pin 7 | - | 未连接 | 待确认 | [S2, PDF p.2] |
| J1 | pin 9 JTAG_TDI | H10 | JTAG 数据输入 | 3.3 V，33 Ω 串联 | [S1, PDF p.1]；[S2, PDF p.2] |
| J1 | pin 2/4/6 | - | VCC3V3 | 3.3 V | [S2, PDF p.2] |
| J1 | pin 8/10 | - | GND | 0 V | [S2, PDF p.2] |
| 配置 | FLASH_CLK | B10 | CCLK/QSPI clock | 板载占用 | [S1, PDF p.1]；[S2, PDF p.2] |
| 配置 | FLASH_IO0/1/2/3 | P24/R25/R20/R21 | QSPI data | 板载占用 | [S1, PDF p.26、p.28]；[S2, PDF p.2] |
| 配置 | FLASH_NCS | U19 | QSPI chip select | 板载占用 | [S1, PDF p.29]；[S2, PDF p.2] |
| 配置 | DONE | M10 | 配置完成 | 板载 DONE 指示电路 | [S1, PDF p.1]；[S2, PDF p.2] |
| 配置 | PROGRAM_B | K10 | FPGA 重配置 | 板载上拉；不是普通 GPIO | [S1, PDF p.1]；[S2, PDF p.2] |
| USB-UART | CP2104_TXD | D18 | USB 桥 TX -> FPGA RX | Bank17/VADJ1；板载占用 | [S1, PDF p.12]；[S2, PDF p.12] |
| USB-UART | CP2104_RXD | D17 | FPGA TX -> USB 桥 RX | Bank17/VADJ1；板载占用 | [S1, PDF p.12]；[S2, PDF p.12] |

### 4.6 板载按键、拨码开关、LED 和数码管

| 资源 | FPGA 网络 | FPGA 封装脚 | 默认电路/有效电平 | 电压与占用 | 来源 |
|---|---|---|---|---|---|
| K1 | KEY1 | E15 | 2.2 kΩ 上拉至 3.3 V；按下接地，低有效 | Bank18，板载占用 | [S1, PDF p.10]；[S2, PDF p.16] |
| K2 | KEY2 | C14 | 同上 | Bank18，板载占用 | [S1, PDF p.10]；[S2, PDF p.16] |
| K3 | KEY3 | D14 | 同上 | Bank18，板载占用 | [S1, PDF p.10]；[S2, PDF p.16] |
| K4 | KEY4 | A13 | 同上 | Bank18，板载占用 | [S1, PDF p.10]；[S2, PDF p.16] |
| K5 | KEY5 | B13 | 同上 | Bank18，板载占用 | [S1, PDF p.10]；[S2, PDF p.16] |
| K6 | KEY6 | E14 | 同上 | Bank18，板载占用 | [S1, PDF p.10]；[S2, PDF p.16] |
| K7 | KEY7 | C11 | 同上 | Bank18，板载占用 | [S1, PDF p.9]；[S2, PDF p.16] |
| K8 | KEY8 | D11 | 同上 | Bank18，板载占用 | [S1, PDF p.9]；[S2, PDF p.16] |
| SW1-SW64 | SW_1..SW_64 | 见 4.8 完整表 | 每位 4.7 kΩ 上拉到 3.3 V；闭合接地，低有效 | SW_1..41 在 Bank14；SW_42..64 在 Bank15；板载占用 | [S1, PDF p.20-p.29]；[S2, PDF p.16] |
| LED1-LED32 | LED1..LED32 | 见 4.8 完整表 | FPGA -> 470 Ω -> LED -> GND，高有效 | Bank16/VADJ1，板载占用 | [S1, PDF p.15-p.19]；[S2, PDF p.13] |
| 双位数码管 1/2 | LED1_A..G/DP/CS1/CS2、LED2_* | 见 4.8 | 板载 100 Ω 段电阻；扫描极性需结合器件验证 | Bank18/3.3 V | [S1, PDF p.9-p.12]；[S2, PDF p.13] |
| 双位数码管 3/4 | LED3_*、LED4_* | 见 4.8 | 同上 | Bank13/VADJ1 | [S1, PDF p.29-p.30]；[S2, PDF p.13] |
| 蜂鸣器 | BEER | 待见 4.8 | 经 1 kΩ 驱动 S8050 三极管 | 板载占用 | [S2, PDF p.13] |

### 4.7 专用板载接口信号摘要

| 接口 | FPGA 网络与封装脚 | 占用/限制 | 来源 |
|---|---|---|---|
| 200 MHz SYSCLK | `SYSCLK_P=AD12`、`SYSCLK_N=AD11` | Bank33/1.5 V 差分专用输入 | [S1, PDF p.3]；[S2, PDF p.5] |
| Ethernet | MDC A11；MDIO A12；RST E11；RXCK H14；RXD0-3 F15/B15/C15/A15；RXDV B14；TXCK B17；TXD0-3 C17/G15/H15/G14；TXEN E16 | 已连接 PHY，不作传感器 GPIO | [S1, PDF p.9-p.13]；[S2, PDF p.11] |
| TF 卡 | CD H24；CLK H26；CMD G27；DAT0 H27；DAT1 H25；DAT2 F28；DAT3 F27 | 已连接 J5 卡座和 4.7 kΩ 上拉 | [S1, PDF p.17-p.18]；[S2, PDF p.13] |
| HDMI | CEC B29；HPD D29；SCL B28；SDA A28；CLK P/N D27/C27；D0 P/N H30/G30；D1 P/N G29/F30；D2 P/N B30/A30 | 已连接 J2、ESD/电平电路；不作通用 I²C | [S1, PDF p.16-p.20]；[S2, PDF p.8] |
| VGA | R3-7 M24/M25/M27/N20/N29；G2-7 N30/N27/N26/N25/N22/N24；B3-7 P23/N21/P22/N19/P21；HS M23；VS M22 | 已连接电阻 DAC 和 J6 | [S1, PDF p.20-p.22]；[S2, PDF p.14] |
| USB OTG/Host | DATA0-7 AC20/AD22/Y21/AA23/AB24/AC25/Y23/Y24；STP AA22；NXT AC21；DIR AC22；CLK AE23；RESET AA30 | 已连接 USB3320C/USB2514 和 J11/J12 | [S1, PDF p.30-p.31]；[S2, PDF p.15] |
| DDR3 | DDR_D0..63、DQS0..7、DM0..7、A0..14、BA0..2、CLK、CKE、CS、RAS、CAS、WE、ODT、RESET | Bank32/33/34，1.5 V，完全板载占用 | [S1, PDF p.1-p.9]；[S2, PDF p.5、p.9-p.10] |

### 4.8 FunctionPin-V2.1 完整原始映射

下表逐行转录 Function Pin Report 可解析的全部 405 条功能引脚记录。它包含板载专用连接和可扩展引脚，不能把每一行都当作空闲 GPIO。`报告分组`是原报告的 FuncDes 标识，`FPGA 原生引脚名`保留器件原始复用能力名称；`电压域`按原理图 Bank 供电归类，`用途分类`不改变原报告内容。

| PDF页 | 报告分组 | I/O Bank | FPGA 原生引脚名 | Slot | Ref | 封装脚 | 板级网络 | 电压域 | 用途分类 | 来源 |
|---:|---|---|---|---|---|---|---|---|---|---|
| 1 | F32 | 配置 Bank 0 | CCLK_0 | G11 | U1 | B10 | FLASH_CLK | VCC3V3 | 配置 Flash | [S1, PDF p.1] |
| 1 | F32 | 配置 Bank 0 | DONE_0 | G11 | U1 | M10 | DONE | VCC3V3 | FPGA 配置 | [S1, PDF p.1] |
| 1 | F32 | 配置 Bank 0 | INIT_B_0 | G11 | U1 | A10 | N29636 | VCC3V3 | FPGA 配置 | [S1, PDF p.1] |
| 1 | F32 | 配置 Bank 0 | PROGRAM_B_0 | G11 | U1 | K10 | N29634 | VCC3V3 | FPGA 配置 | [S1, PDF p.1] |
| 1 | F32 | 配置 Bank 0 | TCK_0 | G11 | U1 | E10 | JTAG_TCK | VCC3V3 | JTAG | [S1, PDF p.1] |
| 1 | F32 | 配置 Bank 0 | TDI_0 | G11 | U1 | H10 | JTAG_TDI | VCC3V3 | JTAG | [S1, PDF p.1] |
| 1 | F32 | 配置 Bank 0 | TDO_0 | G11 | U1 | G10 | JTAG_TDO | VCC3V3 | JTAG | [S1, PDF p.1] |
| 1 | F32 | 配置 Bank 0 | TMS_0 | G11 | U1 | F10 | JTAG_TMS | VCC3V3 | JTAG | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_0_VRN_34 | G10 | U1 | AC6 | B34_01 | VCC1V5 | 板载连接/待确认 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_25_VRP_34 | G10 | U1 | AB7 | B34_55 | VCC1V5 | 板载连接/待确认 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L10N_T1_34 | G10 | U1 | AE3 | DDR_D27 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L11N_T1_SRCC_34 | G10 | U1 | AF5 | DDR_D24 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L11P_T1_SRCC_34 | G10 | U1 | AE5 | DDR_D26 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L12N_T1_MRCC_34 | G10 | U1 | AG5 | DDR_D28 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L12P_T1_MRCC_34 | G10 | U1 | AF6 | DDR_D30 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L13N_T2_MRCC_34 | G10 | U1 | AJ4 | DDR_D3 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L14N_T2_SRCC_34 | G10 | U1 | AH5 | DDR_D7 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L14P_T2_SRCC_34 | G10 | U1 | AH6 | DDR_D5 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L15N_T2_DQS_34 | G10 | U1 | AH1 | DDR_DQS0_N | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L15P_T2_DQS_34 | G10 | U1 | AG2 | DDR_DQS0_P | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L16N_T2_34 | G10 | U1 | AJ2 | DDR_D6 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 1 | F33 | Bank 34 | IO_L16P_T2_34 | G10 | U1 | AH2 | DDR_D4 | VCC1V5 | DDR3 | [S1, PDF p.1] |
| 2 | F33 | Bank 34 | IO_L17N_T2_34 | G10 | U1 | AK1 | DDR_D0 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L17P_T2_34 | G10 | U1 | AJ1 | DDR_D2 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L18N_T2_34 | G10 | U1 | AK3 | DDR_DM0 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L18P_T2_34 | G10 | U1 | AJ3 | DDR_D1 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L19N_T3_VREF_34 | G10 | U1 | AG8 | DDRVREF | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L19P_T3_34 | G10 | U1 | AF8 | DDR_D21 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L1N_T0_34 | G10 | U1 | AD3 | DDR_D12 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L1P_T0_34 | G10 | U1 | AD4 | DDR_D8 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L20N_T3_34 | G10 | U1 | AG7 | DDR_DM2 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L20P_T3_34 | G10 | U1 | AF7 | DDR_D19 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L21N_T3_DQS_34 | G10 | U1 | AJ7 | DDR_DQS2_N | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L21P_T3_DQS_34 | G10 | U1 | AH7 | DDR_DQS2_P | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L22N_T3_34 | G10 | U1 | AK6 | DDR_D18 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L22P_T3_34 | G10 | U1 | AJ6 | DDR_D16 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L23N_T3_34 | G10 | U1 | AK8 | DDR_D23 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L23P_T3_34 | G10 | U1 | AJ8 | DDR_D17 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L24N_T3_34 | G10 | U1 | AK4 | DDR_D20 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L24P_T3_34 | G10 | U1 | AK5 | DDR_D22 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L2N_T0_34 | G10 | U1 | AC1 | DDR_D9 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L2P_T0_34 | G10 | U1 | AC2 | DDR_D15 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L3N_T0_DQS_34 | G10 | U1 | AD1 | DDR_DQS1_N | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L3P_T0_DQS_34 | G10 | U1 | AD2 | DDR_DQS1_P | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L4N_T0_34 | G10 | U1 | AC4 | DDR_DM1 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L4P_T0_34 | G10 | U1 | AC5 | DDR_D11 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L5N_T0_34 | G10 | U1 | AE6 | DDR_D14 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L5P_T0_34 | G10 | U1 | AD6 | DDR_D10 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L6N_T0_VREF_34 | G10 | U1 | AD7 | DDRVREF | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L6P_T0_34 | G10 | U1 | AC7 | DDR_D13 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L7N_T1_34 | G10 | U1 | AF2 | DDR_D25 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L7P_T1_34 | G10 | U1 | AF3 | DDR_D31 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L8N_T1_34 | G10 | U1 | AF1 | DDR_D29 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 2 | F33 | Bank 34 | IO_L8P_T1_34 | G10 | U1 | AE1 | DDR_DM3 | VCC1V5 | DDR3 | [S1, PDF p.2] |
| 3 | F33 | Bank 34 | IO_L9N_T1_DQS_34 | G10 | U1 | AG3 | DDR_DQS3_N | VCC1V5 | DDR3 | [S1, PDF p.3] |
| 3 | F33 | Bank 34 | IO_L9P_T1_DQS_34 | G10 | U1 | AG4 | DDR_DQS3_P | VCC1V5 | DDR3 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_0_VRN_33 | G9 | U1 | Y13 | B33_01 | VCC1V5 | 板载连接/待确认 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_25_VRP_33 | G9 | U1 | AD13 | B33_55 | VCC1V5 | 板载连接/待确认 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_L10N_T1_33 | G9 | U1 | AE9 | DDR_A14 | VCC1V5 | DDR3 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_L10P_T1_33 | G9 | U1 | AD9 | DDR_A13 | VCC1V5 | DDR3 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_L11N_T1_SRCC_33 | G9 | U1 | AF11 | DDR_WE | VCC1V5 | DDR3 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_L11P_T1_SRCC_33 | G9 | U1 | AE11 | DDR_CKE | VCC1V5 | DDR3 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_L12N_T1_MRCC_33 | G9 | U1 | AD11 | SYSCLK_N | VCC1V5 | 系统时钟 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_L12P_T1_MRCC_33 | G9 | U1 | AD12 | SYSCLK_P | VCC1V5 | 系统时钟 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_L13N_T2_MRCC_33 | G9 | U1 | AH10 | DDR_BA1 | VCC1V5 | DDR3 | [S1, PDF p.3] |
| 3 | F34 | Bank 33 | IO_L13P_T2_MRCC_33 | G9 | U1 | AG10 | DDR_BA2 | VCC1V5 | DDR3 | [S1, PDF p.3] |
| 4 | F34 | Bank 33 | IO_L14N_T2_SRCC_33 | G9 | U1 | AF10 | DDR_CS | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L14P_T2_SRCC_33 | G9 | U1 | AE10 | DDR_A9 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L15N_T2_DQS_33 | G9 | U1 | AK9 | DDR_A8 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L15P_T2_DQS_33 | G9 | U1 | AJ9 | DDR_CAS | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L16N_T2_33 | G9 | U1 | AH9 | DDR_RAS | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L16P_T2_33 | G9 | U1 | AG9 | DDR_A0 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L17N_T2_33 | G9 | U1 | AK10 | DDR_A4 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L17P_T2_33 | G9 | U1 | AK11 | DDR_A10 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L18N_T2_33 | G9 | U1 | AJ11 | DDR_A6 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L19N_T3_VREF_33 | G9 | U1 | AF13 | DDRVREF | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L21N_T3_DQS_33 | G9 | U1 | AJ14 | DDR_CLK_N | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L21P_T3_DQS_33 | G9 | U1 | AH14 | DDR_CLK_P | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L2N_T0_33 | G9 | U1 | AB8 | DDR_A3 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L2P_T0_33 | G9 | U1 | AA8 | DDR_A12 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 4 | F34 | Bank 33 | IO_L3N_T0_DQS_33 | G9 | U1 | AC9 | DDR_A5 | VCC1V5 | DDR3 | [S1, PDF p.4] |
| 5 | F34 | Bank 33 | IO_L3P_T0_DQS_33 | G9 | U1 | AB9 | DDR_A1 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F34 | Bank 33 | IO_L5N_T0_33 | G9 | U1 | AA10 | DDR_A11 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F34 | Bank 33 | IO_L6N_T0_VREF_33 | G9 | U1 | AB13 | DDRVREF | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F34 | Bank 33 | IO_L7N_T1_33 | G9 | U1 | AC10 | DDR_A7 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F34 | Bank 33 | IO_L7P_T1_33 | G9 | U1 | AB10 | DDR_RESET | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F34 | Bank 33 | IO_L8N_T1_33 | G9 | U1 | AE8 | DDR_BA0 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F34 | Bank 33 | IO_L8P_T1_33 | G9 | U1 | AD8 | DDR_ODT | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F34 | Bank 33 | IO_L9N_T1_DQS_33 | G9 | U1 | AC11 | DDR_A2 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_0_VRN_32 | G8 | U1 | Y14 | B32_01 | VCC1V5 | 板载连接/待确认 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_25_VRP_32 | G8 | U1 | AB14 | B32_55 | VCC1V5 | 板载连接/待确认 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_L10N_T1_32 | G8 | U1 | AE19 | DDR_D58 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_L11N_T1_SRCC_32 | G8 | U1 | AG18 | DDR_D63 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_L11P_T1_SRCC_32 | G8 | U1 | AF18 | DDR_D57 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_L12N_T1_MRCC_32 | G8 | U1 | AG17 | DDR_D61 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_L12P_T1_MRCC_32 | G8 | U1 | AF17 | DDR_D59 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_L13N_T2_MRCC_32 | G8 | U1 | AE18 | DDR_D40 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 5 | F35 | Bank 32 | IO_L13P_T2_MRCC_32 | G8 | U1 | AD18 | DDR_D42 | VCC1V5 | DDR3 | [S1, PDF p.5] |
| 6 | F35 | Bank 32 | IO_L14N_T2_SRCC_32 | G8 | U1 | AD16 | DDR_D41 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L14P_T2_SRCC_32 | G8 | U1 | AD17 | DDR_D47 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L15N_T2_DQS_32 | G8 | U1 | Y18 | DDR_DQS5_N | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L15P_T2_DQS_32 | G8 | U1 | Y19 | DDR_DQS5_P | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L16N_T2_32 | G8 | U1 | AB18 | DDR_DM5 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L17N_T2_32 | G8 | U1 | AC19 | DDR_D44 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L17P_T2_32 | G8 | U1 | AB19 | DDR_D46 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L18N_T2_32 | G8 | U1 | AC17 | DDR_D45 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L18P_T2_32 | G8 | U1 | AB17 | DDR_D43 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L19N_T3_VREF_32 | G8 | U1 | AE14 | DDRVREF | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L19P_T3_32 | G8 | U1 | AE15 | DDR_D32 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L1N_T0_32 | G8 | U1 | AK15 | DDR_D50 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 6 | F35 | Bank 32 | IO_L1P_T0_32 | G8 | U1 | AK16 | DDR_D48 | VCC1V5 | DDR3 | [S1, PDF p.6] |
| 7 | F35 | Bank 32 | IO_L20N_T3_32 | G8 | U1 | AB15 | DDR_D38 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L20P_T3_32 | G8 | U1 | AA15 | DDR_D39 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L21N_T3_DQS_32 | G8 | U1 | AC15 | DDR_DQS4_N | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L21P_T3_DQS_32 | G8 | U1 | AC16 | DDR_DQS4_P | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L22N_T3_32 | G8 | U1 | AD14 | DDR_D36 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L22P_T3_32 | G8 | U1 | AC14 | DDR_D34 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L23N_T3_32 | G8 | U1 | AA16 | DDR_D37 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L23P_T3_32 | G8 | U1 | AA17 | DDR_D33 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L24N_T3_32 | G8 | U1 | Y15 | DDR_DM4 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L24P_T3_32 | G8 | U1 | Y16 | DDR_D35 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L2N_T0_32 | G8 | U1 | AH15 | DDR_D54 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 7 | F35 | Bank 32 | IO_L2P_T0_32 | G8 | U1 | AG15 | DDR_D49 | VCC1V5 | DDR3 | [S1, PDF p.7] |
| 8 | F35 | Bank 32 | IO_L3N_T0_DQS_32 | G8 | U1 | AJ16 | DDR_DQS6_N | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L3P_T0_DQS_32 | G8 | U1 | AH16 | DDR_DQS6_P | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L4N_T0_32 | G8 | U1 | AG14 | DDR_D52 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L4P_T0_32 | G8 | U1 | AF15 | DDR_DM6 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L5N_T0_32 | G8 | U1 | AJ17 | DDR_D55 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L5P_T0_32 | G8 | U1 | AH17 | DDR_D53 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L6N_T0_VREF_32 | G8 | U1 | AF16 | DDRVREF | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L6P_T0_32 | G8 | U1 | AE16 | DDR_D51 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L7N_T1_32 | G8 | U1 | AK19 | DDR_DM7 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L7P_T1_32 | G8 | U1 | AJ19 | DDR_D60 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L8N_T1_32 | G8 | U1 | AH19 | DDR_D56 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 8 | F35 | Bank 32 | IO_L8P_T1_32 | G8 | U1 | AG19 | DDR_D62 | VCC1V5 | DDR3 | [S1, PDF p.8] |
| 9 | F35 | Bank 32 | IO_L9N_T1_DQS_32 | G8 | U1 | AK18 | DDR_DQS7_N | VCC1V5 | DDR3 | [S1, PDF p.9] |
| 9 | F35 | Bank 32 | IO_L9P_T1_DQS_32 | G8 | U1 | AJ18 | DDR_DQS7_P | VCC1V5 | DDR3 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L10N_T1_18 | G7 | U1 | H12 | LED2_F | VCC3V3 | LED/数码管 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L10P_T1_18 | G7 | U1 | H11 | LED2_B | VCC3V3 | LED/数码管 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L11N_T1_SRCC_18 | G7 | U1 | G14 | ETH1_TXD3 | VCC3V3 | 以太网 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L11P_T1_SRCC_18 | G7 | U1 | H14 | ETH1_RXCK | VCC3V3 | 以太网 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L12N_T1_MRCC_18 | G7 | U1 | F13 | LED2_G | VCC3V3 | LED/数码管 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L12P_T1_MRCC_18 | G7 | U1 | G13 | LED2_DP | VCC3V3 | LED/数码管 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L13P_T2_MRCC_18 | G7 | U1 | D12 | N4009740 | VCC3V3 | 板载连接/待确认 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L14P_T2_SRCC_18 | G7 | U1 | F12 | LED1 | VCC3V3 | LED/数码管 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L16N_T2_18 | G7 | U1 | E11 | ETH_RST | VCC3V3 | 以太网 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L17N_T2_18 | G7 | U1 | A12 | ETH_MDIO | VCC3V3 | 以太网 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L17P_T2_18 | G7 | U1 | A11 | ETH_MDC | VCC3V3 | 以太网 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L18N_T2_18 | G7 | U1 | C11 | KEY7 | VCC3V3 | 按键 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L18P_T2_18 | G7 | U1 | D11 | KEY8 | VCC3V3 | 按键 | [S1, PDF p.9] |
| 9 | F36 | Bank 18 | IO_L19N_T3_VREF_18 | G7 | U1 | E16 | ETH1_TXEN | VCC3V3 | 以太网 | [S1, PDF p.9] |
| 10 | F36 | Bank 18 | IO_L19P_T3_18 | G7 | U1 | F15 | ETH1_RXD0 | VCC3V3 | 以太网 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L1N_T0_18 | G7 | U1 | K16 | LED1_E | VCC3V3 | LED/数码管 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L1P_T0_18 | G7 | U1 | L16 | LED1_CS2 | VCC3V3 | LED/数码管 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L20N_T3_18 | G7 | U1 | E15 | KEY1 | VCC3V3 | 按键 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L20P_T3_18 | G7 | U1 | E14 | KEY6 | VCC3V3 | 按键 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L21N_T3_DQS_18 | G7 | U1 | C14 | KEY2 | VCC3V3 | 按键 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L21P_T3_DQS_18 | G7 | U1 | D14 | KEY3 | VCC3V3 | 按键 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L22N_T3_18 | G7 | U1 | A13 | KEY4 | VCC3V3 | 按键 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L22P_T3_18 | G7 | U1 | B13 | KEY5 | VCC3V3 | 按键 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L23N_T3_18 | G7 | U1 | B15 | ETH1_RXD1 | VCC3V3 | 以太网 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L23P_T3_18 | G7 | U1 | C15 | ETH1_RXD2 | VCC3V3 | 以太网 | [S1, PDF p.10] |
| 10 | F36 | Bank 18 | IO_L24N_T3_18 | G7 | U1 | A15 | ETH1_RXD3 | VCC3V3 | 以太网 | [S1, PDF p.10] |
| 11 | F36 | Bank 18 | IO_L24P_T3_18 | G7 | U1 | B14 | ETH1_RXDV | VCC3V3 | 以太网 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L2N_T0_18 | G7 | U1 | K15 | LED1_F | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L2P_T0_18 | G7 | U1 | L15 | LED1_B | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L3N_T0_DQS_18 | G7 | U1 | L13 | LED1_CS1 | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L3P_T0_DQS_18 | G7 | U1 | L12 | LED1_G | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L4N_T0_18 | G7 | U1 | J13 | LED2_A | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L4P_T0_18 | G7 | U1 | K13 | LED1_DP | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L5N_T0_18 | G7 | U1 | J14 | LED2_CS1 | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L5P_T0_18 | G7 | U1 | K14 | LED1_A | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L6N_T0_VREF_18 | G7 | U1 | K11 | LED2_CS2 | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L6P_T0_18 | G7 | U1 | L11 | LED2_D | VCC3V3 | LED/数码管 | [S1, PDF p.11] |
| 11 | F36 | Bank 18 | IO_L7N_T1_18 | G7 | U1 | G15 | ETH1_TXD1 | VCC3V3 | 以太网 | [S1, PDF p.11] |
| 12 | F36 | Bank 18 | IO_L7P_T1_18 | G7 | U1 | H15 | ETH1_TXD2 | VCC3V3 | 以太网 | [S1, PDF p.12] |
| 12 | F36 | Bank 18 | IO_L8N_T1_18 | G7 | U1 | J12 | LED2_C | VCC3V3 | LED/数码管 | [S1, PDF p.12] |
| 12 | F36 | Bank 18 | IO_L8P_T1_18 | G7 | U1 | J11 | LED2_E | VCC3V3 | LED/数码管 | [S1, PDF p.12] |
| 12 | F36 | Bank 18 | IO_L9N_T1_DQS_18 | G7 | U1 | H16 | LED1_C | VCC3V3 | LED/数码管 | [S1, PDF p.12] |
| 12 | F36 | Bank 18 | IO_L9P_T1_DQS_18 | G7 | U1 | J16 | LED1_D | VCC3V3 | LED/数码管 | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_0_17 | G6 | U1 | G19 | DEBUG_9 | VADJ1 | 外部 DEBUG | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_25_17 | G6 | U1 | E18 | DEBUG_7 | VADJ1 | 外部 DEBUG | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_L10N_T1_17 | G6 | U1 | C22 | DEBUG_35 | VADJ1 | 外部 DEBUG | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_L10P_T1_17 | G6 | U1 | D22 | DEBUG_37 | VADJ1 | 外部 DEBUG | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_L11N_T1_SRCC_17 | G6 | U1 | E21 | DEBUG_36 | VADJ1 | 外部 DEBUG | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_L11P_T1_SRCC_17 | G6 | U1 | F21 | DEBUG_38 | VADJ1 | 外部 DEBUG | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_L13N_T2_MRCC_17 | G6 | U1 | D18 | CP2104_TXD | VADJ1 | USB-UART | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_L13P_T2_MRCC_17 | G6 | U1 | D17 | CP2104_RXD | VADJ1 | USB-UART | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_L16N_T2_17 | G6 | U1 | F18 | DEBUG_5 | VADJ1 | 外部 DEBUG | [S1, PDF p.12] |
| 12 | F37 | Bank 17 | IO_L16P_T2_17 | G6 | U1 | G18 | DEBUG_2 | VADJ1 | 外部 DEBUG | [S1, PDF p.12] |
| 13 | F37 | Bank 17 | IO_L17N_T2_17 | G6 | U1 | B17 | ETH1_TXCK | VADJ1 | 以太网 | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L17P_T2_17 | G6 | U1 | C17 | ETH1_TXD0 | VADJ1 | 以太网 | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L18N_T2_17 | G6 | U1 | F17 | DEBUG_3 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L18P_T2_17 | G6 | U1 | G17 | DEBUG_1 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L19N_T3_VREF_17 | G6 | U1 | B20 | DEBUG_19 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L19P_T3_17 | G6 | U1 | C20 | DEBUG_17 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L1N_T0_17 | G6 | U1 | J18 | DEBUG_26 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L1P_T0_17 | G6 | U1 | K18 | DEBUG_25 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L20N_T3_17 | G6 | U1 | A17 | DEBUG_8 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L20P_T3_17 | G6 | U1 | A16 | DEBUG_6 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L21N_T3_DQS_17 | G6 | U1 | A21 | DEBUG_32 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L21P_T3_DQS_17 | G6 | U1 | A20 | DEBUG_15 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 13 | F37 | Bank 17 | IO_L22N_T3_17 | G6 | U1 | A18 | DEBUG_14 | VADJ1 | 外部 DEBUG | [S1, PDF p.13] |
| 14 | F37 | Bank 17 | IO_L22P_T3_17 | G6 | U1 | B18 | DEBUG_13 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L23N_T3_17 | G6 | U1 | A22 | DEBUG_31 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L23P_T3_17 | G6 | U1 | B22 | DEBUG_20 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L24N_T3_17 | G6 | U1 | B19 | DEBUG_12 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L24P_T3_17 | G6 | U1 | C19 | DEBUG_11 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L2N_T0_17 | G6 | U1 | G20 | DEBUG_10 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L2P_T0_17 | G6 | U1 | H20 | DEBUG_4 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L3N_T0_DQS_17 | G6 | U1 | H17 | DEBUG_29 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L3P_T0_DQS_17 | G6 | U1 | J17 | DEBUG_27 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L4N_T0_17 | G6 | U1 | H19 | DEBUG_30 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L4P_T0_17 | G6 | U1 | J19 | DEBUG_28 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 14 | F37 | Bank 17 | IO_L5N_T0_17 | G6 | U1 | L18 | DEBUG_23 | VADJ1 | 外部 DEBUG | [S1, PDF p.14] |
| 15 | F37 | Bank 17 | IO_L5P_T0_17 | G6 | U1 | L17 | DEBUG_24 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F37 | Bank 17 | IO_L6N_T0_VREF_17 | G6 | U1 | K20 | DEBUG_21 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F37 | Bank 17 | IO_L6P_T0_17 | G6 | U1 | K19 | DEBUG_22 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F37 | Bank 17 | IO_L7N_T1_17 | G6 | U1 | H22 | DEBUG_18 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F37 | Bank 17 | IO_L7P_T1_17 | G6 | U1 | H21 | DEBUG_16 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F37 | Bank 17 | IO_L8N_T1_17 | G6 | U1 | C21 | DEBUG_33 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F37 | Bank 17 | IO_L8P_T1_17 | G6 | U1 | D21 | DEBUG_34 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F37 | Bank 17 | IO_L9N_T1_DQS_17 | G6 | U1 | F22 | DEBUG_39 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F37 | Bank 17 | IO_L9P_T1_DQS_17 | G6 | U1 | G22 | DEBUG_40 | VADJ1 | 外部 DEBUG | [S1, PDF p.15] |
| 15 | F38 | Bank 16 | IO_0_16 | G5 | U1 | F23 | LED3 | VADJ1 | LED/数码管 | [S1, PDF p.15] |
| 15 | F38 | Bank 16 | IO_25_16 | G5 | U1 | G25 | LED15 | VADJ1 | LED/数码管 | [S1, PDF p.15] |
| 15 | F38 | Bank 16 | IO_L10N_T1_16 | G5 | U1 | A26 | LED6 | VADJ1 | LED/数码管 | [S1, PDF p.15] |
| 16 | F38 | Bank 16 | IO_L10P_T1_16 | G5 | U1 | A25 | LED8 | VADJ1 | LED/数码管 | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L11N_T1_SRCC_16 | G5 | U1 | C26 | LED16 | VADJ1 | LED/数码管 | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L11P_T1_SRCC_16 | G5 | U1 | D26 | LED11 | VADJ1 | LED/数码管 | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L12N_T1_MRCC_16 | G5 | U1 | B25 | LED7 | VADJ1 | LED/数码管 | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L12P_T1_MRCC_16 | G5 | U1 | C25 | LED26 | VADJ1 | LED/数码管 | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L13N_T2_MRCC_16 | G5 | U1 | C27 | HDMI1_FCLK_N | VADJ1 | HDMI | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L13P_T2_MRCC_16 | G5 | U1 | D27 | HDMI1_FCLK_P | VADJ1 | HDMI | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L14N_T2_SRCC_16 | G5 | U1 | D28 | LED19 | VADJ1 | LED/数码管 | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L14P_T2_SRCC_16 | G5 | U1 | E28 | LED9 | VADJ1 | LED/数码管 | [S1, PDF p.16] |
| 16 | F38 | Bank 16 | IO_L15N_T2_DQS_16 | G5 | U1 | B29 | HDMI1_F_CEC | VADJ1 | HDMI | [S1, PDF p.16] |
| 17 | F38 | Bank 16 | IO_L15P_T2_DQS_16 | G5 | U1 | C29 | LED25 | VADJ1 | LED/数码管 | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L16N_T2_16 | G5 | U1 | C30 | LED20 | VADJ1 | LED/数码管 | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L16P_T2_16 | G5 | U1 | D29 | HDMI1_F_HPD | VADJ1 | HDMI | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L17N_T2_16 | G5 | U1 | A30 | HDMI1_FD2_N | VADJ1 | HDMI | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L17P_T2_16 | G5 | U1 | B30 | HDMI1_FD2_P | VADJ1 | HDMI | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L18N_T2_16 | G5 | U1 | E30 | LED2 | VADJ1 | LED/数码管 | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L18P_T2_16 | G5 | U1 | E29 | LED13 | VADJ1 | LED/数码管 | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L19N_T3_VREF_16 | G5 | U1 | H25 | TF_DAT1 | VADJ1 | TF 卡 | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L19P_T3_16 | G5 | U1 | H24 | TF_CD | VADJ1 | TF 卡 | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L1N_T0_16 | G5 | U1 | A23 | LED21 | VADJ1 | LED/数码管 | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L1P_T0_16 | G5 | U1 | B23 | LED24 | VADJ1 | LED/数码管 | [S1, PDF p.17] |
| 17 | F38 | Bank 16 | IO_L20N_T3_16 | G5 | U1 | F28 | TF_DAT2 | VADJ1 | TF 卡 | [S1, PDF p.17] |
| 18 | F38 | Bank 16 | IO_L20P_T3_16 | G5 | U1 | G28 | LED10 | VADJ1 | LED/数码管 | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L21N_T3_DQS_16 | G5 | U1 | F27 | TF_DAT3 | VADJ1 | TF 卡 | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L21P_T3_DQS_16 | G5 | U1 | G27 | TF_CMD | VADJ1 | TF 卡 | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L22N_T3_16 | G5 | U1 | F30 | HDMI1_FD1_N | VADJ1 | HDMI | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L22P_T3_16 | G5 | U1 | G29 | HDMI1_FD1_P | VADJ1 | HDMI | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L23N_T3_16 | G5 | U1 | H27 | TF_DAT0 | VADJ1 | TF 卡 | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L23P_T3_16 | G5 | U1 | H26 | TF_CLK | VADJ1 | TF 卡 | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L24N_T3_16 | G5 | U1 | G30 | HDMI1_FD0_N | VADJ1 | HDMI | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L24P_T3_16 | G5 | U1 | H30 | HDMI1_FD0_P | VADJ1 | HDMI | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L2N_T0_16 | G5 | U1 | D23 | LED4 | VADJ1 | LED/数码管 | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L2P_T0_16 | G5 | U1 | E23 | LED27 | VADJ1 | LED/数码管 | [S1, PDF p.18] |
| 18 | F38 | Bank 16 | IO_L3N_T0_DQS_16 | G5 | U1 | E25 | LED29 | VADJ1 | LED/数码管 | [S1, PDF p.18] |
| 19 | F38 | Bank 16 | IO_L3P_T0_DQS_16 | G5 | U1 | F25 | LED17 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L4N_T0_16 | G5 | U1 | D24 | LED12 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L4P_T0_16 | G5 | U1 | E24 | LED30 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L5N_T0_16 | G5 | U1 | E26 | LED18 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L5P_T0_16 | G5 | U1 | F26 | LED14 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L6N_T0_VREF_16 | G5 | U1 | G24 | LED32 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L6P_T0_16 | G5 | U1 | G23 | LED28 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L7N_T1_16 | G5 | U1 | A27 | LED5 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L7P_T1_16 | G5 | U1 | B27 | LED23 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L8N_T1_16 | G5 | U1 | B24 | LED22 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L8P_T1_16 | G5 | U1 | C24 | LED31 | VADJ1 | LED/数码管 | [S1, PDF p.19] |
| 19 | F38 | Bank 16 | IO_L9N_T1_DQS_16 | G5 | U1 | A28 | HDMI1_F_SDA | VADJ1 | HDMI | [S1, PDF p.19] |
| 20 | F38 | Bank 16 | IO_L9P_T1_DQS_16 | G5 | U1 | B28 | HDMI1_F_SCL | VADJ1 | HDMI | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L10N_T1_AD4N_15 | G4 | U1 | J26 | SW_54 | VADJ1 | 拨码开关 | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L10P_T1_AD4P_15 | G4 | U1 | K26 | SW_60 | VADJ1 | 拨码开关 | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L11N_T1_SRCC_AD12N_15 | G4 | U1 | L27 | SW_64 | VADJ1 | 拨码开关 | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L11P_T1_SRCC_AD12P_15 | G4 | U1 | L26 | SW_63 | VADJ1 | 拨码开关 | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L12P_T1_MRCC_AD5P_15 | G4 | U1 | L25 | SW_55 | VADJ1 | 拨码开关 | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L16N_T2_A27_15 | G4 | U1 | M27 | VGA_R5 | VADJ1 | VGA | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L16P_T2_A28_15 | G4 | U1 | N27 | VGA_G3 | VADJ1 | VGA | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L17N_T2_A25_15 | G4 | U1 | N30 | VGA_G2 | VADJ1 | VGA | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L17P_T2_A26_15 | G4 | U1 | N29 | VGA_R7 | VADJ1 | VGA | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L18N_T2_A23_15 | G4 | U1 | N26 | VGA_G4 | VADJ1 | VGA | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L18P_T2_A24_15 | G4 | U1 | N25 | VGA_G5 | VADJ1 | VGA | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L19N_T3_A21_VREF_15 | G4 | U1 | N20 | VGA_R6 | VADJ1 | VGA | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L19P_T3_A22_15 | G4 | U1 | N19 | VGA_B6 | VADJ1 | VGA | [S1, PDF p.20] |
| 20 | F39 | Bank 15 | IO_L1N_T0_AD0N_15 | G4 | U1 | J24 | SW_52 | VADJ1 | 拨码开关 | [S1, PDF p.20] |
| 21 | F39 | Bank 15 | IO_L1P_T0_AD0P_15 | G4 | U1 | J23 | SW_50 | VADJ1 | 拨码开关 | [S1, PDF p.21] |
| 21 | F39 | Bank 15 | IO_L20N_T3_A19_15 | G4 | U1 | N22 | VGA_G6 | VADJ1 | VGA | [S1, PDF p.21] |
| 21 | F39 | Bank 15 | IO_L20P_T3_A20_15 | G4 | U1 | N21 | VGA_B4 | VADJ1 | VGA | [S1, PDF p.21] |
| 21 | F39 | Bank 15 | IO_L21N_T3_DQS_A18_15 | G4 | U1 | N24 | VGA_G7 | VADJ1 | VGA | [S1, PDF p.21] |
| 21 | F39 | Bank 15 | IO_L21P_T3_DQS_15 | G4 | U1 | P23 | VGA_B3 | VADJ1 | VGA | [S1, PDF p.21] |
| 21 | F39 | Bank 15 | IO_L22N_T3_A16_15 | G4 | U1 | P22 | VGA_B5 | VADJ1 | VGA | [S1, PDF p.21] |
| 21 | F39 | Bank 15 | IO_L22P_T3_A17_15 | G4 | U1 | P21 | VGA_B7 | VADJ1 | VGA | [S1, PDF p.21] |
| 21 | F39 | Bank 15 | IO_L23N_T3_FWE_B_15 | G4 | U1 | M25 | VGA_R4 | VADJ1 | VGA | [S1, PDF p.21] |
| 21 | F39 | Bank 15 | IO_L23P_T3_FOE_B_15 | G4 | U1 | M24 | VGA_R3 | VADJ1 | VGA | [S1, PDF p.21] |
| 22 | F39 | Bank 15 | IO_L24N_T3_RS0_15 | G4 | U1 | M23 | VGA_HS | VADJ1 | VGA | [S1, PDF p.22] |
| 22 | F39 | Bank 15 | IO_L24P_T3_RS1_15 | G4 | U1 | M22 | VGA_VS | VADJ1 | VGA | [S1, PDF p.22] |
| 22 | F39 | Bank 15 | IO_L2N_T0_AD8N_15 | G4 | U1 | L23 | SW_51 | VADJ1 | 拨码开关 | [S1, PDF p.22] |
| 22 | F39 | Bank 15 | IO_L2P_T0_AD8P_15 | G4 | U1 | L22 | SW_45 | VADJ1 | 拨码开关 | [S1, PDF p.22] |
| 22 | F39 | Bank 15 | IO_L3N_T0_DQS_AD1N_15 | G4 | U1 | K24 | SW_53 | VADJ1 | 拨码开关 | [S1, PDF p.22] |
| 22 | F39 | Bank 15 | IO_L3P_T0_DQS_AD1P_15 | G4 | U1 | K23 | SW_49 | VADJ1 | 拨码开关 | [S1, PDF p.22] |
| 22 | F39 | Bank 15 | IO_L4N_T0_AD9N_15 | G4 | U1 | K21 | SW_46 | VADJ1 | 拨码开关 | [S1, PDF p.22] |
| 22 | F39 | Bank 15 | IO_L4P_T0_AD9P_15 | G4 | U1 | L21 | SW_43 | VADJ1 | 拨码开关 | [S1, PDF p.22] |
| 22 | F39 | Bank 15 | IO_L5N_T0_AD2N_15 | G4 | U1 | J22 | SW_48 | VADJ1 | 拨码开关 | [S1, PDF p.22] |
| 23 | F39 | Bank 15 | IO_L5P_T0_AD2P_15 | G4 | U1 | J21 | SW_47 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F39 | Bank 15 | IO_L6N_T0_VREF_15 | G4 | U1 | L20 | SW_44 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F39 | Bank 15 | IO_L6P_T0_15 | G4 | U1 | M20 | SW_42 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F39 | Bank 15 | IO_L7N_T1_AD10N_15 | G4 | U1 | H29 | SW_57 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F39 | Bank 15 | IO_L7P_T1_AD10P_15 | G4 | U1 | J29 | SW_59 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F39 | Bank 15 | IO_L8N_T1_AD3N_15 | G4 | U1 | J28 | SW_58 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F39 | Bank 15 | IO_L8P_T1_AD3P_15 | G4 | U1 | J27 | SW_56 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F39 | Bank 15 | IO_L9N_T1_DQS_AD11N_15 | G4 | U1 | K30 | SW_61 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F39 | Bank 15 | IO_L9P_T1_DQS_AD11P_15 | G4 | U1 | L30 | SW_62 | VADJ1 | 拨码开关 | [S1, PDF p.23] |
| 23 | F40 | Bank 14 | IO_25_14 | G3 | U1 | W19 | BEEP | VCC3V3 | 板载连接/待确认 | [S1, PDF p.23] |
| 23 | F40 | Bank 14 | IO_L10N_T1_D15_14 | G3 | U1 | R26 | SW_10 | VCC3V3 | 拨码开关 | [S1, PDF p.23] |
| 24 | F40 | Bank 14 | IO_L10P_T1_D14_14 | G3 | U1 | P26 | SW_6 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 24 | F40 | Bank 14 | IO_L11N_T1_SRCC_14 | G3 | U1 | T28 | SW_17 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 24 | F40 | Bank 14 | IO_L11P_T1_SRCC_14 | G3 | U1 | R28 | SW_11 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 24 | F40 | Bank 14 | IO_L12N_T1_MRCC_14 | G3 | U1 | T27 | SW_16 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 24 | F40 | Bank 14 | IO_L12P_T1_MRCC_14 | G3 | U1 | T26 | SW_15 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 24 | F40 | Bank 14 | IO_L13N_T2_MRCC_14 | G3 | U1 | U28 | SW_22 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 24 | F40 | Bank 14 | IO_L13P_T2_MRCC_14 | G3 | U1 | U27 | SW_23 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 24 | F40 | Bank 14 | IO_L14N_T2_SRCC_14 | G3 | U1 | U25 | SW_27 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 24 | F40 | Bank 14 | IO_L14P_T2_SRCC_14 | G3 | U1 | T25 | SW_14 | VCC3V3 | 拨码开关 | [S1, PDF p.24] |
| 25 | F40 | Bank 14 | IO_L15N_T2_DQS_DOUT_CSO_B_14 | G3 | U1 | U30 | SW_19 | VCC3V3 | 拨码开关 | [S1, PDF p.25] |
| 25 | F40 | Bank 14 | IO_L15P_T2_DQS_RDWR_B_14 | G3 | U1 | U29 | SW_20 | VCC3V3 | 拨码开关 | [S1, PDF p.25] |
| 25 | F40 | Bank 14 | IO_L16N_T2_A15_D31_14 | G3 | U1 | V27 | SW_24 | VCC3V3 | 拨码开关 | [S1, PDF p.25] |
| 25 | F40 | Bank 14 | IO_L16P_T2_CSI_B_14 | G3 | U1 | V26 | SW_28 | VCC3V3 | 拨码开关 | [S1, PDF p.25] |
| 25 | F40 | Bank 14 | IO_L17N_T2_A13_D29_14 | G3 | U1 | V30 | SW_26 | VCC3V3 | 拨码开关 | [S1, PDF p.25] |
| 25 | F40 | Bank 14 | IO_L17P_T2_A14_D30_14 | G3 | U1 | V29 | SW_25 | VCC3V3 | 拨码开关 | [S1, PDF p.25] |
| 25 | F40 | Bank 14 | IO_L18N_T2_A11_D27_14 | G3 | U1 | W26 | SW_29 | VCC3V3 | 拨码开关 | [S1, PDF p.25] |
| 26 | F40 | Bank 14 | IO_L18P_T2_A12_D28_14 | G3 | U1 | V25 | SW_30 | VCC3V3 | 拨码开关 | [S1, PDF p.26] |
| 26 | F40 | Bank 14 | IO_L19N_T3_A09_D25_VREF_14 | G3 | U1 | V20 | SW_37 | VCC3V3 | 拨码开关 | [S1, PDF p.26] |
| 26 | F40 | Bank 14 | IO_L19P_T3_A10_D26_14 | G3 | U1 | V19 | SW_38 | VCC3V3 | 拨码开关 | [S1, PDF p.26] |
| 26 | F40 | Bank 14 | IO_L1N_T0_D01_DIN_14 | G3 | U1 | R25 | FLASH_IO1 | VCC3V3 | 配置 Flash | [S1, PDF p.26] |
| 26 | F40 | Bank 14 | IO_L1P_T0_D00_MOSI_14 | G3 | U1 | P24 | FLASH_IO0 | VCC3V3 | 配置 Flash | [S1, PDF p.26] |
| 26 | F40 | Bank 14 | IO_L20N_T3_A07_D23_14 | G3 | U1 | W24 | SW_32 | VCC3V3 | 拨码开关 | [S1, PDF p.26] |
| 26 | F40 | Bank 14 | IO_L20P_T3_A08_D24_14 | G3 | U1 | W23 | SW_4 | VCC3V3 | 拨码开关 | [S1, PDF p.26] |
| 27 | F40 | Bank 14 | IO_L21N_T3_DQS_A06_D22_14 | G3 | U1 | U23 | SW_41 | VCC3V3 | 拨码开关 | [S1, PDF p.27] |
| 27 | F40 | Bank 14 | IO_L21P_T3_DQS_14 | G3 | U1 | U22 | SW_2 | VCC3V3 | 拨码开关 | [S1, PDF p.27] |
| 27 | F40 | Bank 14 | IO_L22N_T3_A04_D20_14 | G3 | U1 | V22 | SW_35 | VCC3V3 | 拨码开关 | [S1, PDF p.27] |
| 27 | F40 | Bank 14 | IO_L22P_T3_A05_D21_14 | G3 | U1 | V21 | SW_36 | VCC3V3 | 拨码开关 | [S1, PDF p.27] |
| 27 | F40 | Bank 14 | IO_L23N_T3_A02_D18_14 | G3 | U1 | V24 | SW_31 | VCC3V3 | 拨码开关 | [S1, PDF p.27] |
| 27 | F40 | Bank 14 | IO_L23P_T3_A03_D19_14 | G3 | U1 | U24 | SW_21 | VCC3V3 | 拨码开关 | [S1, PDF p.27] |
| 27 | F40 | Bank 14 | IO_L24N_T3_A00_D16_14 | G3 | U1 | W22 | SW_33 | VCC3V3 | 拨码开关 | [S1, PDF p.27] |
| 27 | F40 | Bank 14 | IO_L24P_T3_A01_D17_14 | G3 | U1 | W21 | SW_34 | VCC3V3 | 拨码开关 | [S1, PDF p.27] |
| 28 | F40 | Bank 14 | IO_L2N_T0_D03_14 | G3 | U1 | R21 | FLASH_IO3 | VCC3V3 | 配置 Flash | [S1, PDF p.28] |
| 28 | F40 | Bank 14 | IO_L2P_T0_D02_14 | G3 | U1 | R20 | FLASH_IO2 | VCC3V3 | 配置 Flash | [S1, PDF p.28] |
| 28 | F40 | Bank 14 | IO_L3N_T0_DQS_EMCCLK_14 | G3 | U1 | R24 | EMCCLK | VCC3V3 | 板载连接/待确认 | [S1, PDF p.28] |
| 28 | F40 | Bank 14 | IO_L3P_T0_DQS_PUDC_B_14 | G3 | U1 | R23 | PUDC_B | VCC3V3 | 板载连接/待确认 | [S1, PDF p.28] |
| 28 | F40 | Bank 14 | IO_L4N_T0_D05_14 | G3 | U1 | T21 | SW_1 | VCC3V3 | 拨码开关 | [S1, PDF p.28] |
| 28 | F40 | Bank 14 | IO_L4P_T0_D04_14 | G3 | U1 | T20 | SW_40 | VCC3V3 | 拨码开关 | [S1, PDF p.28] |
| 28 | F40 | Bank 14 | IO_L5N_T0_D07_14 | G3 | U1 | T23 | SW_5 | VCC3V3 | 拨码开关 | [S1, PDF p.28] |
| 28 | F40 | Bank 14 | IO_L5P_T0_D06_14 | G3 | U1 | T22 | SW_3 | VCC3V3 | 拨码开关 | [S1, PDF p.28] |
| 28 | F40 | Bank 14 | IO_L6N_T0_D08_VREF_14 | G3 | U1 | U20 | SW_39 | VCC3V3 | 拨码开关 | [S1, PDF p.28] |
| 29 | F40 | Bank 14 | IO_L6P_T0_FCS_B_14 | G3 | U1 | U19 | FLASH_NCS | VCC3V3 | 配置 Flash | [S1, PDF p.29] |
| 29 | F40 | Bank 14 | IO_L7N_T1_D10_14 | G3 | U1 | R29 | SW_12 | VCC3V3 | 拨码开关 | [S1, PDF p.29] |
| 29 | F40 | Bank 14 | IO_L7P_T1_D09_14 | G3 | U1 | P29 | SW_9 | VCC3V3 | 拨码开关 | [S1, PDF p.29] |
| 29 | F40 | Bank 14 | IO_L8N_T1_D12_14 | G3 | U1 | P28 | SW_8 | VCC3V3 | 拨码开关 | [S1, PDF p.29] |
| 29 | F40 | Bank 14 | IO_L8P_T1_D11_14 | G3 | U1 | P27 | SW_7 | VCC3V3 | 拨码开关 | [S1, PDF p.29] |
| 29 | F40 | Bank 14 | IO_L9N_T1_DQS_D13_14 | G3 | U1 | T30 | SW_18 | VCC3V3 | 拨码开关 | [S1, PDF p.29] |
| 29 | F40 | Bank 14 | IO_L9P_T1_DQS_14 | G3 | U1 | R30 | SW_13 | VCC3V3 | 拨码开关 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L14N_T2_SRCC_13 | G2 | U1 | AF28 | LED4_CS1 | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L15N_T2_DQS_13 | G2 | U1 | AK30 | LED3_F | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L15P_T2_DQS_13 | G2 | U1 | AK29 | LED3_DP | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L16N_T2_13 | G2 | U1 | AF30 | LED4_DP | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L16P_T2_13 | G2 | U1 | AE30 | LED3_D | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L17N_T2_13 | G2 | U1 | AJ29 | LED3_B | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L17P_T2_13 | G2 | U1 | AJ28 | LED3_CS2 | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L18N_T2_13 | G2 | U1 | AH30 | LED3_CS1 | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 29 | F41 | Bank 13 | IO_L18P_T2_13 | G2 | U1 | AG30 | LED3_E | VADJ1 | LED/数码管 | [S1, PDF p.29] |
| 30 | F41 | Bank 13 | IO_L19P_T3_13 | G2 | U1 | AC26 | LED4_D | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L20N_T3_13 | G2 | U1 | AK28 | LED3_G | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L20P_T3_13 | G2 | U1 | AJ27 | LED3_C | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L21N_T3_DQS_13 | G2 | U1 | AG28 | LED4_A | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L21P_T3_DQS_13 | G2 | U1 | AG27 | LED4_B | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L22N_T3_13 | G2 | U1 | AH27 | LED4_F | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L22P_T3_13 | G2 | U1 | AH26 | LED4_CS2 | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L23N_T3_13 | G2 | U1 | AF27 | LED4_C | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L23P_T3_13 | G2 | U1 | AF26 | LED4_E | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L24N_T3_13 | G2 | U1 | AK26 | LED3_A | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L24P_T3_13 | G2 | U1 | AJ26 | LED4_G | VADJ1 | LED/数码管 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L4N_T0_13 | G2 | U1 | Y29 | STP_DEBUG_2 | VADJ1 | 电源调试 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L5N_T0_13 | G2 | U1 | AB28 | STP_DEBUG_3 | VADJ1 | 电源调试 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L5P_T0_13 | G2 | U1 | AA27 | STP_DEBUG_4 | VADJ1 | 电源调试 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L6P_T0_13 | G2 | U1 | AA25 | STP_DEBUG_1 | VADJ1 | 电源调试 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L7N_T1_13 | G2 | U1 | AC30 | STP_DEBUG_6 | VADJ1 | 电源调试 | [S1, PDF p.30] |
| 30 | F41 | Bank 13 | IO_L8N_T1_13 | G2 | U1 | AA30 | OTG_RESET | VADJ1 | USB OTG | [S1, PDF p.30] |
| 31 | F41 | Bank 13 | IO_L8P_T1_13 | G2 | U1 | Y30 | STP_DEBUG_5 | VADJ1 | 电源调试 | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L11P_T1_SRCC_12 | G1 | U1 | AE23 | OTG_CLK | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L19N_T3_VREF_12 | G1 | U1 | AF21 | N15148 | VADJ2 | 板载连接/待确认 | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L1N_T0_12 | G1 | U1 | Y24 | OTG_DATA7 | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L1P_T0_12 | G1 | U1 | Y23 | OTG_DATA6 | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L2P_T0_12 | G1 | U1 | Y21 | OTG_DATA2 | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L4N_T0_12 | G1 | U1 | AA23 | OTG_DATA3 | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L4P_T0_12 | G1 | U1 | AA22 | OTG_STP | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L5N_T0_12 | G1 | U1 | AC21 | OTG_NXT | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L5P_T0_12 | G1 | U1 | AC20 | OTG_DATA0 | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L6N_T0_VREF_12 | G1 | U1 | AB20 | N15136 | VADJ2 | 板载连接/待确认 | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L7N_T1_12 | G1 | U1 | AC25 | OTG_DATA5 | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L7P_T1_12 | G1 | U1 | AB24 | OTG_DATA4 | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L8N_T1_12 | G1 | U1 | AD22 | OTG_DATA1 | VADJ2 | USB OTG | [S1, PDF p.31] |
| 31 | F42 | Bank 12 | IO_L8P_T1_12 | G1 | U1 | AC22 | OTG_DIR | VADJ2 | USB OTG | [S1, PDF p.31] |

## 5. 按功能分类的引脚

### 5.1 电源与地

| 类别 | 可用位置 | 结论 |
|---|---|---|
| 3.3 V | J7/J8/J9/J10 pin 20 | 可给低功耗 3.3 V 模块供电；单口和总扩展负载能力待确认 |
| GND | J7/J8/J9/J10 pin 11-19 | 每口 9 根 GND；外部 5 V 电源必须与板卡共地 |
| 5 V | DEBUG 口没有 5 V | HC-SR04 需外部稳压 5 V，或使用经确认的板上 5 V 接点；不得从 DEBUG pin 20 供电 |
| 12 V | CN1/DC12V_IN、H1 风扇口 | 板卡和风扇电源，不用于小型 3.3/5 V 传感器 |

### 5.2 普通 GPIO

- 唯一明确面向用户的通用扩展 GPIO 是 J7-J10 的 `DEBUG_1..40`。
- 它们均可由 RTL 实现输入、输出、双向、同步采样、消抖或中断，但必须先确认 VADJ1 和 XDC IOSTANDARD。
- 每根线已有 100 Ω 串联和 TVS；不要再次把它们当作无保护的 FPGA 球位直接计算高速阻抗。
- 其他 `SW_*`、`KEY*`、`LED*`、HDMI、VGA、Ethernet、TF、USB、DDR、Flash 信号均已被板载电路占用。

### 5.3 ADC

- 没有文档确认的外部 ADC 排针。
- 当前 GY-302/BH1750 是数字 I²C 光照传感器，不需要 ADC。
- 若以后使用模拟光照传感器，应增加外部 I²C/SPI ADC，不要直接接 DEBUG 口。

### 5.4 PWM 与定时器

- 没有固定的外部 PWM/Timer 专用脚。
- 任一空闲 DEBUG I/O 可由 FPGA 逻辑产生 PWM 或作为输入捕获。
- 当前方案使用 `DEBUG_31/A22` 控制 HC-SR04 TRIG，`DEBUG_32/A21` 作为 ECHO 脉宽捕获。

### 5.5 I²C

| 用途 | SCL | SDA | 要求 |
|---|---|---|---|
| 当前传感器总线 | J10-9 / DEBUG_39 / F22 | J10-10 / DEBUG_40 / G22 | 开漏；上拉电压必须匹配 VADJ1；GY-302 与 OLED 并联 |
| 备用 I²C | 任意两根未占用 DEBUG I/O | 任意两根未占用 DEBUG I/O | 需要在 RTL/XDC 中重新定义 |

板上 HDMI DDC 也有 SCL/SDA，但已连接 HDMI 电平/保护电路，不作为通用传感器 I²C。

### 5.6 SPI

- 配置 Flash SPI 已被启动存储占用，不能直接复用。
- 可在任意四根空闲 DEBUG I/O 上实现 SPI SCLK/MOSI/MISO/CS。
- 当前三个模块都不需要 SPI。

### 5.7 UART

| UART | FPGA RX | FPGA TX | 状态 |
|---|---|---|---|
| 板载 USB-UART | D18 / CP2104_TXD | D17 / CP2104_RXD | 已连接 J4，适合 MSH/日志 |
| 自定义传感器 UART | 任意空闲 DEBUG 输入 | 任意空闲 DEBUG 输出 | 只允许与 VADJ1 兼容的逻辑电平 |

### 5.8 中断、复位与启动相关引脚

| 资源 | 引脚/网络 | 作用 | 注意事项 |
|---|---|---|---|
| 外部中断 | 任一空闲 DEBUG I/O | PL 内边沿检测并接软核中断控制器 | 无固定 IRQ 脚 |
| PROGRAM_B | K10 | 重新配置 FPGA | 不是 GPIO |
| INIT_B | A10 | 配置初始化状态 | FunctionPin 网络名为 `N29636`，用途需按配置规范处理 |
| DONE | M10 | 配置完成 | 已接板载指示电路 |
| JTAG | E10/G10/F10/H10 | 下载/调试 | 保留给工具链 |
| Flash | B10/P24/R25/R20/R21/U19 | 上电配置 | 不得在应用中当普通 SPI 外设复用 |

## 6. 板载器件占用情况

| 资源 | 占用范围 | 是否建议复用 |
|---|---|---|
| DDR3 | Bank32/33/34 大部分 I/O | 否 |
| 配置 Flash/JTAG | Bank0 和 Bank14 配置专用脚 | 否 |
| Ethernet | Bank17/18 多根 I/O | 否 |
| USB-UART | D17/D18 | 保留作串口调试 |
| HDMI/TF/32 LED | Bank16 | 否，除非删除对应板载功能并评估电路负载 |
| 64 拨码开关/VGA | Bank14/15 | 否，存在固定上拉或电阻 DAC |
| 数码管/按键 | Bank13/18 | 否，存在固定板载负载 |
| USB Host/OTG | Bank12/13 | 否 |
| DEBUG_1..40 | Bank17 | 是；用户扩展专用，但需确认 VADJ1 |

## 7. 现有传感器接线建议

### 7.1 推荐统一分配

| 功能 | 板卡连接 | FPGA 脚 | 模块侧 | 额外电路 |
|---|---|---|---|---|
| I²C SCL | J10-9 / DEBUG_39 | F22 | GY-302 SCL + OLED SCL | 开漏上拉；先确认 VADJ1 |
| I²C SDA | J10-10 / DEBUG_40 | G22 | GY-302 SDA + OLED SDA | 开漏上拉；先确认 VADJ1 |
| 3.3 V | J10-20 | - | GY-302 VCC + OLED VCC | 两模块允许 3.3 V 仍需实物确认 |
| GND | J10-11..19 任一 | - | 三个模块 GND | 外部 HC-SR04 5 V 电源也在此共地 |
| HC-SR04 TRIG 控制 | J10-1 / DEBUG_31 | A22 | 经隔离电路到 TRIG | 处理模块内部 5 V 上拉 |
| HC-SR04 ECHO 输入 | J10-2 / DEBUG_32 | A21 | ECHO 经 5 V->VADJ1 转换 | 不得直连 |
| HC-SR04 VCC | 外部稳压 5 V | - | VCC | DEBUG 口无 5 V；电源负极接板卡 GND |

### 7.2 上板前必须确认 VADJ1

1. 板卡断电，确认 J10 pin 20 是固定 3.3 V 电源脚，pin 11-19 是 GND。
2. 板卡上电但暂不接传感器，测量 TP5（VADJ1）对 GND。
3. 若 TP5 为 3.3 V，可把 Bank17 设为与工程一致的 3.3 V I/O 标准，并让 I²C 上拉到 3.3 V。
4. 若 TP5 不是 3.3 V，OLED/GY-302 与 DEBUG_39/40 之间必须增加双向 I²C 电平转换；HC-SR04 两根信号也按 VADJ1 侧电压转换。
5. 测量结果和当前 XDC 不一致时，停止上板，不要依靠 TVS 钳位承受持续过压。

### 7.3 GY-302/BH1750

| 模块丝印 | 连接 | 说明 |
|---|---|---|
| VCC | J10-20 3.3 V | 图片未写供电范围；初次只使用 3.3 V |
| GND | J10-11..19 | 共地 |
| SCL | J10-9 / F22 | 与 OLED 并联 |
| SDA | J10-10 / G22 | 与 OLED 并联 |
| ADDR | GND | 推荐地址 `0x23`；接高电平时为 `0x5C` [I4] |

五针的实际排列顺序必须以模块 PCB 丝印为准。图片只给出名称定义，没有可靠展示实物针序。[I3]、[I4]

### 7.4 四针 I²C OLED

| 模块丝印 | 连接 | 说明 |
|---|---|---|
| GND | J10-11..19 | 共地 |
| VCC | J10-20 3.3 V | 图片未给供电范围；未确认前不接 5 V |
| SCL | J10-9 / F22 | 与 GY-302 并联 |
| SDA | J10-10 / G22 | 与 GY-302 并联 |

仅凭图片不能确认 OLED 控制器、分辨率和地址。软件应先扫描 `0x08-0x77`，再根据芯片标识或商品资料选择驱动，不能直接认定为 SSD1306 或固定为常见地址。[I2]

### 7.5 I²C 上拉检查

1. DEBUG_39/40 上没有原理图标注的专用 I²C 上拉，只有 100 Ω 串联和 TVS。
2. GY-302 照片中可见 `472` 电阻，但单凭照片不能确认它们是否为 SDA/SCL 上拉。[I3]
3. OLED 是否带上拉待确认。
4. 断电测量 SCL/SDA 到模块 VCC 的等效电阻；若没有上拉，再各加约 4.7 kΩ 到与 Bank17 兼容的电源。
5. FPGA 端释放总线后，实测空闲高电平必须与 VADJ1/I/O 标准兼容。
6. I²C RTL 只能主动输出 0，输出 1 时必须切换为高阻态。
7. 初次以 100 kHz 调试，同时用示波器查看 100 Ω 串联电阻后的上升沿。

### 7.6 HC-SR04 电平处理

HC-SR04 图片明确标注 4.5-5.5 V，且 TRIG/ECHO 都有 10 kΩ 内部上拉。[I1] 因此两根线都不能直接接 DEBUG 口。

推荐方案：

| HC-SR04 信号 | 处理方式 | FPGA 侧 |
|---|---|---|
| VCC | 外部稳压 5 V | - |
| GND | 与 J10 GND 共地 | - |
| TRIG | 独立单向电平转换；或 N 沟道 MOSFET 开漏下拉，栅极上拉到 VADJ1 | J10-1/A22 |
| ECHO | 独立 5 V 到 VADJ1 单向转换；若 VADJ1=3.3 V，可最低限度用分压 | 分压后接 J10-2/A21 |

当 VADJ1 实测为 3.3 V时，ECHO 可用 5.1 kΩ 上臂和 10 kΩ 下臂，把理想 5 V 降到约 3.31 V。该分压仍需用示波器确认模块实际高电平和边沿；专用电平转换器更稳妥。

TRIG 的 MOSFET 反相控制示例：源极接 GND、漏极接模块 TRIG、栅极接 A22，并用约 10 kΩ 将栅极上拉到 VADJ1。栅极高时 TRIG 被拉低；触发时 FPGA 将栅极拉低至少 10 us，使模块内部上拉产生高脉冲，再恢复高电平。RTL 端口宜命名为 `hcsr04_trig_gate_o`，明确其反相含义。

### 7.7 测距时序

1. 保持模块 TRIG 低。
2. 产生至少 10 us 的 TRIG 高脉冲；MOSFET 反相方案对应栅极低脉冲。
3. ECHO 输入先经过两级同步器，再进行上升沿/下降沿检测。
4. 计数 ECHO 高电平时间；室温近似 `distance_cm = echo_time_us / 58.3`。
5. 450 cm 往返时间约 26.2 ms，可先设 30 ms 超时、60 ms 测量周期。
6. 软件区分正常、等待上升沿超时、等待下降沿超时和超范围状态。

### 7.8 XDC 示例

下面的 IOSTANDARD **只在 TP5 实测 VADJ1=3.3 V 且与板卡配置一致时使用**：

```tcl
set_property -dict {PACKAGE_PIN F22 IOSTANDARD LVCMOS33} [get_ports sensor_i2c_scl_io]
set_property -dict {PACKAGE_PIN G22 IOSTANDARD LVCMOS33} [get_ports sensor_i2c_sda_io]
set_property -dict {PACKAGE_PIN A22 IOSTANDARD LVCMOS33} [get_ports hcsr04_trig_gate_o]
set_property -dict {PACKAGE_PIN A21 IOSTANDARD LVCMOS33} [get_ports hcsr04_echo_i]
```

建议顶层端口：

```systemverilog
inout  wire sensor_i2c_scl_io;
inout  wire sensor_i2c_sda_io;
output wire hcsr04_trig_gate_o;
input  wire hcsr04_echo_i;
```

### 7.9 驱动和调试顺序

1. 暂不连接 HC-SR04，先测 VADJ1。
2. 连接 GY-302 和 OLED，确认 SCL/SDA 空闲电压。
3. 运行 I²C 扫描，GY-302 ADDR 接地时应检查 `0x23`；记录 OLED 实际地址。
4. 单独读取 BH1750，再单独初始化已确认控制器型号的 OLED。
5. 断电后接入 HC-SR04 电平转换电路，用示波器确认 A22/A21 端不超过 Bank17 允许电平。
6. 单独验证 TRIG 和 ECHO 脉宽计数，最后把距离与照度显示到 OLED。

若运行 RT-Thread/MSH，可规划：`i2c_scan`、`lux_read`、`range_read`、`oled_test`、`sensor_start`、`sensor_stop`。

## 8. 冲突与待确认事项

| 编号 | 冲突/不确定项 | 来源 A | 来源 B | 处理原则 |
|---:|---|---|---|---|
| 1 | 板卡工程版本 | FunctionPin Design Name 指向 `EDABOX2.2/EDABOX2.1.brd` [S1, PDF p.1] | 原理图 PDF 元数据为 `edabox2.4` [S2] | 必须确认实物 PCB 版本和当前 XDC；不要把不同版本资料自动视为完全一致 |
| 2 | DEBUG 电平 | J7-J10 pin20 是固定 +3.3 V [S2, PDF p.17] | Bank17 VCCO 是 VADJ1，文档未写具体电压 [S2, PDF p.7、p.18] | 测 TP5 后确定 IOSTANDARD；未测前禁止直接接 3.3 V 上拉模块 |
| 3 | KEY1-KEY8 封装脚 | FunctionPin 给出 E15/C14/D14/A13/B13/E14/C11/D11 [S1, PDF p.9-p.10] | 原理图 p.16 的 `FPGA_PIN` 文字标注显示另一组且存在重复 | 不自行裁决；按键相关 XDC 需结合实物/网表/既有工程验证 |
| 4 | HC-SR04 5 V 供电入口 | 板上存在 +5 V 电源轨 [S2, PDF p.15、p.18] | DEBUG 排针没有 5 V [S2, PDF p.17] | 优先外部 5 V 共地；使用板上测试点或 USB VBUS 前必须确认物理接点和供电方向 |
| 5 | FPGA 单脚电流和绝对最大电压 | 两份板卡资料未给 XC7K325T 完整 DC 参数 | - | 查对应 Xilinx 数据手册；TVS/100 Ω 不能替代电平转换 |
| 6 | DEBUG 排针 3.3 V 可供电流 | 3.3 V 总稳压器标注 3 A [S2, PDF p.18] | 没有单排针/连接器额定值 | 不把 3 A 当作 J10 可用电流；低功耗传感器也需核算全板余量 |
| 7 | OLED 型号 | 图片仅确认四针 I²C OLED [I2] | 无控制器、分辨率、地址、供电资料 | 先扫描和识别芯片，再选择驱动 |
| 8 | GY-302 上拉和供电 | 图片可见 BH1750、`472` 电阻 [I3] | 无模块原理图/供电范围 | 初次按 3.3 V；实测上拉和空闲电平 |
| 9 | HC-SR04 TRIG/ECHO | 图片称两脚均有 10 kΩ 上拉，模块 4.5-5.5 V [I1] | DEBUG 是 VADJ1 I/O | 两根线均做隔离/转换，不只处理 ECHO |
| 10 | ADC | 器件具备 XADC/辅助模拟能力 | 板上没有经确认的模拟扩展口，AD 管脚接拨码开关 | 模拟传感器增加外部 ADC |

### 8.1 上板前最低检查清单

- [ ] 实物 PCB 版本与 FunctionPin/XDC/原理图版本已核对。
- [ ] TP5 的 VADJ1 已实测并记录。
- [ ] XDC IOSTANDARD 与 VADJ1 一致。
- [ ] GY-302、OLED 的 SDA/SCL 空闲电平与 VADJ1 兼容。
- [ ] OLED 控制器和地址已确认，不按外观猜测。
- [ ] HC-SR04 使用 5 V，且与板卡共地。
- [ ] HC-SR04 TRIG/ECHO 均有电平处理。
- [ ] J10 pin20 只给 3.3 V 模块供电，不接 HC-SR04 VCC。
- [ ] 所有模块均在断电状态完成插拔。
- [ ] 当前 XDC 中 A22、A21、F22、G22 没有重复绑定。

## 9. 信息来源索引

### [S1] FunctionPin-V2.1.pdf

- 路径：`D:/IKnow/study/grade_two_second/JingYeDa_competition/2026年资料/数字孪生平台工程模板/FunctionPin-V2.1.pdf`
- 共 31 页；Function Pin Report。
- PDF p.1-p.9：配置、DDR、系统时钟。
- PDF p.9-p.12：Ethernet、按键、LED1/LED2 数码管。
- PDF p.12-p.15：CP2104 和 DEBUG_1..40。
- PDF p.15-p.20：LED1..32、HDMI、TF。
- PDF p.20-p.23：VGA、SW_42..64。
- PDF p.23-p.29：SW_1..41、配置 Flash。
- PDF p.29-p.31：LED3/LED4、USB OTG。

### [S2] xc7k325t-V2.1.pdf

- 路径：`D:/IKnow/study/grade_two_second/JingYeDa_competition/2026年资料/数字孪生平台工程模板/xc7k325t-V2.1.pdf`
- 共 18 页；原理图 PDF。
- PDF p.1：目录和电源树。
- PDF p.2：配置 Flash 与 JTAG。
- PDF p.3-p.7：FPGA Banks、时钟、DDR 和电源域。
- PDF p.8：HDMI。
- PDF p.9-p.10：DDR3。
- PDF p.11：Ethernet。
- PDF p.12：CP2104 USB-UART。
- PDF p.13：LED、TF、数码管、蜂鸣器、风扇。
- PDF p.14：VGA。
- PDF p.15：USB Host/OTG。
- PDF p.16：64 位拨码开关和 8 个按键。
- PDF p.17：J7-J10 DEBUG 扩展口、100 Ω 和 TVS。
- PDF p.18：电源、测试点和电源调试口。

### [I1] HC-SR04 图片

- 文件：`codex-clipboard-feed2acc-82eb-44ff-9b8a-1c993930f726.jpg`
- 用于确认四针、4.5-5.5 V、TRIG 时序、内部上拉、量程和电流。

### [I2] 四针 I²C OLED 图片

- 文件：`codex-clipboard-49be16b5-b4c4-4c3c-83a7-102ad8cc2642.jpg`
- 只确认 0.96 英寸、蓝色、I²C、`GND/VCC/SCL/SDA`。

### [I3] GY-302 实物图片

- 文件：`codex-clipboard-8ea6412f-b54e-41e2-842a-0dd95527e07a.jpg`
- 用于确认 GY-302/BH1750 板级标识和可见电阻标识。

### [I4] GY-302 引脚图片

- 文件：`codex-clipboard-20519bf0-4f25-44f4-92f4-792bbefe4ee5.jpg`
- 用于确认 `VCC/GND/SCL/SDA/ADDR` 和地址 `0x23/0x5C`。

---

**当前推荐方案：**J10-9/F22 与 J10-10/G22 作为共享 I²C，连接 GY-302 和 OLED；J10-1/A22 与 J10-2/A21 连接经电平处理的 HC-SR04 TRIG/ECHO；OLED/GY-302 由 J10-20 的 3.3 V 供电，HC-SR04 使用外部 5 V 并与 J10 GND 共地。所有连接必须在 VADJ1 和当前 XDC 电气标准确认后实施。
