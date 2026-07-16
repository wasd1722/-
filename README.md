# HAL 库学习日志

STM32F103 HAL 库学习记录，从点灯到 FreeRTOS 迁移。

## 环境
- 芯片：STM32F103C8T6
- 工具：STM32CubeMX + Keil MDK + ST-Link
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

## 项目演进

| 天数 | 项目 | 说明 |
|------|------|------|
| D1 | [RGB 灯控制](projects/D1_RGB_Blink) | PA9(B)/PA10(G)/PA11(R) GPIO 输出，共阴极 |
| D2 | [TIM2 定时器中断 RGB](projects/D1_RGB_Blink) | 1秒切换颜色，中断标志位，SWD 踩坑记录 |

## 笔记
- [D1: RGB 灯控制](docs/D1_RGB_Blink.md)
- [D2: TIM2 定时器中断 RGB](docs/D2_TIM2_RGB.md)

## 踩坑记录
- [SWD 连接失败排查](docs/D2_TIM2_RGB.md#踩坑记录swd-引脚复用导致芯片锁死)

## 环境
- 芯片：STM32F103C8T6
- 工具：STM32CubeMX + Keil MDK + ST-Link
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz
