# 周期信号测量分析装置（2026 电赛 G 题）

基于 STM32F407ZGT6 主控的高精度周期信号采集与频谱分析装置，采用 Goertzel 算法结合最小二乘拟合实现时域、频域一体化测量，7 寸 TFT 实时显示波形与频谱。

## ✨ 功能特性

- ⚡ **高速同步采样**：三路 ADC 多模式同步采集，6 MSPS 采样率，DMA 零丢点传输
- 🎯 **高精度频谱分析**：Goertzel 单频点算法 + 最小二乘拟合，幅值误差 ≤ 1 mV
- 📊 **多参数解算**：峰峰值 Vpp、真有效值 Vrms、基频、各次谐波幅值与相位
- 🔍 **精细频率校正**：三段相关功率搜索，消除栅栏效应，基频误差 ≤ 100 Hz
- 🪟 **汉宁窗抑制**：时域加窗处理，有效抑制频谱泄露
- 📈 **全频段校准**：频率 - 增益查表 + 线性插值，补偿硬件幅频特性误差
- 🛡️ **强抗干扰能力**：八阶硬件低通 + 软件频段筛选，叠加 1 MHz 干扰仍正常工作
- 🖥️ **双界面显示**：1 周期 / 3 周期时域波形 + 离散频谱，按键一键切换
- ⏱️ **快速响应**：整套采集 - 分析 - 显示流程 ≤ 0.3 s，远优于题目 2 s 要求

## 🔧 硬件清单

| 模块 | 型号 / 说明 | 通信方式 |
|------|-------------|----------|
| 主控 | STM32F407ZGT6（Cortex-M4，FPU + DSP 指令集） | - |
| 显示屏 | 7 寸 TFT 液晶屏（800×480） | FSMC 并行总线 |
| ADC 采集 | 内置 12 位 ADC，三路多模式同步 | - |
| 运放 | MS8312 高速轨到轨运放 × 6 片 | - |
| 低通滤波器 | 8 阶 Sallen-Key 有源 RC，截止 550 kHz | - |
| 程控放大 | CD4051 模拟开关 + 同相放大，8 档增益 | GPIO |
| 偏置电路 | 1.65 V 精密直流偏置，电压跟随缓冲 | - |
| 输入防护 | TVS + 肖特基二极管，过压 ESD 保护 | - |
| 电源 | 单路 5 V 供电，低噪声 LDO 模拟独立供电 | - |
| 交互 | 物理按键（模式切换 / 量程） | GPIO |

## 🏗️ 软件架构

基于 **模块化分层架构 + 轮询调度**，采集、分析、显示三层解耦。

### 软件分层

| 层级 | 模块 | 功能 |
|------|------|------|
| 底层驱动层 | adc_test.c | 三路 ADC 同步采集、定时器触发、DMA 双缓冲 |
| 底层驱动层 | tft_fsmc.c | FSMC 总线驱动 7 寸 TFT，绘图接口 |
| 底层驱动层 | key.c | 按键扫描、消抖、模式切换 |
| 信号处理层 | signal_analysis.c | Goertzel 频谱分析、最小二乘拟合、谐波解算 |
| 信号处理层 | frequency_calibration.c | 频率 - 增益查表、线性插值补偿 |
| 信号处理层 | fir_filter.c | 129 阶凯泽窗 FIR 低通滤波、降采样 |
| 人机交互层 | display_ui.c | 时域波形、频谱界面绘制、参数显示 |
| 主逻辑调度层 | main.c | 轮询调度、采集 - 分析 - 显示流程控制 |

### 信号处理流程
原始采样 → 去直流 → FIR 降采样滤波 → 汉宁窗加窗 → Goertzel 频谱分析

↓

显示输出 ← 波形重构 ← 最小二乘拟合 ← 谱峰检索 ← 频率精细校正


### 核心算法

| 算法 | 说明 |
|------|------|
| Goertzel 算法 | 单频点频谱计算，只计算有效频段，效率高于 FFT |
| 最小二乘拟合 | 多谐波分量联合求解，直接得到幅值与相位 |
| 汉宁窗 | 抑制频谱泄露，提升邻频分量区分度 |
| 三段相关功率搜索 | 粗搜→细搜→精搜，校正栅栏效应，提高频率精度 |
| FIR 低通滤波 | 129 阶凯泽窗，降采样前抗混叠 |
| 频率 - 幅值校准 | 500 Hz 步长查表 + 线性插值，补偿硬件幅频误差 |

### 工作状态
SIGNAL_STATE_IDLE // 空闲等待

SIGNAL_STATE_SAMPLING // ADC 采集中

SIGNAL_STATE_PROCESSING // 信号分析中（Goertzel + 拟合）

SIGNAL_STATE_DISPLAY // 界面刷新

SIGNAL_STATE_OVERLOAD // 过载告警


## 🚀 使用说明

### 1. 编译环境

- Keil uVision5
- STM32CubeMX（外设初始化配置）
- ST-Link / J-Link 下载器
- CMSIS-DSP 库（可选，用于优化运算）

### 2. 关键配置

```c
// 采样率
#define SIGNAL_ANALYSIS_SAMPLE_RATE_HZ    6000000.0f

// 有效频段
#define SIGNAL_MIN_FREQUENCY_HZ           10000.0f
#define SIGNAL_MAX_FREQUENCY_HZ           500000.0f

// 频谱分辨率
#define SIGNAL_BIN_WIDTH_HZ               500.0f
#define SIGNAL_SPECTRUM_COUNT             981U

// FIR 滤波器
#define SIGNAL_FIR_TAP_COUNT              129U
#define SIGNAL_DECIMATION_RATIO           8U

// 校准表
#define FREQUENCY_CAL_START_HZ            10000.0f
#define FREQUENCY_CAL_STEP_HZ             500.0f
```

### 3. 校准表更新
在 frequency_calibration.c 中替换增益数组：
```c
static const float g_amplitude_cal[FREQUENCY_CAL_COUNT] = {
    0.99419f,  // 10 kHz
    0.99440f,  // 10.5 kHz
    // ... 共 981 个频点，500 Hz 步长
};
```
