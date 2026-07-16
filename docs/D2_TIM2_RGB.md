# D2: TIM2 定时器中断控制 RGB 灯

## 目标
通过 TIM2 定时器中断，每 1 秒自动切换 RGB 灯颜色，理解中断机制与工程化编程规范。

## 硬件
- 芯片：STM32F103C8T6
- RGB 灯：PA9(蓝 B)、PA10(绿 G)、PA11(红 R)
- 共阴极，高电平点亮

## CubeMX 配置

### TIM2 定时器
| 参数 | 值 | 说明 |
|------|-----|------|
| Clock Source | Internal Clock | 内部时钟 72MHz |
| Prescaler (PSC) | 7199 | 72MHz / 7200 = 10kHz |
| Counter Period (ARR) | 9999 | 10kHz / 10000 = 1Hz = 1秒 |
| Auto-reload preload | Enable | 自动重载 |
| NVIC | TIM2 global interrupt ✓ | 使能中断 |

### GPIO（D1 已有，确认共阴极）
| 引脚 | 功能 | 初始状态 |
|------|------|---------|
| PA9 | GPIO_Output | Low |
| PA10 | GPIO_Output | Low |
| PA11 | GPIO_Output | Low |

## 关键代码

### 中断标志位（PV 区域）
```c
/* USER CODE BEGIN PV */
__IO uint8_t flag_tim2 = 0;  // 中断标志位，volatile 防止编译器优化
/* USER CODE END PV */
```

### 中断回调（USER CODE BEGIN 0）
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        flag_tim2 = 1;  // 只置标志位，快进快出
    }
}
```

### RGB 控制函数
```c
void RGB_SetColor(uint8_t r, uint8_t g, uint8_t b)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_11, r ? GPIO_PIN_SET : GPIO_PIN_RESET);  // R
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_10, g ? GPIO_PIN_SET : GPIO_PIN_RESET);  // G
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9,  b ? GPIO_PIN_SET : GPIO_PIN_RESET);  // B
}
```

### 主循环（USER CODE BEGIN 3）
```c
while (1)
{
    if (flag_tim2 == 1)
    {
        flag_tim2 = 0;

        static uint8_t color = 0;
        color++;
        if (color >= 4) color = 1;

        switch (color)
        {
            case 1: RGB_SetColor(1,0,0); break;  // 红
            case 2: RGB_SetColor(0,1,0); break;  // 绿
            case 3: RGB_SetColor(0,0,1); break;  // 蓝
            default: break;
        }
    }
}
```

## 踩坑记录：SWD 引脚复用导致芯片锁死

### 事故经过
调试过程中，**不小心将 PA13(SWDIO) 和 PA14(SWCLK) 配置为普通 GPIO**，导致芯片上电后 SWD 调试接口被禁用，ST-Link 无法连接。

### 现象
- ST-LINK Utility 报错：`Can not connect to target!`
- Keil 报错：`Flash Download failed - Target DLL has been cancelled`
- SW Device 列表无设备

### 根因分析
PA13/PA14 默认是 SWD 调试接口。程序运行后将其初始化为 GPIO，ST-Link 失去与芯片的通信通道。

### 解决方法
**Connect Under Reset 模式**：
1. ST-LINK Utility → Target → Settings
2. Mode 选择 `Connect Under Reset`
3. 点击连接，在复位信号拉低期间抢到 SWD 控制权
4. 成功识别后，立即 `Target → Erase Chip` 全片擦除
5. 擦除后 SWD 恢复默认功能，改回 Normal 模式

### 关键教训
1. **PA13/PA14 永远不要配置为普通 GPIO**，这是调试生命线
2. **Connect Under Reset 是救命稻草**，只要物理连线正常，99% 的 SWD 失联可用此法恢复
3. **配置 GPIO 前先看引脚默认功能**，CubeMX 会标注复用功能
4. **ST-LINK Utility 比 Keil 更可靠**，Keil 连不上时先用它排查

## 学到的

### 定时器中断
- `HAL_TIM_Base_Start_IT()` 启动定时器 + 中断模式
- `HAL_TIM_PeriodElapsedCallback()` 是回调函数，溢出时系统自动调用
- 中断里只置标志位，复杂逻辑放主循环

### 工程规范
- 全局变量放 `USER CODE BEGIN PV`
- 函数声明放 `USER CODE BEGIN PFP`
- 回调函数放 `USER CODE BEGIN 0`
- 主循环代码放 `USER CODE BEGIN 3`
- `__IO` 等价于 `volatile`，防止编译器优化中断共享变量

### 调试经验
- SWD 引脚保护机制
- Connect Under Reset 恢复流程
- 排查流程：Connect Under Reset → 手动复位 → 降频 → 检查接线

## 验证现象
| 时间 | 颜色 |
|------|------|
| 第 1 秒 | 红 |
| 第 2 秒 | 绿 |
| 第 3 秒 | 蓝 |
| 第 4 秒 | 红（循环）|

## 下一步
D3: 串口命令控制 RGB 灯，理解 UART 中断接收与协议解析。