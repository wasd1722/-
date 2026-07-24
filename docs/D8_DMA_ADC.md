# D8: DMA + ADC 连续采集

## 目标
用 DMA 替代 CPU 轮询搬运 ADC 数据，ADCTask 改为信号量驱动的事件响应模式。理解 DMA 总线架构、环形缓冲区、CMSIS-RTOS v2 中断安全 API。

## 硬件
- 芯片：STM32F103C8T6
- 光敏模块：AO→PA0(ADC1_IN0)
- DMA：DMA1 Channel 1（F103 硬连线）
- 信号量：lightSemaphore（D7 配置，D8 启用）

## 环境
- STM32CubeMX + Keil MDK + ST-Link
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

---

## CubeMX 配置

### ADC1 → DMA Settings（新增）
**在哪**：`Analog → ADC1 → DMA Settings 标签 → Add → ADC1`

| 参数 | 值 | 为什么 |
|------|-----|--------|
| DMA Request | ADC1 | F103 硬连线 DMA1_Channel1 |
| Mode | Circular | 环形——写完最后一个绕回第一个，永不停 |
| Data Width | Half Word (16-bit) | ADC_DR 寄存器 16 位宽 |
| Increment → Peripheral | 不勾 | ADC_DR 只有一个物理地址 |
| Increment → Memory | 勾上 | buf[0]→buf[1]→...DMA 自动推进写指针 |
| Priority | Low | 无多路 DMA 竞争 |

### NVIC
`DMA1 Channel 1 global interrupt` 勾上。

### lightSemaphore → Initial State
**在哪**：`FREERTOS → Semaphores → lightSemaphore`

| 参数 | 旧值 | 新值 | 为什么 |
|------|------|------|--------|
| Initial State | Available | **Not Available** | 初始计数从 1 改 0——上电时没事件，ADCTask 第一次 Acquire 等 DMA 填满缓冲区 |

对应代码：`osSemaphoreNew(1, 0, &attributes)`

### 保留外设（D7 配置）
- TIM1 PWM、USART2、FreeRTOS 任务/队列/互斥锁全部不动

---

## 关键代码

### 全局变量（USER CODE BEGIN PV）
```c
uint16_t adc_dma_buf[128];   /* DMA 环形缓冲区 */
```

### ADC 启动（USER CODE BEGIN 2）
```c
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_dma_buf, 128);
```
替代 D5 的 `HAL_ADC_Start`。三个参数：ADC 句柄、目标缓冲区首地址、传输数量。

### DMA 完成回调（USER CODE BEGIN 0）
```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
    if (hadc->Instance == ADC1)
        osSemaphoreRelease(lightSemaphoreHandle);
}
```
DMA 写满 128 个样本触发中断 → 回调里释放信号量。CMSIS-RTOS v2 的 `osSemaphoreRelease` 自动判断 ISR 上下文，无需 `FromISR` 后缀。

### ADCTask（信号量驱动版）
```c
static uint32_t adc_min = 4095;
static uint32_t adc_max = 0;

for(;;)
{
    osSemaphoreAcquire(lightSemaphoreHandle, osWaitForever);  /* 等 DMA 填满一帧 */

    uint32_t sum = 0;
    for (int i = 48; i < 80; i++)
        sum += adc_dma_buf[i];
    uint32_t val = sum / 32;                           /* 32 个样本平均 */

    if (val < adc_min) adc_min = val;
    if (val > adc_max) adc_max = val;
    uint32_t pwm = 0;
    if (adc_max > adc_min)
        pwm = (val - adc_min) * 999 / (adc_max - adc_min);

    osMessageQueuePut(adcQueueHandle, &pwm, 0, 0);
}
```
**为什么不留 `osDelay`**：`osSemaphoreAcquire` 本身就是阻塞点。DMA 填满一帧（128×21μs≈2.7ms）才发一次信号量，任务被唤醒处理完立刻回去等下一帧——纯事件驱动，不空转。

### PWMTask、BTCommandTask — 不动

---

## 踩坑记录

### 坑1：信号量初始值 1 导致首帧读到零
- 现象：复位后 `adc_min` 掉到 0，标定失效
- 原因：CubeMX 默认信号量初始 1（Available），ADCTask 第一次 `Acquire` 不阻塞直接通过，此时 DMA 还未填满缓冲区——读到初值为 0 的数组
- 解决：CubeMX Semaphore 配置 → Initial State 改为 Not Available（对应 `osSemaphoreNew(1,0,...)`）
- 教训：信号量用于通知场景，初始计数必须 0——"还没发生任何事件"

### 坑2：CMSIS-RTOS v2 无 FromISR 后缀
- 现象：`osSemaphoreReleaseFromISR` 编译报 undefined symbol
- 原因：CMSIS-RTOS v2 统一了 API——`osSemaphoreRelease` 内部自动判断调用上下文
- 解决：直接用 `osSemaphoreRelease`，删掉 `static BaseType_t` 和 `portYIELD`

---

## 学到的

### DMA 总线架构
- DMA 是独立于 CPU 的总线控制器，ADC 转换完成 → DMA 自动搬运数据到 RAM，CPU 零参与
- F103 的 ADC1 固定绑定 DMA1_Channel1（硅片硬连线）
- Circular 模式：写完最后一个自动绕回第一个，一次启动永不停
- Half Word (16-bit) 对齐 ADC_DR 寄存器宽度

### 环形缓冲区
- DMA 在 128 个元素的环形数组上持续写入
- CPU 在 DMA 满中断后取中间 32 个样本平均，抗单点抖动
- 写指针由 DMA 自动推进，CPU 只读不写

### 信号量的正确初始值
- 通知场景（事件驱动）：初始计数 0——"没事件，等"
- 资源管理场景（池）：初始计数 N——"N 个资源可用"
- CubeMX 默认 Available(1) 适合资源池，不适合 DMA 完成通知

### CMSIS-RTOS v1 vs v2
- v1: `osSemaphoreRelease`（任务级）/ `osSemaphoreReleaseFromISR`（中断级）
- v2: `osSemaphoreRelease`（统一，内部自动判断上下文）
- 选 v2（Interface: CMSIS_V2）避免 FromISR 后缀问题

### 架构演进
| 版本 | ADC 方式 | 唤醒机制 | CPU 效率 |
|------|---------|---------|---------|
| D5 裸机 | CPU 轮询 `HAL_ADC_GetValue` | `HAL_Delay` 空转 | 50Hz 每次 CPU 全参与 |
| D7 FreeRTOS | CPU 轮询 + `osDelay(20)` | 主动让出 CPU | 50Hz 每次 CPU 参与 |
| D8 DMA | DMA 环形 2.7ms/帧 | 信号量事件驱动 | CPU 仅被叫醒处理，余下时间休眠 |

---

## 验证现象

| 操作 | 现象 |
|------|------|
| 上电复位 | ADC Task 等第一帧填满后才处理，标定正常 |
| 遮住光敏模块 | ADC 值降 → 灯变亮 |
| 手电筒照模块 | ADC 值升 → 灯变暗 |
| 手动模式（蓝牙） | B 命令切手动，不受 ADC 影响 |

---

## 下一步

超声波 HC-SR04 输入捕获，或信号量进阶（任务通知 / 计数信号量）。
