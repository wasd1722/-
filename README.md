# HAL 库学习日志

STM32F103 HAL 库学习记录，从点灯到 FreeRTOS 迁移。

## 环境
- 芯片：STM32F103C8T6
- 工具：STM32CubeMX + Keil MDK + ST-Link
- 调试工具：ST-Link SWO
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

## 项目演进

| 天数 | 项目 | 关键技术 | 状态 |
|------|------|---------|------|
| D1 | [RGB 灯控制](projects/D1_RGB_Blink) | GPIO 输出、共阴极/共阳极 | ✅ 完成 |
| D2 | [TIM2 定时器中断 RGB](projects/D1_RGB_Blink) | 定时器中断、状态机、标志位设计 | ✅ 完成 |
| D3 | [串口蓝牙控制 RGB](projects/D1_RGB_Blink) | UART 中断接收、波特率配置、蓝牙通信 | ✅ 完成 |
| D4 | [PWM 呼吸灯](projects/D1_RGB_Blink) | TIM1 PWM 模式、占空比、人眼非线性 | ✅ 完成 |
| D5 | [ADC 光敏自动调光](projects/D1_RGB_Blink) | ADC 采集、自动标定、PWM 联动调光 | ✅ 完成 |
| D6 | [FreeRTOS 多任务调度](projects/D1_RGB_Blink) | 任务调度、队列通信、优先级、栈管理 | ✅ 完成 |
| D7 | [信号量与互斥锁](projects/D1_RGB_Blink) | 互斥锁、二进制信号量、蓝牙命令控灯 | ✅ 完成 |
| D8 | [DMA + ADC 连续采集](projects/D1_RGB_Blink) | DMA 环形缓冲、信号量事件驱动 | ✅ 完成 |
| D9 | 超声波输入捕获 | TIM 输入捕获、脉宽测距 | ⏳ 计划 |

## 笔记

| 天数 | 内容 | 链接 |
|------|------|------|
| D1 | GPIO 点灯、RGB 控制 | [D1_RGB_Blink.md](docs/D1_RGB_Blink.md) |
| D2 | TIM2 中断、SWD 踩坑 | [D2_TIM2_RGB.md](docs/D2_TIM2_RGB.md) |
| D3 | 串口蓝牙、波特率踩坑 | [D3_UART_RGB.md](docs/D3_UART_RGB.md) |
| D4 | PWM 呼吸灯、MOE 踩坑 | [D4_PWM_Breathe.md](docs/D4_PWM_Breathe.md) |
| D5 | ADC 自动调光、自标定踩坑 | [D5_ADC_AutoLight.md](docs/D5_ADC_AutoLight.md) |
| D6 | FreeRTOS 多任务、队列通信踩坑 | [D6_FreeRTOS.md](docs/D6_FreeRTOS.md) |
| D7 | 信号量与互斥锁、蓝牙命令控灯 | [D7_Mutex_Semaphore.md](docs/D7_Mutex_Semaphore.md) |
| D8 | DMA+ADC 连续采集、信号量初始值踩坑 | [D8_DMA_ADC.md](docs/D8_DMA_ADC.md) |

## 踩坑记录汇总

| 来源 | 问题 | 解决 |
|------|------|------|
| D2 | SWD 引脚复用导致芯片锁死 | Connect Under Reset 恢复 |
| D3 | 波特率不匹配导致串口乱码 | 改 38400 |
| D4 | MOE 主输出使能疑问 | 查 HAL 源码发现自动使能 |
| D5 | 遮光灯变暗（映射方向反） | 去掉取反，暗→高ADC→高PWM |
| D5 | 亮度变化不明显 | 自动标定（动态追踪 adc_min/max） |
| D5 | ADC 时钟超标（/2→36MHz） | 改分频 /6→12MHz |
| D6 | 核心代码在 for(;;) 外→只跑一次 | 全部搬进循环内 |
| D6 | 队列 uint16_t 类型→栈越界 | 改 uint32_t |
| D6 | Low 任务被 Normal 永久饿死 | 改 Normal 同级轮转 |
| D8 | 信号量初始值 1→首帧读到零 | CubeMX Semaphore Initial State 改 Not Available |
| D8 | osSemaphoreReleaseFromISR 编译报错 | CMSIS v2 统一 API，直接 osSemaphoreRelease |

## 技能树

- [x] GPIO 输入输出
- [x] 定时器中断
- [x] UART 中断通信
- [x] PWM 输出
- [x] ADC 采集
- [x] DMA 传输
- [x] FreeRTOS 任务调度
- [x] 互斥锁与信号量
- [ ] 低功耗模式
