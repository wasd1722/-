# D3: 串口蓝牙控制 RGB 灯

## 目标
通过蓝牙串口发送命令，控制 RGB 灯颜色。理解 UART 中断接收、回调机制、波特率配置。

## 硬件
- 芯片：STM32F103C8T6
- RGB 灯：PA9(B)、PA10(G)、PA11(R)，共阴极，高电平点亮
- 蓝牙模块：HC-05，波特率 38400
- 接线：HC-05_TXD→PA3(USART2_RX)，HC-05_RXD→PA2(USART2_TX)，VCC→3.3V，GND→GND

## 环境
- STM32CubeMX + Keil MDK + ST-Link
- 手机蓝牙串口 APP（如"蓝牙串口"）
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

---

## CubeMX 配置

### 新增 USART2
| 参数 | 值 | 说明 |
|------|-----|------|
| Mode | Asynchronous | 异步串口 |
| Baud Rate | 38400 | HC-05 默认波特率，踩坑见下文 |
| Word Length | 8 Bits | 数据位 8 |
| Stop Bits | 1 | 停止位 1 |
| Parity | None | 无校验 |
| NVIC | USART2 global interrupt ✓ | 使能中断接收 |

### 保留 TIM2（D2 配置）
| 参数 | 值 |
|------|-----|
| Clock Source | Internal Clock |
| Prescaler | 7199 |
| Counter Period | 9999 |
| NVIC | TIM2 global interrupt ✓ |

### 保留 GPIO（D1 配置）
| 引脚 | 功能 | 初始状态 |
|------|------|---------|
| PA9 | GPIO_Output | Low |
| PA10 | GPIO_Output | Low |
| PA11 | GPIO_Output | Low |

---

## 关键代码

### 全局变量（USER CODE BEGIN PV）
```c
__IO uint8_t flag_tim2 = 0;   // TIM2 中断标志
__IO uint8_t rx_cmd = 0;       // 串口接收到的命令
uint8_t rx_buffer[1];            // 串口接收缓冲区
```

### RGB 控制函数（USER CODE BEGIN 0）
```c
void RGB_SetColor(uint8_t r, uint8_t g, uint8_t b)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_11, r ? GPIO_PIN_SET : GPIO_PIN_RESET);  // R
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_10, g ? GPIO_PIN_SET : GPIO_PIN_RESET);  // G
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9,  b ? GPIO_PIN_SET : GPIO_PIN_RESET);  // B
}
```

### TIM2 中断回调（USER CODE BEGIN 0）
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        flag_tim2 = 1;  // 只置标志位，快进快出
    }
}
```

### USART2 接收回调（USER CODE BEGIN 0）
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART2)
    {
        uint8_t ch = rx_buffer[0];

        // 过滤换行符，只保存有效命令
        if (ch != '\r' && ch != '\n')
        {
            rx_cmd = ch;
        }

        // 重新开启接收，准备下一次
        HAL_UART_Receive_IT(&huart2, rx_buffer, 1);
    }
}
```

### 主函数初始化（USER CODE BEGIN 2）
```c
// 启动 TIM2 定时器中断
HAL_TIM_Base_Start_IT(&htim2);

// 启动 USART2 中断接收
HAL_UART_Receive_IT(&huart2, rx_buffer, 1);
```

### 主循环（USER CODE BEGIN 3）
```c
while (1)
{
    // 串口命令处理
    if (rx_cmd != 0)
    {
        switch (rx_cmd)
        {
            case 'R':
            case 'r': RGB_SetColor(1,0,0); break;  // 红

            case 'G':
            case 'g': RGB_SetColor(0,1,0); break;  // 绿

            case 'B':
            case 'b': RGB_SetColor(0,0,1); break;  // 蓝

            case 'Y':
            case 'y': RGB_SetColor(1,1,0); break;  // 黄

            case 'W':
            case 'w': RGB_SetColor(1,1,1); break;  // 白

            case 'O':
            case 'o': RGB_SetColor(0,0,0); break;  // 灭

            default: break;
        }
        rx_cmd = 0;  // 处理完清零
    }

    // TIM2 定时器颜色切换（D2 功能保留）
    if (flag_tim2 == 1)
    {
        flag_tim2 = 0;
        // color 切换逻辑...
    }
}
```

---

## 踩坑记录：波特率不匹配

### 现象
- 手机蓝牙发 `G`，串口回显 `G` ✅
- 手机蓝牙发 `R`，串口回显 `^` ❌（ASCII 94，不是 82）

### 排查过程
1. 怀疑换行符问题 → 过滤 `\r\n`，无效
2. 怀疑大小写问题 → 加 `case 'r'`/`'R'`，小写没反应
3. 加 ASCII 回显调试 → 发现 `R`(82) 变成 `^`(94)，位错误
4. 查 HC-05 手册 → 默认波特率 **38400**，CubeMX 配的是 **9600**

### 根因
波特率不匹配导致采样点偏移，部分字符解错。`G`(71) 碰巧正确，`R`(82) 位模式容错失败。

### 解决
CubeMX 改 USART2 BaudRate 为 38400，重新生成代码。

### 教训
| 要点 | 说明 |
|------|------|
| 通信先确认波特率 | 双方必须一致，不能"大概" |
| 默认波特率因模块而异 | HC-05 默认 38400，HC-06 默认 9600 |
| 部分正确最误导 | `G` 对、`R` 错，容易怀疑代码而非波特率 |
| 回显调试最有效 | 把收到的字节原样发回，一眼看出问题 |

---

## 学到的

### UART 中断接收
- `HAL_UART_Receive_IT` 一次性配置，收到指定字节数后触发中断
- 中断回调里必须**重新开启接收**，否则只能收一次
- 多个串口共用 `HAL_UART_RxCpltCallback`，用 `huart->Instance` 区分

### 工程规范
- 中断共享变量用 `__IO`（volatile），防止编译器优化
- 回调函数只置标志位或保存数据，复杂逻辑放主循环
- 串口命令支持大小写，提升用户体验

### 调试技巧
- 回显（Echo）是排查通信问题的首选方法
- ASCII 码对照表是嵌入式工程师的基本功

---

## 验证现象

| 手机发送 | 灯颜色 | 说明 |
|---------|--------|------|
| R 或 r | 红 | PA11 高电平 |
| G 或 g | 绿 | PA10 高电平 |
| B 或 b | 蓝 | PA9 高电平 |
| Y 或 y | 黄 | PA11+PA10 高电平 |
| W 或 w | 白 | 三色全亮 |
| O 或 o | 灭 | 全低电平 |

---

## 下一步

D4: PWM 调光，实现 RGB 呼吸灯效果，理解定时器 PWM 模式。


# HAL 库学习日志

STM32F103 HAL 库学习记录，从点灯到 FreeRTOS 迁移。

## 环境
- 芯片：STM32F103C8T6
- 工具：STM32CubeMX + Keil MDK + ST-Link
- 调试工具：逻辑分析仪（计划购入）、ST-Link SWO
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

## 项目演进

| 天数 | 项目 | 关键技术 | 状态 |
|------|------|---------|------|
| D1 | [RGB 灯控制](projects/D1_RGB_Blink) | GPIO 输出、共阴极/共阳极 | ✅ 完成 |
| D2 | [TIM2 定时器中断 RGB](projects/D1_RGB_Blink) | 定时器中断、状态机、标志位设计 | ✅ 完成 |
| D3 | [串口蓝牙控制 RGB](projects/D1_RGB_Blink) | UART 中断接收、波特率配置、蓝牙通信 | ✅ 完成 |
| D4 | PWM 呼吸灯 | 定时器 PWM 模式、占空比 | 🔄 待开始 |
| D5 | ADC 采集 | 模数转换、DMA | ⏳ 计划 |
| D6 | FreeRTOS 集成 | 任务调度、信号量、互斥锁 | ⏳ 计划 |

## 笔记

| 天数 | 内容 | 链接 |
|------|------|------|
| D1 | GPIO 点灯、RGB 控制 | [D1_RGB_Blink.md](docs/D1_RGB_Blink.md) |
| D2 | TIM2 中断、SWD 踩坑 | [D2_TIM2_RGB.md](docs/D2_TIM2_RGB.md) |
| D3 | 串口蓝牙、波特率踩坑 | [D3_UART_RGB.md](docs/D3_UART_RGB.md) |

## 踩坑记录汇总

| 日期 | 问题 | 解决 | 文档 |
|------|------|------|------|
| 2026-07-16 | SWD 引脚复用导致芯片锁死 | Connect Under Reset 恢复 | [D2 文档](docs/D2_TIM2_RGB.md) |
| 2026-07-16 | 波特率不匹配导致串口乱码 | 改 38400 | [D3 文档](docs/D3_UART_RGB.md) |

## 技能树

- [x] GPIO 输入输出
- [x] 定时器中断
- [x] UART 中断通信
- [ ] PWM 输出
- [ ] ADC 采集
- [ ] DMA 传输
- [ ] FreeRTOS 任务调度
- [ ] 低功耗模式

