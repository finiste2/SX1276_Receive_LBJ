# SX1276_Receive_LBJ

一个基于 TTGO LoRa 32 v1.6.1（ESP32 + SX1276 868MHz）的 LBJ 消息接收器

SX1276_Receive_LBJ 项目修改自 [RadioLib](https://github.com/jgromes/RadioLib) 的 Pager_Receive.ino，以及
[LilyGo-LoRa-Series](https://github.com/Xinyuan-LilyGO/LilyGo-LoRa-Series) 的模板，
基于 TTGO LoRa 32 v1.6.1 开发板构建。

本项目旨在为通常用于接收中国铁路列车接近报警（Chinese Railway Train Proximity Alarm，简称 LBJ）的昂贵的可编程寻呼机提供一种替代方案。LBJ
消息通常由车载 LBJ 系统通过 CIR 发送，在 821.2375MHz 频率上采用 POCSAG 协议传输。

## 注意

这是一个用于学习嵌入式设备编程及 Sub-GHz 信号接收的实验性项目，因此代码质量较差，可能存在未知问题。请自行承担使用风险。

## 硬件

基于 [TTGO LoRa 32 v1.6.1 开发板](https://www.lilygo.cc/products/lora3)，利用 SX1276 的 FSK 调制解调器接收 LBJ
消息。该开发板原理图可在[这里](https://github.com/Xinyuan-LilyGO/LilyGo-LoRa-Series/blob/master/schematic/T3_V1.6.1.pdf)
查看。

### 附加功能

***注意：以下功能为可选功能，需要额外连接相应硬件。***

通过将指定外设连接至对应引脚，可启用以下附加功能：

#### 1. DS3231 RTC

与板载 SSD1306 OLED 共用 IIC 总线。

```c++
#define I2C_SDA                     21
#define I2C_SCL                     22
```

可在断电时保持时间。如果未使用，则每次重启后都会从 NTP 服务器获取时间。

若已连接，请在编译时取消注释 [utilities.h](src/utilities.h) 中的以下代码：

```c++
#define HAS_RTC // 已焊接外部 DS3231 模块。
```

#### 2. 模拟按键（AD Buttons，开发中）

由于 GPIO 数量不足，采用四键模拟按键方案。

```c++
#define BUTTON_PIN                  34
```

可为 OLED 上的交互功能（包括日志查看及设备设置，开发中）提供输入。

若已连接，请编译 ***button_equipped*** 分支，而不是 master 分支。

更多引脚定义信息请参见 [utilities.h](src/utilities.h)。

## 功能特性

- 在 OLED 上显示格式化后的 LBJ 消息。
- 将接收到的消息以纯文本和 CSV 格式保存至 TF 卡。
- 提供 Telnet 服务器，并向客户端发送格式化后的消息。
- 集成来自 MMDVM_HS_Hat 的 [BCH3121.cpp](https://github.com/phl0/POCSAG_HS/blob/master/BCH3121.cpp) 的 BCH 纠错算法。

## 已知问题

- 当 TF 卡中存在大量文件时，启动速度非常慢。
- **不支持** TF 卡热插拔。设备上电后拔出 TF 卡，将导致下一次接收消息时崩溃。
- WiFi 连接不稳定。
- Telnet 服务不支持多客户端连接。
- 无蜂鸣器报警功能。
- 部分解码或损坏的消息仍会显示在屏幕上。
- 文档及许可证仍为初步版本，后续需要整理和完善。

## 详细说明

### 1. 关于 LBJ 扩展消息（Extend Message）

该类消息未出现在 `TB/T 3504-2018` 标准中。

本人在使用 SDR 监听 821.2375MHz 时接收到过此类消息。一些列车会在 POCSAG 地址 `1234002` 上传输这些消息。

目前尚不清楚这类消息的正式名称及结构，其内容主要依据推测进行解析。当前已识别出的典型 LBJ 扩展消息字段如下：

| 半字节（4 bit） | 编码方式   | 含义                                        |
|------------|--------|-------------------------------------------|
| 0-3        | ASCII  | 类型                                        |
| 4-11       | 十进制    | 8 位机车注册号                                  |
| 12-13      | 未知     | 机车端位，31 表示 A 端，32 表示 B 端，30 表示无 A/B 端或未登记 |
| 14-29      | GB2312 | 区段（Route）                                 |
| 30-38      | 十进制    | 经度（XXX°XX.XXXX′ E）                        |
| 39-46      | 十进制    | 纬度（XX°XX.XXXX′ N）                         |
| 47-49      | 未知     | 未知字段，通常为 000                              |

整条消息共 50 个半字节（200 bit），占用 10 个 POCSAG Frame 进行发送。

### 2. SX1276 配置

- 频率 = 821.2375MHz + 6 ppm（默认）
- 模式 = FSK，RxDirect（DIO2）
- 增益 = 001 + LnaBoostHf（关闭 AGC）
- 接收带宽（RxBW）= 12.5 kHz

由于开发板所使用的 SX1276 模块没有配备 TCXO，因此实现了一套自动频率校正机制。

该机制利用 SX1276 的频率误差指示器（FEI），测量接收到的载波和前导码的频率误差，在检测到载波或前导码后尝试锁定信号，以补偿晶振或发射端造成的频率偏移。

可通过串口命令 `afc off` 禁用该机制。

### 3. Telnet / 串口命令

#### 串口（Serial）

- `ping`：串口状态测试，返回 `pong`。
- `time`：返回系统时间。
- `sd end`：卸载 TF 卡。
- `sd begin`：挂载 TF 卡。
- `mem`：返回剩余内存（调用 `esp_get_free_heap_size()`）。
- `rst`：返回复位原因。
- `ppm read`：返回当前 ppm。
- `afc off`：关闭自动频率校正。
- `afc on`：开启自动频率校正。

#### Telnet

- `ping`：Telnet 状态测试，返回 `pong`。
- `read`：读取 TF 卡日志中的 1000 字节数据。
- `log read [int]`：读取 TF 卡日志中的 `[int]` 字节数据。
- `log status`：返回 TF 卡日志功能是否启用。
- `afc off`：关闭自动频率校正。
- `afc on`：开启自动频率校正。
- `ppm [float]`：将 ppm 设置为 `[float]`。
- `bat`：返回电池电压。
- `rssi`：返回 SX1276 当前 RSSI。
- `gain`：返回 SX1276 当前增益。
- `time`：返回系统时间。
- `bye`：断开 Telnet 连接。

### 4. 接收失败

有时接收到的消息可能发生损坏、部分解码失败或被错误纠错，因此显示内容可能并不可靠。

如果出现 `<NUL>`、`NA`、`********` 或其中部分字符，则表示对应部分消息已经损坏。

## 使用/参考的库与代码

[//]: # (todo: 添加链接)

- [RadioLib](https://github.com/jgromes/RadioLib)（已修改）
- [U8G2](https://github.com/olikraus/u8g2)
- [ESP32-Arduino](https://github.com/espressif/arduino-esp32)
- [PlatformIO](https://platformio.org/)
- [ESP32AnalogRead](https://github.com/madhephaestus/ESP32AnalogRead.git)（用于电池电压检测）
- 来自 [POCSAG_HS](https://github.com/phl0/POCSAG_HS) 的 BCH3121.cpp/.h
- [LilyGo-LoRa-Series](https://github.com/Xinyuan-LilyGO/LilyGo-LoRa-Series)
- [ESPTelnet](https://github.com/LennartHennigs/ESPTelnet)
- [RTClib](https://github.com/adafruit/RTClib.git)
- [Multimon-ng](https://github.com/EliasOenal/multimon-ng)
- 项目灵感来源于 [GoRail_Pager](https://github.com/killeder/GoRail_Pager)。
- 以及其他暂时想不起来的项目，在此一并致歉。

衷心感谢所有作者和贡献者。