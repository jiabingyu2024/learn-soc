# SocRV 温度传感器、摄像头与显示屏扩展说明

本文档汇总 SocRV 系统连接温度传感器、摄像头和显示屏的整体方案，覆盖硬件从 CPU 到板外管脚的连接，以及软件从 MMIO 驱动到 RT-Thread MSH 命令的组织方式。

## 1. 当前状态

| 外设 | 状态 | 说明 |
| --- | --- | --- |
| SHT30 温度传感器 | 已实现并完成行为仿真 | 已加入 APB-I2C 控制器、SHT30 驱动及 `temp_start`/`temp_stop` 命令 |
| OV7670 摄像头 | 规划中，尚未修改工程 | 建议使用 QVGA、RGB565、8 位 DVP 接口 |
| ILI9341 显示屏 | 规划中，尚未修改工程 | 建议使用 320x240、RGB565、SPI 接口 |

摄像头和显示屏部分是后续实现规范，不代表相关 RTL 和软件文件当前已经存在。

## 2. 系统总体结构

```text
                                    配置与控制通路
RT-Thread / MSH
        |
        v
CPU MMIO -> HXI 总线 -> HXI-to-APB -> APB 互连
                                      |    |       |
                                      |    |       +-> Display 控制器（规划）
                                      |    +----------> Camera 控制器（规划）
                                      +---------------> I2C 控制器（已实现）
                                                           |
                                         +-----------------+-----------------+
                                         |                                   |
                                      SHT30                              OV7670 SCCB

                                    图像数据通路（规划）
OV7670 D0-D7/PCLK/HREF/VSYNC
        |
        v
摄像头采集模块 -> 双帧缓冲 BRAM -> ILI9341 SPI 显示控制器 -> 320x240 显示屏
```

CPU 只负责初始化、启动、停止和读取状态。摄像头像素不应由 CPU 逐字节搬运，而应由 FPGA 硬件直接写入帧缓冲并发送到显示屏。

## 3. 温度传感器扩展（已实现）

### 3.1 硬件链路

```text
CPU
 -> HXI 总线
 -> HXI-to-APB
 -> APB 互连
 -> APB-I2C 控制器（0x3000_3000）
 -> 开漏控制信号
 -> Xilinx IOBUF
 -> F22/G22
 -> 外部 SHT30
```

### 3.2 地址映射

I2C 外设占用一个 4 KiB APB 槽：

| 项目 | 地址 |
| --- | --- |
| I2C 基地址 | `0x3000_3000` |
| I2C 地址范围 | `0x3000_3000 - 0x3000_3FFF` |

寄存器如下：

| 地址 | 寄存器 | 用途 |
| --- | --- | --- |
| `0x3000_3000` | CONTROL | 控制器使能、IRQ 使能、软复位 |
| `0x3000_3004` | STATUS | Busy、Done、ACK Error、RX Valid、Bus Active |
| `0x3000_3008` | CLKDIV | I2C 时钟分频，默认 100 kHz |
| `0x3000_300C` | TXDATA | 发送字节 |
| `0x3000_3010` | RXDATA | 接收字节 |
| `0x3000_3014` | COMMAND | GO、START、STOP、READ、NACK |

### 3.3 I2C 控制器能力

- START、重复 START 和 STOP。
- 单字节发送与接收。
- 从设备 ACK 检测。
- 主设备 ACK/NACK 发送。
- 等待 SCL 释放，支持时钟拉伸。
- Busy、Done、ACK Error 和 RX Valid 状态。
- 软复位和可选中断。
- 默认 100 kHz，总线频率可通过分频寄存器调整。

SCL、SDA 采用开漏方式：FPGA 只主动拉低总线，逻辑高电平由 3.3 V 上拉电阻产生。

### 3.4 管脚与接线

| SHT30 信号 | 板卡位置 | FPGA 管脚 | 电气标准 |
| --- | --- | --- | --- |
| SCL | J10-9，DEBUG_39 | F22 | LVCMOS33 |
| SDA | J10-10，DEBUG_40 | G22 | LVCMOS33 |
| VCC | J10-20，+3.3 V | - | 必须使用 3.3 V |
| GND | J10-11 至 J10-19 中任意 GND | - | 共地 |

Bank 17/VADJ1 为 3.3 V。板卡的 DEBUG 信号线上有 100 ohm 串联电阻。若使用裸 SHT30，应在 SCL、SDA 上分别增加约 4.7 kohm 到 3.3 V 的上拉；常见模块通常已经带有上拉。

不要使用 5 V 给模块供电，否则模块上的 I2C 上拉可能使 FPGA 管脚承受 5 V。

### 3.5 软件层次

```text
temp_start / temp_stop
        |
        v
RT-Thread 温度监控线程 tempmon
        |
        v
SHT30 设备驱动
        |
        v
SocRV BSP I2C 驱动
        |
        v
MMIO 访问 0x3000_3000
```

底层 I2C 驱动负责：

- 初始化分频器。
- 发送和接收字节。
- 产生 START/STOP。
- 检查 NACK。
- 轮询完成状态。
- 超时后复位控制器。

SHT30 驱动负责：

- 先尝试地址 `0x44`，无应答时再尝试 `0x45`。
- 发送单次测量命令 `0x2400`。
- 等待测量完成并读取 6 字节温湿度数据。
- 使用多项式 `0x31` 检查 CRC-8。
- 将原始温度转换为千分之一摄氏度。

温度监控线程配置：

| 项目 | 配置 |
| --- | --- |
| 线程名 | `tempmon` |
| 优先级 | 22 |
| 栈空间 | 1536 字节 |
| 默认采样周期 | 1000 ms |
| 采样周期范围 | 10 - 60000 ms |

### 3.6 MSH 使用方式

默认每秒采样一次：

```text
msh > temp_start
```

指定 500 ms 采样周期：

```text
msh > temp_start 500
```

停止：

```text
msh > temp_stop
```

预期输出：

```text
Temperature monitor started, period=1000 ms
SHT30 detected at 0x44
Temperature: 25.000 C
Temperature: 25.031 C
Temperature monitor stop requested
Temperature monitor stopped
```

### 3.7 建议购买型号

推荐：

```text
DFRobot Fermion SHT30 数字温湿度传感器
SKU：SEN0330
```

搜索关键词：

```text
DFRobot SEN0330 SHT30 数字温湿度传感器 I2C
```

该模块具有 SDA、SCL、VCC、GND、INT 和 RST。当前应用只需要 SDA、SCL、VCC、GND；INT 暂不使用，RST 应保持高电平。不要购买 DFR0588 SHT30 模拟输出版本，因为现有硬件使用的是 I2C 数字接口。

## 4. 摄像头与显示屏扩展（规划方案）

### 4.1 建议的工作格式

| 项目 | 配置 |
| --- | --- |
| 摄像头 | OV7670，无 FIFO，18 针 DVP 模块 |
| 摄像头输出 | QVGA 320x240，RGB565 |
| 显示屏 | ILI9341，320x240，SPI |
| 像素大小 | 16 bit，即 2 字节 |
| 目标帧率 | 10 - 15 fps |

OV7670 的 QVGA RGB565 与 ILI9341 的 320x240 RGB565 可以直接对应，FPGA 不需要进行 JPEG 解码或复杂颜色空间转换。

### 4.2 为什么需要独立帧缓冲

当前 SoC 的 DATA 存储器只有 64 KiB：

```text
DATA：0x1000_0000 - 0x1000_FFFF
```

一张 RGB565 图像需要：

```text
320 x 240 x 2 = 153600 字节，约 150 KiB
```

因此图像不能存入当前操作系统数据 RAM。应在 FPGA 中增加两个独立 BRAM 帧缓冲：

```text
帧缓冲 A：153600 字节
帧缓冲 B：153600 字节
合计：307200 字节，约 300 KiB
```

摄像头写 A 时显示控制器读取 B；一帧结束后交换缓冲区，从而避免显示撕裂。

帧缓冲不需要映射到 APB。CPU 只访问控制和状态寄存器。

### 4.3 规划的地址映射

| 外设 | 建议地址范围 | 状态 |
| --- | --- | --- |
| Camera 控制器 | `0x3000_4000 - 0x3000_4FFF` | 待实现 |
| Display 控制器 | `0x3000_5000 - 0x3000_5FFF` | 待实现 |

两个地址都位于现有 APB 窗口 `0x3000_0000 - 0x3000_FFFF` 内，只需要扩展 APB 互连译码。

Camera 控制寄存器建议如下：

| 偏移 | 寄存器 | 用途 |
| --- | --- | --- |
| `0x00` | CONTROL | 启用、停止、软复位 |
| `0x04` | STATUS | 正在采集、帧完成、溢出 |
| `0x08` | FORMAT | 宽度、高度和 RGB565 格式 |
| `0x0C` | FRAME_COUNT | 已采集帧数 |
| `0x10` | BUFFER_STATUS | 当前写入和显示缓冲区 |

Display 控制寄存器建议如下：

| 偏移 | 寄存器 | 用途 |
| --- | --- | --- |
| `0x00` | CONTROL | 初始化、启动、停止 |
| `0x04` | STATUS | Busy、初始化完成、帧完成 |
| `0x08` | SPI_DIV | SPI 时钟分频 |
| `0x0C` | COMMAND | 发送 ILI9341 命令 |
| `0x10` | DATA | 发送 ILI9341 配置数据 |
| `0x14` | FRAME_COUNT | 已显示帧数 |

### 4.4 摄像头采集硬件

建议增加以下 RTL 模块：

```text
rtl/peripheral/camera/ov7670_capture.sv
rtl/peripheral/camera/apb_camera_ctrl.sv
rtl/peripheral/video/video_framebuffer.sv
rtl/peripheral/video/video_buffer_switch.sv
```

摄像头采集模块输入：

| 信号 | 方向 | 用途 |
| --- | --- | --- |
| `cam_data_i[7:0]` | 输入 | 8 位图像数据 |
| `cam_pclk_i` | 输入 | 像素时钟 |
| `cam_href_i` | 输入 | 当前行数据有效 |
| `cam_vsync_i` | 输入 | 帧同步 |
| `cam_xclk_o` | 输出 | 提供给 OV7670 的 24/25 MHz 时钟 |
| `cam_reset_no` | 输出 | 摄像头复位 |
| `cam_pwdn_o` | 输出 | 摄像头休眠控制 |

OV7670 每个 RGB565 像素分两次输出。采集模块需要将两个 8 位数据拼成一个 16 位像素，并写入：

```text
framebuffer_address = y * 320 + x
```

摄像头写端工作在 PCLK 时钟域，显示读取端工作在 50 MHz SoC 时钟域，因此帧缓冲应采用双口双时钟 BRAM，并对帧完成和缓冲区编号进行跨时钟域同步。

### 4.5 显示屏硬件

建议增加：

```text
rtl/peripheral/display/ili9341_spi.sv
rtl/peripheral/display/apb_display_ctrl.sv
```

显示信号：

| 信号 | 用途 |
| --- | --- |
| `lcd_sclk_o` | SPI 时钟 |
| `lcd_mosi_o` | SPI 数据输出 |
| `lcd_cs_no` | 显示屏片选 |
| `lcd_dc_o` | 命令/数据选择 |
| `lcd_reset_no` | 显示屏复位 |
| `lcd_bl_o` | 背光控制，可选 |

MISO、触摸和 MicroSD 在第一阶段都不需要使用。

在 50 MHz SoC 时钟下，可以生成约 25 MHz SPI 时钟。发送一张 320x240 RGB565 图像理论上约需 49 ms，适合作为 10 - 15 fps 的简单实时显示测试。

### 4.6 OV7670 建议管脚

OV7670 的 SCCB 配置接口与 SHT30 共享现有 I2C 总线：

| OV7670 | DEBUG 信号 | FPGA 管脚 | 板卡排针 |
| --- | --- | --- | --- |
| SIOC | DEBUG_39 | F22 | J10-9 |
| SIOD | DEBUG_40 | G22 | J10-10 |

OV7670 的 7 位地址通常为 `0x21`，与 SHT30 的 `0x44/0x45` 不冲突。

并行图像接口建议如下：

| OV7670 信号 | DEBUG 信号 | FPGA 管脚 | 板卡排针 |
| --- | --- | --- | --- |
| D0 | DEBUG_1 | G17 | J7-1 |
| D1 | DEBUG_2 | G18 | J7-2 |
| D2 | DEBUG_3 | F17 | J7-3 |
| D3 | DEBUG_4 | H20 | J7-4 |
| D4 | DEBUG_5 | F18 | J7-5 |
| D5 | DEBUG_6 | A16 | J7-6 |
| D6 | DEBUG_7 | E18 | J7-7 |
| D7 | DEBUG_8 | A17 | J7-8 |
| PCLK | DEBUG_9 | G19 | J7-9 |
| VSYNC | DEBUG_10 | G20 | J7-10 |
| HREF | DEBUG_11 | C19 | J8-1 |
| XCLK | DEBUG_12 | B19 | J8-2 |
| RESET | DEBUG_13 | B18 | J8-3 |
| PWDN | DEBUG_14 | A18 | J8-4 |

摄像头使用 3.3 V 供电并与 FPGA 共地。不同厂家的 OV7670 模块排针顺序可能不同，实际接线必须以模块丝印为准。

### 4.7 ILI9341 建议管脚

| 显示屏信号 | DEBUG 信号 | FPGA 管脚 | 板卡排针 |
| --- | --- | --- | --- |
| SCLK | DEBUG_21 | K20 | J9-1 |
| MOSI | DEBUG_22 | K19 | J9-2 |
| CS | DEBUG_23 | L18 | J9-3 |
| DC | DEBUG_24 | L17 | J9-4 |
| RESET | DEBUG_25 | K18 | J9-5 |
| BL，可选 | DEBUG_26 | J18 | J9-6 |
| MISO，可选 | DEBUG_27 | J17 | J9-7 |
| VCC | - | - | J9-20，+3.3 V |
| GND | - | - | J9-11 至 J9-19 中任意 GND |

第一阶段不连接 `TOUCH_CS`、触摸中断和 `SDCS`。DFR0665 背光具有默认状态，BL 可以暂时不接。

### 4.8 摄像头与显示屏软件层次

```text
video_start / video_stop / video_status
        |
        v
RT-Thread video_service
        |
        +-> OV7670 设备驱动 -> 共享 I2C/SCCB
        |
        +-> Camera BSP 驱动 -> 0x3000_4000
        |
        +-> ILI9341 初始化驱动
        |
        +-> Display BSP 驱动 -> 0x3000_5000
```

建议增加：

```text
software/camera/include/ov7670.h
software/camera/ov7670.c
software/bsp/include/drv_camera.h
software/bsp/drivers/drv_camera.c
software/bsp/include/drv_display.h
software/bsp/drivers/drv_display.c
software/video/include/video_service.h
software/video/video_service.c
software/video/command.c
```

OV7670 驱动负责：

- 通过地址 `0x21` 配置寄存器。
- 设置 QVGA 320x240。
- 设置 RGB565。
- 设置 10 - 15 fps。
- 配置自动曝光和自动白平衡。
- 打开或关闭内部测试彩条。

ILI9341 驱动负责：

- 硬复位和软件复位。
- 退出休眠。
- 设置 RGB565 像素格式。
- 设置横屏 320x240。
- 设置完整显示窗口。
- 启动硬件帧发送。

`video_start` 建议执行：

```text
锁定视频服务
 -> 锁定共享 I2C 总线
 -> 初始化 OV7670
 -> 释放 I2C 总线
 -> 初始化 ILI9341
 -> 清空双帧缓冲
 -> 启动 Camera
 -> 等待第一帧
 -> 启动 Display
```

`video_stop` 建议执行：

```text
停止 Camera
 -> 等待当前像素写入结束
 -> 停止 Display
 -> 可选关闭背光或清屏
 -> 释放视频服务资源
```

### 4.9 共享 I2C 的软件互斥

温度线程和视频初始化程序会共享同一个 I2C 控制器，因此在增加 OV7670 驱动时，应同步给 BSP I2C 驱动增加 RT-Thread 互斥量：

```c
int socrv_i2c_lock(void);
void socrv_i2c_unlock(void);
```

否则 `tempmon` 正在读取 SHT30 时，`video_start` 可能插入 OV7670 配置字节，导致两个事务互相破坏。

### 4.10 规划的 MSH 命令

启动：

```text
msh > video_start
```

查询：

```text
msh > video_status
```

停止：

```text
msh > video_stop
```

预期输出：

```text
OV7670 detected at 0x21
OV7670 configured: QVGA RGB565 10 fps
ILI9341 initialized: 320x240 RGB565
Video pipeline started

capture frames: 126
display frames: 124
camera overflow: 0

Video pipeline stopped
```

### 4.11 建议购买型号

摄像头建议购买：

```text
OV7670 无 FIFO 摄像头模块
18 针 DVP 并行接口
3.3 V
支持 RGB565 和 QVGA
```

搜索关键词：

```text
OV7670 无FIFO 摄像头模块 18针 RGB565 3.3V FPGA
```

购买前必须确认模块引出了：

```text
D0-D7、PCLK、VSYNC、HREF、XCLK、SIOC、SIOD、RESET、PWDN、3.3V、GND
```

不要购买 USB 摄像头、树莓派 MIPI CSI 摄像头、UART JPEG 摄像头或只有模拟视频输出的模块。

显示屏建议购买：

```text
DFRobot DFR0665
2.8 英寸 320x240 TFT
ILI9341
4 线 SPI
RGB565
```

搜索关键词：

```text
DFRobot DFR0665 ILI9341 2.8寸 320x240 SPI
```

虽然 DFR0665 带有触摸和 MicroSD，但第一阶段只使用显示功能。

## 5. 组合后的地址规划

| 地址范围 | 外设 | 当前状态 |
| --- | --- | --- |
| `0x3000_0000 - 0x3000_0FFF` | UART | 已存在 |
| `0x3000_1000 - 0x3000_1FFF` | GPIO | 已存在 |
| `0x3000_2000 - 0x3000_2FFF` | TEST_STATUS | 已存在 |
| `0x3000_3000 - 0x3000_3FFF` | I2C | 已实现 |
| `0x3000_4000 - 0x3000_4FFF` | Camera 控制器 | 规划中 |
| `0x3000_5000 - 0x3000_5FFF` | Display 控制器 | 规划中 |

## 6. 组合后的板外连接

| 外设 | 主要排针 | 供电 |
| --- | --- | --- |
| SHT30 | J10-9、J10-10 | J10-20 的 3.3 V |
| OV7670 数据口 | J7-1 至 J7-10、J8-1 至 J8-4 | J7/J8 的 3.3 V 与 GND |
| OV7670 SCCB | 与 SHT30 共享 J10-9、J10-10 | 3.3 V |
| ILI9341 | J9-1 至 J9-6 | J9-20 的 3.3 V |

所有模块必须共地。摄像头数据线、PCLK 和 XCLK 应尽量短，建议使用转接板或固定排线，避免十几根松散杜邦线产生时序和接触问题。

## 7. 推荐实施顺序

1. 保持现有 SHT30 功能不变。
2. 增加 I2C 总线互斥量，验证温度命令仍可运行。
3. 只实现 ILI9341 控制器，先显示 FPGA 内部生成的纯色和测试彩条。
4. 增加双帧缓冲 BRAM，并显示 BRAM 中的固定测试图。
5. 增加 OV7670 SCCB 初始化驱动。
6. 让 OV7670 输出内部彩条，验证摄像头采集到显示屏的完整链路。
7. 关闭 OV7670 彩条，切换到真实摄像画面。
8. 增加 `video_start`、`video_status` 和 `video_stop`。
9. 最后测试 `temp_start` 与 `video_start` 同时运行。

先测试彩条非常重要：如果彩条正确而真实画面异常，问题集中在 OV7670 寄存器和曝光配置；如果彩条也异常，问题通常位于 DVP 采样、字节顺序、帧缓冲切换或 SPI 显示链路。

## 8. 最终目标操作流程

```text
msh > temp_start 1000
Temperature monitor started, period=1000 ms
SHT30 detected at 0x44
Temperature: 25.000 C

msh > video_start
OV7670 detected at 0x21
ILI9341 initialized: 320x240 RGB565
Video pipeline started

msh > video_status
capture frames: 100
display frames: 99
camera overflow: 0

msh > video_stop
Video pipeline stopped

msh > temp_stop
Temperature monitor stopped
```

最终效果是温度传感器和摄像头共享 I2C 配置总线，温度由 RT-Thread 后台线程输出到串口 MSH，摄像头像素由 FPGA 硬件直接送入双帧缓冲并显示在 ILI9341 屏幕上。
