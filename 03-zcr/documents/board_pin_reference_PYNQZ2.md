# PYNQ-Z2 板卡与现有传感器接线参考

> 适用对象：TUL PYNQ-Z2（主板）以及可选的 EES_363DP 数字子卡（Arduino 扩展板）。  
> 主要用途：为 HC-SR04、GY-302（BH1750）和四针 I²C OLED 的接线、FPGA 约束、驱动开发及调试提供可直接检索的依据。  
> 页码规则：本文中的“PDF p.x”指 PDF 文件从封面开始计数的实际页码。  
> 结论边界：未被文档明确说明的电流能力、复用控制器或电气参数均标为“待确认”，不按经验补全。

## 1. 板卡概述

| 项目 | 信息 | 来源 |
|---|---|---|
| 主板型号 | PYNQ-Z2 | [S1, PDF p.1]；[S2, PDF p.1] |
| 原理图版本 | PYNQ-Z2 R10，日期 2018-01-09 | [S1, PDF p.1] |
| 参考手册版本 | PYNQ-Z2 Reference Manual v1.0，日期 2018-05-17 | [S2, PDF p.1] |
| 主控 | Xilinx Zynq-7000 XC7Z020-1CLG400C | [S1, PDF p.9、p.11]；[S2, PDF p.4] |
| 处理系统 | 双核 ARM Cortex-A9，最高 650 MHz | [S2, PDF p.4] |
| 可编程逻辑 | 13,300 slices、630 KB Block RAM、220 DSP slices、片上 XADC | [S2, PDF p.4] |
| DDR3 | Micron MT41K512M16HA-125:A，512 MB，16 bit，总线最高 525 MHz/1050 Mbps | [S1, PDF p.13]；[S2, PDF p.10] |
| QSPI Flash | Spansion S25FL128S，16 MB，3.3 V，MIO[1:6,8] | [S2, PDF p.11] |
| 用户扩展接口 | 2 个 Pmod、Arduino Shield 接口、40 pin Raspberry Pi 接口、XADC 接口 | [S2, PDF p.4-p.5、p.23-p.30] |
| 可选数字子卡 | EES_363DP，PYNQ-Z2 专用 Arduino 扩展板，带四位八段数码管、6 个拨码开关、8 个 LED | [S3, PDF p.3-p.6] |

### 1.1 与传感器连接最相关的结论

- Pmod、Arduino 数字接口和 Raspberry Pi 排针上的可编程逻辑信号均按 **3.3 V 逻辑**使用，不能直接接受 5 V 数字输出。
- Arduino A0-A5 可测量 0-3.3 V 单端模拟信号；板上分压后送入 XADC。专用 XADC `V_P/V_N` 只允许 0-1 V 差分输入，且不能作为数字 GPIO。
- PYNQ-Z2 的 Arduino、Pmod 和 Raspberry Pi 信号主要连接到 PL。UART、I²C、SPI、PWM、定时器和中断是否存在，取决于 FPGA bitstream 中是否实现相应控制器，并不是仅接线就自动具备。
- 板载 PS UART0 已连接 FT2232HL USB-UART，用于电脑串口；它不是空闲的外部传感器 UART 接口。
- 如果装有 EES_363DP 数字子卡，多个 Arduino GPIO、I²C 和 SPI 标号引脚已被数码管、LED 和拨码开关使用。新增传感器优先选择 Pmod B，或拆下数字子卡。

### 1.2 当前已有模块

| 模块 | 图片中可确认的型号/接口 | 已确认参数 | 尚待确认 |
|---|---|---|---|
| 超声波测距 | HC-SR04；`VCC/TRIG/ECHO/GND` | DC 4.5-5.5 V；工作电流约 5 mA；静态电流小于 2 mA；TRIG 脉冲至少 10 us；探测范围 2-450 cm；ECHO 和 TRIG 各有内部 10 kΩ 上拉 | ECHO 输出驱动结构、不同批次实际时序误差 [I1] |
| 光照强度 | GY-302，板上标识 BH1750；I²C；`VCC/GND/SCL/SDA/ADDR` | ADDR 低时 7-bit 地址 `0x23`；ADDR 高时 7-bit 地址 `0x5C` | 模块允许供电范围、板载上拉的准确连接关系 [I3]、[I4] |
| 显示屏 | 0.96 英寸、蓝色、四针 I²C；`GND/VCC/SCL/SDA` | 只有 I²C 接口和四针丝印可由图片确认 | 控制器型号、分辨率、7-bit 地址、允许供电范围均待确认 [I2] |

本文采用一套统一方案：GY-302 与 OLED 共用 PMODB 上的 I²C 总线，HC-SR04 使用 PMODB 的另外两根 GPIO；这样即使 EES_363DP 数字子卡安装在 Arduino 接口上，也不会发生信号占用冲突。

## 2. 电气特性与使用限制

### 2.1 数字 I/O

| 项目 | 限制或说明 | 来源 |
|---|---|---|
| 逻辑电平 | 扩展数字 I/O 按 3.3 V 逻辑使用 | [S2, PDF p.23-p.26] |
| Arduino 数字引脚推荐工作范围 | 上电时 -0.2 V 至 3.4 V | [S2, PDF p.26，Table 15] |
| Arduino 数字引脚绝对范围 | 上电时 -0.4 V 至 3.75 V；断电时 -0.4 V 至 0.55 V | [S2, PDF p.26，Table 15] |
| Arduino 数字引脚串联保护 | FPGA 与 Arduino 数字 I/O 之间有 200 Ω 串联电阻 | [S1, PDF p.4]；[S2, PDF p.25] |
| 5 V 传感器输出 | 禁止直接送入 PL/Arduino/Pmod/Raspberry Pi GPIO；应使用分压器、单向缓冲器或电平转换器 | [S2, PDF p.25-p.26] |
| 单个 PL I/O 最大输出电流 | 文档未给出，待查 XC7Z020 数据手册；不得用 GPIO 直接驱动电机、继电器、蜂鸣器或大功率 LED | 待确认 |
| Pmod 电源能力 | 每个 Pmod 的 VCC/GND 电源针脚组可提供最高 1 A；这是电源针脚能力，不是 FPGA I/O 驱动电流 | [S2, PDF p.23] |
| 外部上电倒灌 | 板卡断电时 Arduino 数字脚绝对最大值仅 0.55 V；外设不得在主板断电时继续向 GPIO 驱动高电平 | [S2, PDF p.26，Table 15] |

### 2.2 模拟输入

| 接口 | 输入范围 | 数字功能 | 板上网络 | 注意事项 | 来源 |
|---|---:|---|---|---|---|
| Arduino A0-A5 | 0-3.3 V，单端对板卡 GND | 可作为数字 I/O | 外部电阻网络把 0-3.3 V 缩放到 XADC 范围；同时在分压前连接 PL 数字节点 | 不得超过 3.3 V；作为数字输出时还会连接模拟分压网络 | [S1, PDF p.4]；[S2, PDF p.27-p.29] |
| J5 XADC `V_P/V_N` | 0-1 V 差分 | 不可作为数字 I/O | 140 Ω 电阻；文档注明仅 `V_P/V_N` 装有 1 nF 电容 | 禁止接 3.3 V 或 5 V 模拟输出 | [S1, PDF p.4、p.9]；[S2, PDF p.28-p.29] |
| 文档所称其他差分模拟通道 | 待确认 | 待确认 | 手册称另有六组差分模拟输入，但对“AR0-AR13”的表述与连接器标号不够明确 | 使用前必须结合实际 XDC 和原理图网络复核 | [S2, PDF p.25、p.28] |

### 2.3 供电与电源入口

| 电源/接口 | 电压或用途 | 限制 | 来源 |
|---|---|---|---|
| Micro-USB J8 | 可给板卡供电，同时连接 USB-JTAG/UART | J9 跳线置 USB；高负载设计可能超过电脑 USB 供电能力 | [S2, PDF p.7、p.17] |
| DC1 圆孔电源 | 7-15 VDC，中心正极，12 V 推荐 | J9 跳线置 REG | [S2, PDF p.7] |
| Arduino J7 `VIN/VU_CK` | 可连接电池正极 | J9 置 REG；手册没有单独给出该引脚允许的电池电压范围，待确认 | [S1, PDF p.4]；[S2, PDF p.7、p.27] |
| 3.3 V 排针 | Pmod 6/12、Arduino J7 5/7、Arduino SPI 2、Raspberry Pi 1/17 | 传感器优先使用 3.3 V；各接口合计电流上限未完整给出 | [S2, PDF p.23-p.30] |
| 5 V 排针 | Arduino J7-4、Raspberry Pi 2/4 | 5 V 只能用于供电；不得因此把 5 V 输出信号直接接入 3.3 V GPIO | [S1, PDF p.4]；[S2, PDF p.27、p.30] |
| GND | 各扩展接口均提供地 | 传感器与板卡必须共地 | [S2, PDF p.23-p.30] |

## 3. 接口与连接器位置说明

| 接口 | 连接器 | 位置/用途 | 来源 |
|---|---|---|---|
| Pmod A、Pmod B | 2x6、2.54 mm、直角母座 | 板边双排扩展口；每口 8 个 PL I/O、2 个 3.3 V、2 个 GND | [S2, PDF p.23-p.24] |
| Arduino Shield | J1、J3、J4、J7、SPI_CNN1 | 板底部 Arduino 兼容排母；数字 GPIO、I²C、SPI、模拟输入和电源 | [S1, PDF p.4]；[S2, PDF p.25-p.29] |
| XADC | J5，1x4 | Arduino 接口区域内，专用 `V_P/V_N`、模拟地和参考端 | [S1, PDF p.4]；[S2, PDF p.29] |
| Raspberry Pi | RPI_IDE，2x20 | 板顶部 40 pin 排针，28 个信号连接 Zynq PL；8 个信号与 Pmod A 共享 | [S1, PDF p.4]；[S2, PDF p.30] |
| PROG UART | J8 Micro-USB | 板载 FT2232HL，提供 JTAG、PS UART0 和 USB 供电 | [S1, PDF p.8]；[S2, PDF p.17] |
| HDMI | 源/输出与接收/输入 Type-A | 直接连接 PL；含 TMDS、CEC、DDC、HPD 和 5 V | [S1, PDF p.6]；[S2, PDF p.18-p.19] |
| EES_363DP | 插入 Arduino Shield 接口 | 可选子卡，装上后占用多根 Arduino 引脚 | [S3, PDF p.3-p.6] |

## 4. 全部引脚总表

### 4.1 Pmod A（PMODA）

所有信号均为 PL 3.3 V I/O；可在 FPGA 中实现 GPIO、PWM、定时器输入、外部中断、UART、I²C 或 SPI。`_P/_N` 表示成对布线名称，不代表当前 bitstream 已启用差分标准。

| 接口 | 物理引脚 | GPIO/网络名 | 主控封装引脚 | 默认功能 | 复用/可实现功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| PMODA | 1 | JA1_P | Y18 | PL I/O | GPIO/PWM/UART/I²C/SPI/中断 | 3.3 V | 与 RPi `RPIO_04` 共享 | 使用任一接口会影响另一接口 | [S1, PDF p.4]；[S2, PDF p.24] |
| PMODA | 2 | JA1_N | Y19 | PL I/O | 同上 | 3.3 V | 与 RPi `RPIO_05` 共享 | 同上 | [S1, PDF p.4]；[S2, PDF p.24] |
| PMODA | 3 | JA2_P | Y16 | PL I/O | 同上 | 3.3 V | 与 RPi `RPIO_SD` 共享 | 同上 | [S1, PDF p.4]；[S2, PDF p.24] |
| PMODA | 4 | JA2_N | Y17 | PL I/O | 同上 | 3.3 V | 与 RPi `RPIO_SC` 共享 | 同上 | [S1, PDF p.4]；[S2, PDF p.24] |
| PMODA | 5 | GND | - | 地 | - | 0 V | - | - | [S2, PDF p.23-p.24] |
| PMODA | 6 | 3V3 | - | 电源 | - | 3.3 V | - | Pmod VCC/GND 针脚组总能力最高 1 A | [S2, PDF p.23-p.24] |
| PMODA | 7 | JA3_P | U18 | PL I/O | GPIO/PWM/UART/I²C/SPI/中断 | 3.3 V | 与 RPi `RPIO_06` 共享 | 使用任一接口会影响另一接口 | [S1, PDF p.4]；[S2, PDF p.24] |
| PMODA | 8 | JA3_N | U19 | PL I/O | 同上 | 3.3 V | 与 RPi `RPIO_07` 共享 | 同上 | [S1, PDF p.4]；[S2, PDF p.24] |
| PMODA | 9 | JA4_P | W18 | PL I/O | 同上 | 3.3 V | 与 RPi `RPIO_02` 共享 | 同上 | [S1, PDF p.2、p.4]；[S2, PDF p.24] |
| PMODA | 10 | JA4_N | W19 | PL I/O | 同上 | 3.3 V | 与 RPi `RPIO_03` 共享 | 同上 | [S1, PDF p.2、p.4]；[S2, PDF p.24] |
| PMODA | 11 | GND | - | 地 | - | 0 V | - | - | [S2, PDF p.23-p.24] |
| PMODA | 12 | 3V3 | - | 电源 | - | 3.3 V | - | Pmod VCC/GND 针脚组总能力最高 1 A | [S2, PDF p.23-p.24] |

### 4.2 Pmod B（PMODB）

Pmod B 不与 Raspberry Pi 排针共享，通常是连接独立小型传感器最稳妥的接口。

| 接口 | 物理引脚 | GPIO/网络名 | 主控封装引脚 | 默认功能 | 复用/可实现功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| PMODB | 1 | JB1_P | W14 | PL I/O | GPIO/PWM/UART/I²C/SPI/中断 | 3.3 V | 无专用占用 | 需要 bitstream/XDC | [S1, PDF p.3、p.11]；[S2, PDF p.24] |
| PMODB | 2 | JB1_N | Y14 | PL I/O | 同上 | 3.3 V | 无专用占用 | 同上 | 同上 |
| PMODB | 3 | JB2_P | T11 | PL I/O | 同上 | 3.3 V | 无专用占用 | 同上 | 同上 |
| PMODB | 4 | JB2_N | T10 | PL I/O | 同上 | 3.3 V | 无专用占用 | 同上 | 同上 |
| PMODB | 5 | GND | - | 地 | - | 0 V | - | - | [S2, PDF p.23-p.24] |
| PMODB | 6 | 3V3 | - | 电源 | - | 3.3 V | - | Pmod VCC/GND 针脚组总能力最高 1 A | [S2, PDF p.23-p.24] |
| PMODB | 7 | JB3_P | V16 | PL I/O | GPIO/PWM/UART/I²C/SPI/中断 | 3.3 V | 无专用占用 | 需要 bitstream/XDC | [S1, PDF p.3、p.11]；[S2, PDF p.24] |
| PMODB | 8 | JB3_N | W16；手册印为 `W156` | PL I/O | 同上 | 3.3 V | 无专用占用 | 文档冲突，使用前核对 XDC | [S1, PDF p.11]；[S2, PDF p.24] |
| PMODB | 9 | JB4_P | V12 | PL I/O | 同上 | 3.3 V | 无专用占用 | 需要 bitstream/XDC | [S1, PDF p.3、p.11]；[S2, PDF p.24] |
| PMODB | 10 | JB4_N | W13 | PL I/O | 同上 | 3.3 V | 无专用占用 | 同上 | 同上 |
| PMODB | 11 | GND | - | 地 | - | 0 V | - | - | [S2, PDF p.23-p.24] |
| PMODB | 12 | 3V3 | - | 电源 | - | 3.3 V | - | Pmod VCC/GND 针脚组总能力最高 1 A | [S2, PDF p.23-p.24] |

### 4.3 Arduino Shield 数字与电源排针

#### J3

| 接口 | 物理引脚 | GPIO/网络名 | 主控封装引脚 | 默认功能 | 复用/可实现功能 | 电压 | EES_363DP 占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| Arduino J3 | 1 | AR8 | V17 | PL I/O | GPIO/PWM/定时器/中断 | 3.3 V | 数码管段选/LED8 | 200 Ω 串联；子卡安装时不可自由使用 | [S1, PDF p.4]；[S2, PDF p.26]；[S3, PDF p.4、p.6] |
| Arduino J3 | 2 | AR9 | V18 | PL I/O | GPIO/PWM/定时器/中断 | 3.3 V | 未在子卡表中列出 | 200 Ω 串联 | [S1, PDF p.4]；[S2, PDF p.26] |
| Arduino J3 | 3 | AR10 | T16 | PL I/O | GPIO/PWM/定时器/中断 | 3.3 V | SW6 输入 | 子卡安装时被占用 | [S2, PDF p.26]；[S3, PDF p.5] |
| Arduino J3 | 4 | AR11 | R17 | PL I/O | GPIO/PWM/定时器/中断 | 3.3 V | SW5 输入 | 子卡安装时被占用 | 同上 |
| Arduino J3 | 5 | AR12 | P18 | PL I/O | GPIO/PWM/定时器/中断 | 3.3 V | SW4 输入 | 子卡安装时被占用 | 同上 |
| Arduino J3 | 6 | AR13 | N17 | PL I/O | GPIO/PWM/定时器/中断 | 3.3 V | SW3 输入 | 子卡安装时被占用 | 同上 |
| Arduino J3 | 7 | GND | - | 地 | - | 0 V | 子卡供电回路 | - | [S2, PDF p.26] |
| Arduino J3 | 8 | A / AR_IOA | Y13 | PL I/O | GPIO/PWM/定时器/中断 | 3.3 V | 未在子卡表中列出 | 名称 `A` 的用途未进一步说明，待确认 | [S1, PDF p.4、p.11]；[S2, PDF p.26] |
| Arduino J3 | 9 | AR_SDA | P16 | PL I/O，Arduino SDA 标号 | I²C SDA/GPIO | 3.3 V | SW2 输入 | 板上 2.2 kΩ 上拉至 3.3 V；子卡安装时被占用 | [S1, PDF p.4]；[S2, PDF p.26]；[S3, PDF p.5] |
| Arduino J3 | 10 | AR_SCL | P15 | PL I/O，Arduino SCL 标号 | I²C SCL/GPIO | 3.3 V | SW1 输入 | 板上 2.2 kΩ 上拉至 3.3 V；子卡安装时被占用 | 同上 |

#### J4

| 接口 | 物理引脚 | GPIO/网络名 | 主控封装引脚 | 默认功能 | 复用/可实现功能 | 电压 | EES_363DP 占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| Arduino J4 | 1 | AR0 | T14 | PL I/O | GPIO/PWM/定时器/中断 | 3.3 V | 数码管位选 K4 | 子卡安装时被占用 | [S2, PDF p.26]；[S3, PDF p.4] |
| Arduino J4 | 2 | AR1 | U12 | PL I/O | 同上 | 3.3 V | 数码管位选 K3 | 同上 | 同上 |
| Arduino J4 | 3 | AR2 | U13 | PL I/O | 同上 | 3.3 V | 数码管位选 K2 | 同上 | 同上 |
| Arduino J4 | 4 | AR3 | V13 | PL I/O | 同上 | 3.3 V | 数码管位选 K1 | 同上 | 同上 |
| Arduino J4 | 5 | AR4 | V15 | PL I/O | 同上 | 3.3 V | 段选/LED4 | 同上 | [S2, PDF p.26]；[S3, PDF p.4、p.6] |
| Arduino J4 | 6 | AR5 | T15 | PL I/O | 同上 | 3.3 V | 段选/LED5 | 同上 | 同上 |
| Arduino J4 | 7 | AR6 | R16 | PL I/O | 同上 | 3.3 V | 段选/LED6 | 同上 | 同上 |
| Arduino J4 | 8 | AR7 | U17 | PL I/O | 同上 | 3.3 V | 段选/LED7 | 同上 | 同上 |

#### J7 电源与复位

| 接口 | 物理引脚 | GPIO/网络名 | 主控封装引脚 | 默认功能 | 复用功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| Arduino J7 | 1 | VU_CK / VIN | - | 外部电源/电池入口 | - | 待确认 | 板卡供电 | J9 必须置 REG；允许电池电压范围未单列 | [S1, PDF p.4]；[S2, PDF p.7、p.27] |
| Arduino J7 | 2 | GND | - | 地 | - | 0 V | - | - | [S2, PDF p.27] |
| Arduino J7 | 3 | GND | - | 地 | - | 0 V | - | - | [S2, PDF p.27] |
| Arduino J7 | 4 | 5V5 | - | 5 V 电源 | - | 约 5 V | 板卡电源轨 | 仅供电；不得接入 3.3 V GPIO | [S1, PDF p.4]；[S2, PDF p.27] |
| Arduino J7 | 5 | 3V3 | - | 3.3 V 电源 | - | 3.3 V | 板卡电源轨 | 总电流上限待确认 | [S2, PDF p.27] |
| Arduino J7 | 6 | AR_RST | PS_MIO12_500 | Shield 复位 | 复位信号 | 3.3 V 逻辑 | 与系统复位逻辑关联 | 不是普通 PL GPIO；SRST 会触发 Shield 复位 | [S1, PDF p.4、p.10]；[S2, PDF p.22、p.27] |
| Arduino J7 | 7 | 3V3 | - | 3.3 V 电源 | - | 3.3 V | 板卡电源轨 | 总电流上限待确认 | [S2, PDF p.27] |
| Arduino J7 | 8 | N/C | - | 未连接 | - | - | - | 禁止假定可用 | [S2, PDF p.27] |

#### Arduino SPI_CNN1（2x3）

| 接口 | 物理引脚 | GPIO/网络名 | 主控封装引脚 | 默认功能 | 复用功能 | 电压 | EES_363DP 占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| SPI_CNN1 | 1 | AR_MISO | W15 | PL I/O，SPI MISO 标号 | SPI/GPIO | 3.3 V | 未在子卡表中列出 | 200 Ω 串联；方向取决于控制器设计 | [S1, PDF p.4]；[S2, PDF p.27] |
| SPI_CNN1 | 2 | 3V3 | - | 电源 | - | 3.3 V | 子卡供电 | 总电流上限待确认 | [S2, PDF p.27] |
| SPI_CNN1 | 3 | AR_SCK | H15 | PL I/O，SPI 时钟标号 | SPI SCK/GPIO | 3.3 V | 段选/LED3 | 子卡安装时被占用 | [S2, PDF p.27]；[S3, PDF p.4、p.6] |
| SPI_CNN1 | 4 | AR_MOSI | T12 | PL I/O，SPI MOSI 标号 | SPI MOSI/GPIO | 3.3 V | 段选/LED2 | 子卡安装时被占用 | 同上 |
| SPI_CNN1 | 5 | AR_SS | F16 | PL I/O，SPI 片选标号 | SPI CS/GPIO | 3.3 V | 段选/LED1 | 子卡安装时被占用 | 同上 |
| SPI_CNN1 | 6 | GND | - | 地 | - | 0 V | 子卡供电回路 | - | [S2, PDF p.27] |

### 4.4 Arduino 模拟接口 J1 与专用 XADC J5

| 接口 | 物理引脚 | GPIO/网络名 | 主控封装引脚 | 默认功能 | 复用功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| Arduino J1 | 1 | A5 | U10 | 0-3.3 V 单端模拟输入 | PL 数字 GPIO | 0-3.3 V | 模拟分压/滤波网络 | 禁止超过 3.3 V | [S1, PDF p.4、p.11]；[S2, PDF p.27-p.28] |
| Arduino J1 | 2 | A4 | T5 | 同上 | PL 数字 GPIO | 0-3.3 V | 同上 | 同上 | 同上 |
| Arduino J1 | 3 | A3 | V11 | 同上 | PL 数字 GPIO | 0-3.3 V | 同上 | 同上 | 同上 |
| Arduino J1 | 4 | A2 | W11 | 同上 | PL 数字 GPIO | 0-3.3 V | 同上 | 同上 | 同上 |
| Arduino J1 | 5 | A1 | Y12 | 同上 | PL 数字 GPIO | 0-3.3 V | 同上 | 同上 | 同上 |
| Arduino J1 | 6 | A0 | Y11 | 同上 | PL 数字 GPIO | 0-3.3 V | 同上 | 同上 | 同上 |
| XADC J5 | 1 | XADC_V_P / VP_0 | K9（专用模拟脚） | XADC 差分正端 | 仅模拟 | 0-1 V | 140 Ω、1 nF 网络 | 不可作数字 GPIO | [S1, PDF p.4、p.9]；[S2, PDF p.28-p.29] |
| XADC J5 | 2 | XADC_V_N / VN_0 | L10（专用模拟脚） | XADC 差分负端 | 仅模拟 | 0-1 V 差分 | 140 Ω、1 nF 网络 | 不可作数字 GPIO | 同上 |
| XADC J5 | 3 | XADCGND | - | 模拟地 | - | 0 V | 模拟地网络 | 与数字大电流回路分开布线 | [S1, PDF p.4、p.9]；[S2, PDF p.29] |
| XADC J5 | 4 | XADCVREF | - | XADC 参考端 | - | 待确认 | 板载参考网络 | 不应当作普通传感器电源 | [S1, PDF p.4、p.9]；[S2, PDF p.29] |

### 4.5 Raspberry Pi 40 pin 排针（RPI_IDE）

`RPIO_xx` 名称来自原理图。I²C、SPI、UART 等名称沿用 Raspberry Pi 排针习惯，但这些脚连接 Zynq PL，必须在 FPGA 中实现相应控制器。表中主控封装引脚优先记录参考手册 Table 22；与原理图 Pmod A 共享表不一致的项同时列出并标记。

| 接口 | 物理引脚 | GPIO/网络名 | 主控封装引脚 | 默认功能 | 复用/可实现功能 | 电压 | 板载占用 | 使用限制 | 来源 |
|---|---:|---|---|---|---|---|---|---|---|
| RPI_IDE | 1 | 3V3 | - | 电源 | - | 3.3 V | 板卡电源轨 | 总电流上限待确认 | [S1, PDF p.4]；[S2, PDF p.30] |
| RPI_IDE | 2 | 5V | - | 电源 | - | 5 V | 板卡电源轨 | 仅供电 | 同上 |
| RPI_IDE | 3 | RPIO_02_R | W18 | GPIO，SDA 标号 | I²C SDA/GPIO/中断 | 3.3 V | 与 PMODA-9 共享；2.2 kΩ 上拉 | 禁止外部上拉至 5 V | [S1, PDF p.4]；[S2, PDF p.30] |
| RPI_IDE | 4 | 5V | - | 电源 | - | 5 V | 板卡电源轨 | 仅供电 | 同上 |
| RPI_IDE | 5 | RPIO_03_R | W19 | GPIO，SCL 标号 | I²C SCL/GPIO/中断 | 3.3 V | 与 PMODA-10 共享；2.2 kΩ 上拉 | 禁止外部上拉至 5 V | [S1, PDF p.4]；[S2, PDF p.30] |
| RPI_IDE | 6 | GND | - | 地 | - | 0 V | - | - | 同上 |
| RPI_IDE | 7 | RPIO_04_R | 手册：V6；原理图共享表：Y18 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 原理图称与 PMODA-1 共享 | 引脚冲突，使用前核对实际 XDC/连通性 | [S1, PDF p.4]；[S2, PDF p.24、p.30] |
| RPI_IDE | 8 | RPIO_14_R | Y18 | GPIO，TXD 标号 | UART TX/GPIO | 3.3 V | 无明确占用 | 不是 PS UART0 | [S1, PDF p.4]；[S2, PDF p.30] |
| RPI_IDE | 9 | GND | - | 地 | - | 0 V | - | - | 同上 |
| RPI_IDE | 10 | RPIO_15_R | Y19 | GPIO，RXD 标号 | UART RX/GPIO | 3.3 V | 无明确占用 | 不是 PS UART0 | 同上 |
| RPI_IDE | 11 | RPIO_17_R | U7 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 12 | RPIO_18_R | C20 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 13 | RPIO_27_R | V7 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 14 | GND | - | 地 | - | 0 V | - | - | 同上 |
| RPI_IDE | 15 | RPIO_22_R | U8 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 16 | RPIO_23_R | W6 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 17 | 3V3 | - | 电源 | - | 3.3 V | 板卡电源轨 | 总电流上限待确认 | 同上 |
| RPI_IDE | 18 | RPIO_24_R | U18 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 与原理图共享关系存在间接不一致，需核对 XDC | 同上 |
| RPI_IDE | 19 | RPIO_10_R | V8 | GPIO，MOSI 标号 | SPI MOSI/GPIO | 3.3 V | 无明确占用 | 需要 PL SPI 控制器 | 同上 |
| RPI_IDE | 20 | GND | - | 地 | - | 0 V | - | - | 同上 |
| RPI_IDE | 21 | RPIO_09_R | V10 | GPIO，MISO 标号 | SPI MISO/GPIO | 3.3 V | 无明确占用 | 需要 PL SPI 控制器 | 同上 |
| RPI_IDE | 22 | RPIO_25_R | U19 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 与原理图共享关系存在间接不一致，需核对 XDC | 同上 |
| RPI_IDE | 23 | RPIO_11_R | W10 | GPIO，SCLK 标号 | SPI SCLK/GPIO | 3.3 V | 无明确占用 | 需要 PL SPI 控制器 | 同上 |
| RPI_IDE | 24 | RPIO_08_R | F19 | GPIO，CE0 标号 | SPI CS0/GPIO | 3.3 V | 无明确占用 | 需要 PL SPI 控制器 | 同上 |
| RPI_IDE | 25 | GND | - | 地 | - | 0 V | - | - | 同上 |
| RPI_IDE | 26 | RPIO_07_R | 手册：F20；原理图共享表：U19 | GPIO，CE1 标号 | SPI CS1/GPIO | 3.3 V | 原理图称与 PMODA-8 共享 | 引脚冲突，使用前核对实际 XDC/连通性 | [S1, PDF p.4]；[S2, PDF p.24、p.30] |
| RPI_IDE | 27 | RPIO_SD_R | Y16 | GPIO，ID_SD 标号 | I²C SDA/GPIO | 3.3 V | 与 PMODA-3 共享；2.2 kΩ 上拉 | 与 PMODA 共用；禁止 5 V 上拉 | [S1, PDF p.4]；[S2, PDF p.24、p.30] |
| RPI_IDE | 28 | RPIO_SC_R | Y17 | GPIO，ID_SC 标号 | I²C SCL/GPIO | 3.3 V | 与 PMODA-4 共享；2.2 kΩ 上拉 | 与 PMODA 共用；禁止 5 V 上拉 | 同上 |
| RPI_IDE | 29 | RPIO_05_R | 手册：Y6；原理图共享表：Y19 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 原理图称与 PMODA-2 共享 | 引脚冲突，使用前核对实际 XDC/连通性 | [S1, PDF p.4]；[S2, PDF p.24、p.30] |
| RPI_IDE | 30 | GND | - | 地 | - | 0 V | - | - | [S2, PDF p.30] |
| RPI_IDE | 31 | RPIO_06_R | 手册：Y7；原理图共享表：U18 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 原理图称与 PMODA-7 共享 | 引脚冲突，使用前核对实际 XDC/连通性 | [S1, PDF p.4]；[S2, PDF p.24、p.30] |
| RPI_IDE | 32 | RPIO_12_R | B20 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | [S1, PDF p.4]；[S2, PDF p.30] |
| RPI_IDE | 33 | RPIO_13_R | W8 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 34 | GND | - | 地 | - | 0 V | - | - | 同上 |
| RPI_IDE | 35 | RPIO_19_R | Y8 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 36 | RPIO_16_R | B19 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 37 | RPIO_26_R | W9 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 38 | RPIO_20_R | A20 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |
| RPI_IDE | 39 | GND | - | 地 | - | 0 V | - | - | 同上 |
| RPI_IDE | 40 | RPIO_21_R | Y9 | GPIO | GPIO/PWM/定时器/中断 | 3.3 V | 无明确占用 | 需要 bitstream/XDC | 同上 |

### 4.6 板载按钮、开关和 LED

这些信号已经连接板载器件，不是外接排针。它们适合调试驱动和状态，但不应在 XDC 中再次绑定给外部传感器。

| 板载器件 | 信号 | 主控封装引脚 | 默认方向/有效电平 | 电气/占用说明 | 来源 |
|---|---|---|---|---|---|
| 按钮 BTN0 | BTN0 | D19 | 输入；按下为高 | 板载占用 | [S1, PDF p.3、p.11]；[S2, PDF p.20] |
| 按钮 BTN1 | BTN1 | D20 | 输入；按下为高 | 板载占用 | 同上 |
| 按钮 BTN2 | BTN2 | L20 | 输入；按下为高 | 板载占用 | 同上 |
| 按钮 BTN3 | BTN3 | L19 | 输入；按下为高 | 板载占用 | 同上 |
| 拨码 SW0 | SW0 | M20 | 输入；拨到 up/闭合为高 | 板载占用 | [S1, PDF p.3、p.11]；[S2, PDF p.20] |
| 拨码 SW1 | SW1 | M19 | 输入；拨到 up/闭合为高 | 板载占用 | 同上 |
| LED0 | LED0 | R14 | 输出高点亮 | 330 Ω 串联 | [S1, PDF p.3、p.11]；[S2, PDF p.21] |
| LED1 | LED1 | P14 | 输出高点亮 | 330 Ω 串联 | 同上 |
| LED2 | LED2 | N16 | 输出高点亮 | 330 Ω 串联 | 同上 |
| LED3 | LED3 | M14 | 输出高点亮 | 330 Ω 串联 | 同上 |
| RGB LD4 Blue | LED4_B | L15 | 经晶体管反相 | 建议 PWM；高强度 LED | [S1, PDF p.3、p.11]；[S2, PDF p.21] |
| RGB LD4 Red | LED4_R | N15 | 经晶体管反相 | 建议 PWM | 同上 |
| RGB LD4 Green | LED4_G | G17 | 经晶体管反相 | 建议 PWM | 同上 |
| RGB LD5 Blue | LED5_B | G14 | 经晶体管反相 | 建议 PWM | 同上 |
| RGB LD5 Red | LED5_R | M15 | 经晶体管反相 | 建议 PWM | 同上 |
| RGB LD5 Green | LED5_G | L14 | 经晶体管反相 | 建议 PWM | 同上 |

### 4.7 专用板载接口与被占用的主控引脚

下列接口是可识别的板载连接，但已专用于存储、网络、USB、串口、视频或音频。除非重新设计整套硬件/bitstream，否则不要用于普通小型传感器。

| 接口/器件 | 主控信号或引脚 | 默认用途 | 板载占用与限制 | 来源 |
|---|---|---|---|---|
| Quad-SPI Flash | MIO1 CS；MIO2 DQ0；MIO3 DQ1；MIO4 DQ2；MIO5 DQ3；MIO6 SCLK；MIO8 SCLK_FB；MIO7 VCFG0 | 启动与非易失存储 | 已连接 S25FL128S；MIO8 仅接 20 kΩ 上拉 | [S2, PDF p.11] |
| USB Host PHY | MIO11 Over Current；MIO28 DATA4；29 DIR；30 STP；31 NXT；32 DATA0；33 DATA1；34 DATA2；35 DATA3；36 CLK；37 DATA5；38 DATA6；39 DATA7；46 RESETN | USB 2.0 ULPI Host | 已连接 TUSB1210；不支持 OTG/device 模式 | [S2, PDF p.12] |
| MicroSD SD1 | MIO40 CCLK；41 CMD；42 D0；43 D1；44 D2；45 D3；47 CD | SDIO0 | 已被卡槽占用；最高 50 MHz；不支持 SPI 模式 | [S2, PDF p.14] |
| Ethernet PHY | MIO9 Reset；10 Interrupt；16 TXCK；17-20 TXD0-3；21 TXCTL；22 RXCK；23-26 RXD0-3；27 RXCTL；52 MDC；53 MDIO | 10/100/1000 Ethernet | 已连接 RTL8211E-VL；MDIO 地址 00001 | [S2, PDF p.15-p.16] |
| USB-UART/JTAG J8 | MIO14 UART Input；MIO15 UART Output；JTAG 专用链路 | PC 串口和下载调试 | 已连接 FT2232HL；不要当作空闲传感器 UART | [S1, PDF p.8、p.10]；[S2, PDF p.17] |
| HDMI TX/source | D2 P/N J18/H18；D1 K19/J19；D0 K17/K18；CLK L16/L17；CEC G15；DDC B9/B13；HPD R19 | HDMI 源/输出 | 直接连接 PL；HDMI 含 5 V，不能当普通 3.3 V 排针 | [S1, PDF p.6、p.11]；[S2, PDF p.18] |
| HDMI RX/sink | D2 P/N N20/P20；D1 T20/U20；D0 V20/W20；CLK N18/P19；CEC H17；DDC U14/U15；HPD T19 | HDMI 接收/输入 | 直接连接 PL；DDC/HPD 经过板上电路 | [S1, PDF p.6、p.11]；[S2, PDF p.18] |
| HDMI PS I²C | MIO50 HDMI_TX_SCL；MIO51 HDMI_TX_SDA | HDMI DDC 控制 | 已用于 HDMI 相关接口 | [S2, PDF p.19] |
| Audio Codec | AU_MCLK、BCLK、WCLK、DIN、DOUT、SDA、SCL 等 | ADAU1761 音频 | 已连接板载 Codec 和音频插孔 | [S1, PDF p.7、p.11]；[S2, PDF p.13] |
| PL 参考时钟 | H16 SYSCLK | 125 MHz PL 参考时钟 | 来自 Ethernet PHY；`PHYRSTB` 拉低时 CLK125 停止 | [S1, PDF p.5、p.11]；[S2, PDF p.9] |
| PS 时钟 | PS_CLK | 50 MHz | 板载固定时钟 | [S2, PDF p.9] |

## 5. 按功能分类的引脚

### 5.1 电源与地

| 类别 | 可用位置 | 说明 |
|---|---|---|
| 3.3 V | PMODA 6/12；PMODB 6/12；Arduino J7 5/7；SPI_CNN1-2；RPI_IDE 1/17 | 适合 3.3 V 小型传感器；Pmod 电源/地针脚组最高 1 A，其他排针总能力待确认 |
| 5 V | Arduino J7-4；RPI_IDE 2/4 | 可给 5 V 传感器供电，但信号仍须转换为 3.3 V |
| VIN | Arduino J7-1 | 板卡外部供电入口，不是传感器电源输出；J9 置 REG |
| GND | PMODA 5/11；PMODB 5/11；Arduino J3-7、J7-2/3、SPI-6；RPI_IDE 6/9/14/20/25/30/34/39；XADC J5-3 | 传感器必须与板卡共地；模拟测量优先靠近 XADC/Arduino 模拟口接地 |

### 5.2 普通 GPIO

- 首选：PMODB 1-4、7-10。它不与 RPi 共享，也不被 EES_363DP 占用。
- 次选：未装 EES_363DP 时的 Arduino J3/J4 或 SPI_CNN1 信号。
- 可选：Raspberry Pi 排针的 28 个 PL 信号；但 Pmod A 共享项及文档冲突项必须先用万用表/XDC 验证。
- Pmod A 的 8 个信号全部与 Raspberry Pi 排针共享，不应把两侧分别连接不同外设。
- 所有外部 GPIO 均应按 3.3 V 使用；5 V 输入必须转换电平。

### 5.3 ADC

| 用途 | 推荐引脚 | 范围 | 说明 |
|---|---|---:|---|
| 普通模拟传感器 | Arduino J1 A0-A5 | 0-3.3 V 单端 | 板上分压到 XADC 范围；最适合模拟光照模块 |
| 精密差分输入 | XADC J5 `V_P/V_N` | 0-1 V 差分 | 不能数字复用；超出 1 V 会有风险 |

当前已有的 GY-302/BH1750 是 I²C 数字光照传感器，不接 A0-A5，也不使用 XADC。

### 5.4 PWM 与定时器

- 文档没有列出固定的外部 PWM 或定时器通道。
- 任一空闲 PL GPIO 都可由 FPGA 逻辑产生 PWM，或作为计数器/输入捕获信号。
- 板载 RGB LED 建议使用 PWM；其驱动经晶体管反相。
- 超声波测距需要至少一个输出 GPIO（TRIG）和一个带计时/捕获逻辑的输入 GPIO（ECHO）。

### 5.5 I²C

| 接口 | SDA | SCL | 上拉/占用 | 结论 |
|---|---|---|---|---|
| Arduino | J3-9 `AR_SDA` / P16 | J3-10 `AR_SCL` / P15 | 各有 2.2 kΩ 上拉至 3.3 V；EES_363DP 将其用作 SW2/SW1 | 未装子卡时推荐 |
| Raspberry Pi 主 I²C 标号 | pin 3 `RPIO_02` / W18 | pin 5 `RPIO_03` / W19 | 2.2 kΩ 上拉至 3.3 V；与 PMODA-9/10 共享 | 可用，但注意共享 |
| Raspberry Pi ID I²C 标号 | pin 27 `RPIO_SD` / Y16 | pin 28 `RPIO_SC` / Y17 | 2.2 kΩ 上拉至 3.3 V；与 PMODA-3/4 共享 | 可用，但通常保留作 ID 总线 |
| PMODB 自定义 | 任意两根空闲 I/O | 任意两根空闲 I/O | 文档未给出上拉；需外加 3.3 V 上拉或使用带正确上拉的模块 | 装 EES_363DP 时最推荐 |

I²C 是开漏总线。若传感器模块自带上拉电阻，必须确认上拉目标是 3.3 V，而不是模块的 5 V 电源。不同模块的并联上拉会降低等效电阻，应核对总线电流和波形。

当前方案固定使用 PMODB-1 / W14 作为 SCL、PMODB-2 / Y14 作为 SDA，GY-302 与 OLED 并联到这两根线。

### 5.6 SPI

当前三个模块均不使用 SPI；本节只保留作后续扩展参考。

| 接口 | SCLK | MOSI | MISO | CS | 说明 |
|---|---|---|---|---|---|
| Arduino SPI_CNN1 | pin 3 H15 | pin 4 T12 | pin 1 W15 | pin 5 F16 | 明确标注的 SPI 排针；EES_363DP 占用 SCLK/MOSI/CS |
| Raspberry Pi SPI 标号 | pin 23 W10 | pin 19 V8 | pin 21 V10 | pin 24 F19 / pin 26 见冲突 | 需要 PL SPI 控制器；pin 26 映射需复核 |
| PMODB 自定义 SPI | 任意输出 | 任意输出 | 任意输入 | 任意输出 | 装数字子卡时的推荐方案；由 XDC 和 RTL 自行定义 |

### 5.7 UART

当前三个模块均不使用外部 UART；板载 USB-UART 继续用于软件日志和 MSH 命令行。

| 接口 | TX | RX | 状态 |
|---|---|---|---|
| 板载 USB-UART | PS MIO15 | PS MIO14 | 已连接 FT2232HL，用于电脑串口，不建议改作传感器接口 |
| Raspberry Pi UART 标号 | pin 8 / Y18 | pin 10 / Y19 | 连接 PL，不是 PS UART0；需要实现 PL UART |
| PMODB 自定义 UART | 任意空闲输出 | 任意空闲输入 | 推荐用于 3.3 V UART 传感器；传感器 TX 接 FPGA RX，传感器 RX 接 FPGA TX |

### 5.8 中断、复位与启动相关引脚

| 信号/资源 | 引脚或位置 | 作用 | 注意事项 | 来源 |
|---|---|---|---|---|
| 传感器中断 | 任一空闲 PL GPIO | 可在 PL 中做边沿检测并接入处理器中断 | 文档没有固定外部 IRQ 脚；必须在 bitstream 中实现 | [S2, PDF p.23-p.30] |
| AR_RST | Arduino J7-6，PS MIO12 | Shield 复位 | 系统 SRST 会触发该复位；不当普通 GPIO | [S1, PDF p.4、p.10]；[S2, PDF p.22、p.27] |
| SRST | 板载系统复位按钮 | 重置 PS、清空 PS 内存并清除 PL | 不重新采样启动模式绑带 | [S2, PDF p.22] |
| PROG | 板载 PROG 按钮/PROG_B | 仅重置 PL 配置 | PL 保持未配置，直到重新下载 bitstream | [S1, PDF p.9]；[S2, PDF p.22] |
| JP1 | Boot Mode 跳线 | 选择 MicroSD、QSPI 或 JTAG 启动 | 上电前按板上丝印设置 | [S2, PDF p.8] |
| J9 | Power Source 跳线 | USB 或 REG 电源选择 | 外部 DC/VIN 时置 REG | [S1, PDF p.15]；[S2, PDF p.7] |

## 6. 板载器件占用情况

### 6.1 PYNQ-Z2 固定占用

| 资源 | 占用信号 | 能否直接改作传感器接口 |
|---|---|---|
| DDR3 | PS DDR 专用 Bank 502 | 否 |
| QSPI Flash | MIO1-8 中相关信号 | 不建议，会影响启动/存储 |
| MicroSD | MIO40-47 中相关信号 | 不建议，会影响启动/文件系统 |
| Ethernet | MIO9、10、16-27、52、53；PL H16 时钟 | 不建议；还可能影响 125 MHz PL 时钟 |
| USB Host | MIO11、28-39、46 | 否 |
| USB-UART/JTAG | MIO14/15 和 JTAG 链 | 不建议，会失去串口/下载调试 |
| HDMI | 多根 PL 差分/控制引脚及 MIO50/51 | 只有完全不使用 HDMI 且重做 bitstream 时才考虑 |
| Audio Codec | 多根 PL 音频/I²C 信号 | 只有不使用音频时才考虑 |
| 按钮、开关、LED | 见 4.6 | 已有板载负载，可用于调试但不是自由外接脚 |
| Pmod A/RPi 共享 | 8 根网络 | 只能作为同一物理信号使用，不可分配给两个独立外设 |

### 6.2 安装 EES_363DP 后的 Arduino 占用

| 子卡功能 | 子卡标号 | PYNQ-Z2 封装引脚 | 对应 Arduino 信号 | 来源 |
|---|---|---|---|---|
| 数码管段选/LED3 | K(0/1/2/3)_11 / LED3 | H15 | AR_SCK | [S3, PDF p.4、p.6] |
| 数码管段选/LED1 | K(0/1/2/3)_7 / LED1 | F16 | AR_SS | 同上 |
| 数码管段选/LED5 | K(0/1/2/3)_4 / LED5 | T15 | AR5 | 同上 |
| 数码管段选/LED8 | K(0/1/2/3)_2 / LED8 | V17 | AR8 | 同上 |
| 数码管段选/LED7 | K(0/1/2/3)_1 / LED7 | U17 | AR7 | 同上 |
| 数码管段选/LED2 | K(0/1/2/3)_10 / LED2 | T12 | AR_MOSI | 同上 |
| 数码管段选/LED4 | K(0/1/2/3)_5 / LED4 | V15 | AR4 | 同上 |
| 数码管段选/LED6 | K(0/1/2/3)_3 / LED6 | R16 | AR6 | 同上 |
| 数码管位选 K1 | K1_12 | V13 | AR3 | [S3, PDF p.4] |
| 数码管位选 K2 | K2_9 | U13 | AR2 | 同上 |
| 数码管位选 K3 | K3_8 | U12 | AR1 | 同上 |
| 数码管位选 K4 | K4_6 | T14 | AR0 | 同上 |
| 拨码 SW1 | SW1_2 | P15 | AR_SCL | [S3, PDF p.5] |
| 拨码 SW2 | SW2_2 | P16 | AR_SDA | 同上 |
| 拨码 SW3 | SW3_2 | N17 | AR13 | 同上 |
| 拨码 SW4 | SW4_2 | P18 | AR12 | 同上 |
| 拨码 SW5 | SW5_2 | R17 | AR11 | 同上 |
| 拨码 SW6 | SW6_2 | T16 | AR10 | 同上 |

EES_363DP 手册说明数码管段选与 LED 共用相同约束。因此这 8 根线不能同时作为彼此独立的功能使用。子卡由 PYNQ-Z2 供电，SW7 拨到 ON 后 D3 电源指示灯常亮。[S3, PDF p.3-p.6]

## 7. 现有模块的统一接线与实现方案

### 7.1 推荐的 PMODB 信号分配

这套分配可同时连接 HC-SR04、GY-302 和四针 I²C OLED，并避开 EES_363DP 已占用的 Arduino 引脚。

| 用途 | PMODB 物理脚 | 板级网络 | Zynq 封装脚 | FPGA 方向 | 外部电路 |
|---|---:|---|---|---|---|
| 传感器 I²C SCL | 1 | JB1_P | W14 | `inout`，开漏 | 上拉到 3.3 V；GY-302 和 OLED 并联 |
| 传感器 I²C SDA | 2 | JB1_N | Y14 | `inout`，开漏 | 上拉到 3.3 V；GY-302 和 OLED 并联 |
| HC-SR04 TRIG 驱动 | 3 | JB2_P | T11 | 输出 | 推荐经 5 V 隔离/电平转换后接 TRIG |
| HC-SR04 ECHO 采样 | 4 | JB2_N | T10 | 输入 | 必须先把 5 V ECHO 降到不高于 3.4 V |
| 传感器地 | 5 或 11 | GND | - | - | 三个模块与 PYNQ-Z2 共地 |
| I²C 模块电源 | 6 或 12 | 3V3 | - | - | 给 GY-302 与 OLED 提供 3.3 V |
| HC-SR04 电源 | Arduino J7-4 或 RPI_IDE 2/4 | 5V | - | - | HC-SR04 图片标明要求 DC 4.5-5.5 V [I1] |

注意：PMODB 的电源脚只有 3.3 V，没有 5 V。HC-SR04 的 VCC 必须从板上的 5 V 电源脚单独引出，但 GND 必须和 PMODB GND 相连。

### 7.2 GY-302（BH1750）接线

GY-302 与 OLED 共用同一对 SCL/SDA。

| GY-302 丝印 | 连接位置 | 说明 |
|---|---|---|
| VCC | PMODB-6 或 PMODB-12（3.3 V） | 当前图片未明确模块允许供电范围；先按 3.3 V 方案使用，不先接 5 V |
| GND | PMODB-5 或 PMODB-11 | 与 FPGA、OLED、HC-SR04 共地 |
| SCL | PMODB-1 / W14 | I²C 时钟，开漏 |
| SDA | PMODB-2 / Y14 | I²C 数据，开漏 |
| ADDR | GND | 推荐配置为 7-bit 地址 `0x23` [I4] |

若把 ADDR 接 3.3 V，图片给出的 7-bit 地址为二进制 `1011100`，即 `0x5C`；ADDR 接低电平时为二进制 `0100011`，即 `0x23`。[I4]

图片没有展示五针接口从上到下的实际顺序，因此实物接线必须按模块 PCB 丝印 `VCC/GND/SCL/SDA/ADDR` 逐针确认，不能根据照片方向猜测。[I3]、[I4]

### 7.3 四针 I²C OLED 接线

照片中 OLED 排针旁的丝印顺序可读为 `GND VCC SCL SDA`；实际插线时仍应正对实物丝印逐针确认。[I2]

| OLED 丝印 | 连接位置 | 说明 |
|---|---|---|
| GND | PMODB-5 或 PMODB-11 | 共地 |
| VCC | PMODB-6 或 PMODB-12（3.3 V） | 图片未给供电范围；先用 3.3 V，禁止未经确认直接接 5 V |
| SCL | PMODB-1 / W14 | 与 GY-302 SCL 并联 |
| SDA | PMODB-2 / Y14 | 与 GY-302 SDA 并联 |

仅凭图片不能确认 OLED 的控制器、分辨率和 I²C 地址。常见地址不能当作本模块的确定值。软件应先扫描 `0x08-0x77` 范围，再结合模块背面芯片标识或商家资料确认驱动；在控制器未确认前，不应直接固定为 SSD1306 或固定地址。

### 7.4 两个 I²C 模块共用总线时的电气检查

1. PYNQ-Z2 文档没有说明 PMODB 自带 I²C 上拉，因此 SCL/SDA 必须存在到 **3.3 V** 的上拉。
2. GY-302 照片中可见标记为 `472` 的电阻，可能是 4.7 kΩ 电阻，但单凭图片不能确认它们是否就是 SDA/SCL 上拉。[I3]
3. OLED 模块是否带上拉、上拉阻值和上拉目标电压均待确认。
4. 断电时用万用表测量 SCL 到 3.3 V、SDA 到 3.3 V 的等效电阻。若模块均没有上拉，可各加一只约 4.7 kΩ 电阻到 3.3 V；若已经存在合适上拉，不要重复盲目并联。
5. 给模块上电后，在 FPGA 端释放 SCL/SDA，测量总线空闲高电平，必须接近 3.3 V，且不得超过 3.4 V 推荐上限。
6. 若任一模块只能用 5 V，或模块将 SDA/SCL 上拉到 5 V，必须加入双向 I²C 电平转换器，不能把总线直接接入 W14/Y14。
7. 初次调试从 100 kHz I²C 时钟开始；两块模块都稳定后再考虑提高速率。

GY-302 地址建议设为 `0x23`。OLED 地址待扫描确认；只要扫描结果与 `0x23` 不同，两者即可在同一总线上工作。

### 7.5 HC-SR04 安全接线

#### 7.5.1 已确认的模块参数

| 项目 | 图片信息 |
|---|---|
| 引脚 | `VCC/TRIG/ECHO/GND` |
| 工作电压 | DC 4.5-5.5 V |
| 工作电流 | 约 5 mA |
| 静态电流 | 小于 2 mA |
| TRIG | 先拉低，再给至少 10 us 的脉冲 |
| 内部上拉 | ECHO 和 TRIG 各有 10 kΩ 上拉 |
| 探测范围 | 2-450 cm |
| 超声频率 | 40 kHz |

以上均来自所提供商品图片 [I1]。

#### 7.5.2 为什么 TRIG 和 ECHO 都不能直接接 FPGA

- ECHO 可能输出接近模块 5 V 电源的高电平，而 PYNQ-Z2 PL I/O 按 3.3 V 使用，推荐最大输入为 3.4 V。
- 图片明确说明 TRIG 内部也有 10 kΩ 上拉。模块以 5 V 供电时，若 FPGA 尚未配置、引脚处于高阻，TRIG 可能通过内部上拉把 FPGA 引脚抬向 5 V。
- 因此不能只处理 ECHO 而把 TRIG 直接接 T11；两个方向都应进行隔离或电平处理。

#### 7.5.3 推荐前端电路

| HC-SR04 端 | 推荐电路 | PYNQ-Z2 端 |
|---|---|---|
| VCC | 直接接板上 5 V | Arduino J7-4 或 RPI_IDE 2/4 |
| GND | 直接共地 | PMODB-5/11 |
| TRIG | 使用独立单向 3.3 V 到 5 V 电平转换；或使用 N 沟道 MOSFET 开漏下拉方案 | PMODB-3 / T11 |
| ECHO | 使用独立单向 5 V 到 3.3 V 电平转换；最低限度使用电阻分压 | 分压后接 PMODB-4 / T10 |

可行的 ECHO 分压示例：HC-SR04 ECHO 经过 5.1 kΩ 接到 FPGA 采样节点，采样节点再经过 10 kΩ 接 GND。理想 5 V 输入被降为约 3.31 V。该方案仍需在示波器上确认高电平、边沿和模块实际输出结构；专用单向电平转换器更稳妥。

一种可让 TRIG 上电默认保持低电平的隔离方式如下：

1. 使用小信号 N 沟道 MOSFET，源极接 GND，漏极接 HC-SR04 TRIG。
2. MOSFET 栅极接 PMODB-3 / T11，并用约 10 kΩ 电阻上拉到 PYNQ-Z2 的 3.3 V。
3. 栅极为高时 MOSFET 导通，TRIG 被拉低；这是上电和空闲状态。
4. 触发时 FPGA 把栅极拉低至少 10 us，MOSFET 关断，TRIG 由模块内部 10 kΩ 上拉形成高脉冲；随后 FPGA 再把栅极拉高结束脉冲。
5. 这种控制逻辑是反相的，RTL 中应使用类似 `hcsr04_trig_gate` 的名称，避免把电平含义写反。

如果改用集成电平转换器，应选择适合推挽数字信号、两个方向可分别配置的器件；不能在未核对方向和输出结构时直接套用普通双向 I²C 电平转换板。

#### 7.5.4 测距时序

1. 空闲时让模块 TRIG 保持低电平。
2. 产生至少 10 us 的 TRIG 高脉冲；采用上述 MOSFET 方案时，对应栅极输出至少 10 us 的低脉冲。
3. 等待 ECHO 上升沿；ECHO 是异步输入，进入计数逻辑前至少经过两级触发器同步。
4. 测量 ECHO 高电平时间。按室温声速近似，距离可用 `distance_cm = echo_time_us / 58.3` 估算。
5. 450 cm 对应往返时间约 26.2 ms，可设置约 30 ms 的 ECHO 超时；重复测量可先采用不短于 60 ms 的保守周期。
6. 软件需区分正常值、等待上升沿超时、等待下降沿超时和超范围四种状态。

### 7.6 FPGA 顶层端口与 XDC 示例

建议的顶层信号：

```systemverilog
inout  wire sensor_i2c_scl_io;
inout  wire sensor_i2c_sda_io;
output wire hcsr04_trig_gate_o; // 采用 MOSFET 方案时：1=TRIG 低，0=TRIG 高
input  wire hcsr04_echo_i;      // 已经经过 5 V 到 3.3 V 电平处理
```

对应 PMODB 分配：

```tcl
set_property -dict {PACKAGE_PIN W14 IOSTANDARD LVCMOS33} [get_ports sensor_i2c_scl_io]
set_property -dict {PACKAGE_PIN Y14 IOSTANDARD LVCMOS33} [get_ports sensor_i2c_sda_io]
set_property -dict {PACKAGE_PIN T11 IOSTANDARD LVCMOS33} [get_ports hcsr04_trig_gate_o]
set_property -dict {PACKAGE_PIN T10 IOSTANDARD LVCMOS33} [get_ports hcsr04_echo_i]
```

I²C 输出必须使用开漏逻辑：控制器只能主动输出 `0`，输出 `1` 时应把引脚切换为高阻态，由外部 3.3 V 上拉产生高电平。不要把 SCL/SDA 写成普通推挽高输出。

### 7.7 FPGA 与软件功能划分

| 层次 | 建议功能 |
|---|---|
| I²C 物理/控制器 RTL | 开漏 SCL/SDA、START/STOP、ACK/NACK、读写字节、超时和总线恢复 |
| HC-SR04 RTL | TRIG 状态机、ECHO 双触发同步、脉宽计数、30 ms 超时、结果锁存和状态位 |
| 总线寄存器层 | I²C 命令/数据寄存器；HC-SR04 启动、忙、完成、超时、脉宽计数寄存器 |
| GY-302 驱动 | 使用扫描确认后的 `0x23` 或 `0x5C` 地址；按 BH1750 数据手册初始化和换算照度 |
| OLED 驱动 | 先确认控制器、分辨率和地址，再选择对应初始化序列、字库和显存布局 |
| 应用层 | 周期读取距离和照度，在 OLED 上显示；同时可通过 UART/MSH 输出诊断信息 |

软件第一次上板建议按以下顺序进行：

1. 不连接 HC-SR04，只接 GY-302 和 OLED，运行 I²C 扫描。
2. 确认发现 GY-302 的 `0x23`，记录 OLED 的实际地址。
3. 单独读取 GY-302，确认照度随遮挡和照明变化。
4. 查明 OLED 控制器后单独点亮屏幕，再显示固定字符串。
5. 断电后安装 HC-SR04 电平处理电路；先用示波器确认 TRIG 和 ECHO 在 FPGA 端均不超过 3.4 V。
6. 单独验证 HC-SR04 脉宽计数，最后才把距离与照度同时显示到 OLED。

如果系统使用 MSH，可规划以下测试命令：

| 命令 | 作用 |
|---|---|
| `i2c_scan` | 扫描并打印 GY-302 与 OLED 的实际 7-bit 地址 |
| `lux_read` | 读取一次 GY-302 并打印原始值和照度 |
| `range_read` | 触发一次 HC-SR04 并打印 ECHO 脉宽、距离或超时状态 |
| `oled_test` | 清屏并显示固定测试图案/字符串 |
| `sensor_start` | 周期读取距离和照度并更新 OLED |
| `sensor_stop` | 停止周期任务并释放软件资源 |

### 7.8 上电前接线清单

- [ ] HC-SR04 VCC 接 5 V，不接 PMODB 3.3 V。
- [ ] OLED 和 GY-302 VCC 先接 PMODB 3.3 V，不先接 5 V。
- [ ] 三个模块与 PYNQ-Z2 共地。
- [ ] GY-302 ADDR 已明确接 GND（`0x23`）或 3.3 V（`0x5C`），没有悬空。
- [ ] OLED 与 GY-302 的 SCL/SDA 并联，且空闲电压实测约为 3.3 V。
- [ ] I²C 总线上只有合适的一组等效上拉，没有任何上拉接到 5 V。
- [ ] HC-SR04 TRIG 已隔离，模块内部 5 V 上拉不会送入 T11。
- [ ] HC-SR04 ECHO 已分压/转换，T10 端高电平不超过 3.4 V。
- [ ] XDC 使用 W14、Y14、T11、T10，且没有与工程其他顶层端口重复绑定。
- [ ] 所有模块在板卡断电时完成插拔；按图片提示，HC-SR04 应先接好再通电 [I1]。

## 8. 冲突与待确认事项

| 编号 | 冲突或不确定项 | 文档 A | 文档 B | 处理原则 |
|---:|---|---|---|---|
| 1 | Pmod B `JB3_N` 封装引脚 | 原理图 Bank 34 标为 W16 [S1, PDF p.11] | 参考手册表格印为 `W156` [S2, PDF p.24] | `W156` 不是合法的 CLG400 球位格式，疑似排版错误，但本文不擅自修正为唯一结论；使用 W16 前核对官方 XDC |
| 2 | RPI pin 7 `RPIO_04` | 原理图共享表：与 JA1_P 共用，JA1_P=Y18 [S1, PDF p.4] | 参考手册 Table 22：pin 7=V6 [S2, PDF p.30] | 冲突；实物连通性、官方 XDC 和当前 bitstream 三方核对后才能使用 |
| 3 | RPI pin 26 `RPIO_07` | 原理图共享表：与 JA3_N 共用，JA3_N=U19 [S1, PDF p.4] | 参考手册 Table 22：pin 26=F20 [S2, PDF p.30] | 同上 |
| 4 | RPI pin 29 `RPIO_05` | 原理图共享表：与 JA1_N 共用，JA1_N=Y19 [S1, PDF p.4] | 参考手册 Table 22：pin 29=Y6 [S2, PDF p.30] | 同上 |
| 5 | RPI pin 31 `RPIO_06` | 原理图共享表：与 JA3_P 共用，JA3_P=U18 [S1, PDF p.4] | 参考手册 Table 22：pin 31=Y7 [S2, PDF p.30] | 同上 |
| 6 | RPI pin 18/22 的 U18/U19 关系 | 参考手册 Table 22 把 U18/U19 分配给 RPI pin 18/22 [S2, PDF p.30] | 原理图共享表把 U18/U19 分配给与 RPI pin 31/26 相连的 JA3_P/JA3_N [S1, PDF p.4] | 属于同一组间接冲突；使用相关脚前必须核对 |
| 7 | 原理图版本与手册引用版本 | 上传原理图为 R10 [S1, PDF p.1] | 参考手册链接指向 R12 原理图 [S2, PDF p.5、p.31] | 板卡丝印版本待确认；不要混用不同版本 XDC |
| 8 | Arduino “另六组差分模拟输入”标号 | 手册称另有六组、标为 AR0-AR13 [S2, PDF p.25] | 连接器表没有逐组给出对应关系 [S2, PDF p.26-p.28] | 待确认；本文只把明确的 A0-A5 和 J5 `V_P/V_N` 作为可直接使用的 ADC 入口 |
| 9 | Arduino J7-1 电压 | 手册允许电池正极接 VIN [S2, PDF p.7] | 表格只标 `VU_CK POWER`，未给电压范围 [S2, PDF p.27] | 待确认；不得把它当作固定电压传感器电源 |
| 10 | 各 3.3 V/5 V 排针总供电能力 | 只明确 Pmod VCC/GND 可达 1 A [S2, PDF p.23] | Arduino/RPi 排针未给独立电流上限 | 待查电源树、稳压器余量和实际负载 |
| 11 | FPGA 单引脚驱动能力 | 本组三份文档未给出完整 DC 参数 | - | 必须查 XC7Z020 数据手册并结合所选 IOSTANDARD/DRIVE/SLEW |
| 12 | EES_363DP 与传感器共用 | 子卡把 Arduino I²C、部分 SPI 和 GPIO 用于开关/LED/数码管 [S3, PDF p.4-p.6] | PYNQ 手册将这些脚列为通用接口 [S2, PDF p.25-p.27] | 两者并非文档错误，而是安装状态不同；装子卡时按“已占用”处理 |
| 13 | OLED 控制器、分辨率和地址 | 图片只能确认 0.96 英寸、四针 I²C 蓝色 OLED [I2] | 未提供背面芯片标识或数据手册 | 必须先扫描地址并确认控制器，再选择驱动；禁止直接认定为 SSD1306 |
| 14 | OLED 供电范围和上拉 | 图片仅显示 `GND/VCC/SCL/SDA` [I2] | 未标注电压或电阻网络 | 初始按 3.3 V 供电；空闲总线电压必须实测，未确认前不接 5 V |
| 15 | GY-302 供电与上拉 | 图片可见 BH1750 标识和若干电阻 [I3] | 未提供模块原理图 | 先用 3.3 V；`472` 只能确认阻值标识，不能据此断定总线等效上拉 |
| 16 | HC-SR04 TRIG 内部 5 V 上拉 | 图片明确称 TRIG/ECHO 各有 10 kΩ 上拉 [I1] | FPGA 为 3.3 V I/O [S2, PDF p.25-p.26] | TRIG 与 ECHO 均做隔离/电平处理，不能只给 ECHO 分压 |

### 8.1 上板前的最低核对清单

1. 确认主板 PCB 丝印版本与将使用的 XDC 版本一致。
2. 对 RPi/Pmod A 冲突脚做断电通断测试；不要在未确认时下载同时驱动两个候选封装脚的 bitstream。
3. 用万用表确认 OLED/GY-302 的 SDA/SCL 空闲电压，并用示波器确认 HC-SR04 的 TRIG/ECHO 电平。
4. 确认主板和传感器共地，再连接信号线。
5. 先用限流电源或低风险测试程序验证方向，避免 FPGA 输出与外设输出相互对冲。
6. 检查当前 bitstream 的 XDC，确保同一封装引脚只绑定一个顶层端口。
7. EES_363DP 安装时，禁止把其已占用的 Arduino 引脚再次配置为相反方向的强驱动输出。

## 9. 信息来源索引

### [S1] PYNQ-Z2 原理图

- 文件：`PYNQ Z2_原理图 用于查询相关pin脚分布.pdf`
- 文档标识：PYNQ-Z2 R10，2018-01-09，共 15 页。
- 重点页：
  - PDF p.3：Pmod、按钮、LED 电路。
  - PDF p.4：Arduino、Raspberry Pi、Pmod A 共享表、模拟输入网络和 I²C 上拉。
  - PDF p.5：Ethernet、MicroSD。
  - PDF p.6：HDMI。
  - PDF p.7：USB、音频。
  - PDF p.8：FT2232HL、JTAG、USB-UART。
  - PDF p.9：FPGA 配置、QSPI、XADC 专用脚。
  - PDF p.10：PS DDR/MIO Banks。
  - PDF p.11：PL Bank 13/34/35 完整封装球位与板级网络。
  - PDF p.14-p.15：稳压与供电选择。

### [S2] PYNQ-Z2 Reference Manual

- 文件：`pynqz2_user_manual_v1_0-1525725.pdf`
- 版本：v1.0，2018-05-17，共 32 页。
- 重点页：
  - PDF p.4-p.5：板卡特性、主控、存储和扩展接口。
  - PDF p.7-p.9：供电、启动模式和时钟。
  - PDF p.11-p.19：QSPI、USB、MicroSD、Ethernet、UART、HDMI。
  - PDF p.20-p.22：按钮、开关、LED、复位。
  - PDF p.23-p.24：Pmod 完整引脚表和供电能力。
  - PDF p.25-p.29：Arduino 完整引脚表、电气限制和 XADC。
  - PDF p.30：Raspberry Pi 40 pin 排针与 PL 封装引脚。

### [S3] EES_363DP 数字子卡用户手册

- 文件：`数字子卡用户手册.pdf`
- 版本：2018.12 ver1.0，共 8 页。
- 重点页：
  - PDF p.3：子卡用途和供电开关。
  - PDF p.4：四位八段数码管引脚约束。
  - PDF p.5：6 个拨码开关引脚约束。
  - PDF p.6：8 个 LED 引脚约束及与段选复用关系。

### [I1] HC-SR04 商品参数图片

- 文件：`codex-clipboard-beaa9b50-c5de-4c01-b8c3-bc3d019fcfbe.jpg`
- 可确认信息：HC-SR04 型号、四针定义、4.5-5.5 V、TRIG 至少 10 us、TRIG/ECHO 内部 10 kΩ 上拉、量程和电流参数。

### [I2] 四针 I²C OLED 图片

- 文件：`codex-clipboard-5ed69dfe-b989-46e8-a1e9-54b5491c7535.jpg`
- 可确认信息：0.96 英寸、蓝色 OLED、四针 I²C、丝印 `GND/VCC/SCL/SDA`。
- 无法确认：控制器、分辨率、地址和供电范围。

### [I3] GY-302 模块实物图片

- 文件：`codex-clipboard-f16ee390-49f0-4a53-97ad-6d9b492484be.jpg`
- 可确认信息：GY-302、板上 BH1750 标识、五针接口、可见 `472` 等阻值标识。

### [I4] GY-302 引脚与地址图片

- 文件：`codex-clipboard-a890f180-d75a-407f-b04d-ad3944f02da9.jpg`
- 可确认信息：`VCC/GND/SCL/SDA/ADDR` 功能；ADDR 高时 `0x5C`，低时 `0x23`。

---

**当前推荐方案：**PMODB-1/2 作为共享 I²C，连接 GY-302 和 OLED；PMODB-3/4 连接经过电平处理的 HC-SR04 TRIG/ECHO；OLED 与 GY-302 使用 3.3 V，HC-SR04 使用 5 V，所有模块共地。保留板载 USB-UART 作为调试串口，避免使用存在版本冲突的 Pmod A/Raspberry Pi 共享脚。
