# D7: 信号量与互斥锁 — 蓝牙命令抢灯控权

## 目标
在 D6 双任务基础上添加蓝牙命令任务，用互斥锁保护共享变量（模式标志 + 手动亮度），信号量作为通知机制（本次配置但未启用）。理解互斥锁所有权、优先级继承、二进制信号量 vs 互斥锁的本质区别。

## 硬件
- 芯片：STM32F103C8T6
- RGB 灯：PA9(B)、PA10(G)、PA11(R)，TIM1 PWM
- 光敏模块：AO→PA0(ADC1_IN0)
- 蓝牙模块：HC-05，USART2，38400 波特
- FreeRTOS：CMSIS-RTOS v2

## 环境
- STM32CubeMX + Keil MDK + ST-Link
- 手机蓝牙串口 APP
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

---

## CubeMX 配置

### 新增任务
| 任务 | 入口函数 | 优先级 | 栈大小 |
|------|---------|--------|--------|
| BTCommandTask | btCmdTaskFunc | osPriorityNormal | 128 words |

**为什么 Normal**：蓝牙人工操作不需要高于 ADC/PWM 的优先级。50ms 轮询频率足够，设 High 反而可能因蓝牙数据涌入饿死 ADCTask。

**为什么栈 128**：逻辑仅判断 `rx_cmd` 字符 + 改变量 + Lock/Unlock，无大数组无 `sprintf`。栈够用。

### 新增互斥锁
| 名称 | 类型 |
|------|------|
| lightMutex | Mutex (binary) |

**为什么需要互斥锁**：`light_mode` 和 `manual_pwm` 被两个任务同时读写——BTCommandTask 写、PWMTask 读。不锁保护 → 调度器可能在赋值中间切走 → PWMTask 读到半截值。

**为什么不用二进制信号量当锁**：信号量无所有权检查——任何任务都能 Give/Release。互斥锁记录"谁 Lock 的"，只有 Lock 者才能 Unlock；且支持优先级继承，防止 Low 任务拿着锁被 Medium 饿死导致 High 任务死等。

### 新增信号量（配置但未启用）
| 名称 | 类型 | 初始值 |
|------|------|--------|
| lightSemaphore | Binary | 1 |

**为什么配置了未使用**：PWMTask 已阻塞在 `osMessageQueueGet(osWaitForever)`——ADC 每 20ms 必有新数据到达唤醒它，唤醒后顺手检查 `light_mode`。BT 命令的响应延迟 ≤20ms，人工按键完全无感。信号量的真正用武之地在后继 DMA 章节——当 PWMTask 不靠队列阻塞时，需信号量做外部唤醒。

### 保留外设（D6 配置）
- TIM1 PWM、ADC1、USART2、FreeRTOS 全部不动

---

## 关键代码

### 共享变量（USER CODE BEGIN PV）
```c
static uint8_t  light_mode = 0;     // 0=AUTO(默认), 1=MANUAL
static uint32_t manual_pwm = 500;   // 手动亮度默认 50%
```

**为什么 static**：仅本文件可见，外部 `.c` 不能意外修改。
**为什么放 PV 不在任务内**：三个任务都要读写同一个变量，放在任务外共享。

### BTCommandTask（USER CODE BEGIN btCmdTaskFunc）
```c
for(;;)
{
    if(rx_cmd != 0)
    {
        osMutexAcquire(lightMutexHandle, osWaitForever);

        if (rx_cmd == 'A' || rx_cmd == 'a')
            light_mode = 0;                        // 切回自动
        else if (rx_cmd == 'M' || rx_cmd == 'm')
            light_mode = 1;                        // 切手动
        else if (rx_cmd == 'O' || rx_cmd == 'o')
        {
            manual_pwm = 0;                        // 熄灯
            light_mode = 1;
        }
        else if (rx_cmd >= '1' && rx_cmd <= '9')
        {
            manual_pwm = (uint32_t)(rx_cmd - '0') * 111;  // '1'→111, '9'→999
            light_mode = 1;                        // 数字自动切手动
        }

        osMutexRelease(lightMutexHandle);
    }
    rx_cmd = 0;
    osDelay(50);
}
```

**为什么 Lock 包住改变量的区域**：临界区只保护"改变量"，不保护 `osDelay` 之类的外围逻辑。
**为什么数字键 `*111`**：ARR=999，分 10 档（'0'~'9'），每档约 111，'5'→555≈55% 亮度。
**为什么 `'O'` 熄灯设 `light_mode=1`**：熄灯是手动操作，不设 MANUAL 状态会被 ADC 自动值覆盖。

### PWMTask（修改自 D6）
```c
for(;;)
{
    osMessageQueueGet(adcQueueHandle, &pwm, NULL, osWaitForever);

    osMutexAcquire(lightMutexHandle, osWaitForever);
    if (light_mode == 1)       // MANUAL: 覆盖队列值
        pwm = manual_pwm;
    // else: AUTO, 不改 pwm，用 ADCTask 队列值
    osMutexRelease(lightMutexHandle);

    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, pwm);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, pwm);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, pwm);
}
```

### ADCTask — 不动
照旧采集、标定、发队列。不管模式——读写分离。

---

## 设计要点

- 信号量本次配置但未启用：PWMTask 已靠队列阻塞唤醒，BT 命令 20ms 延迟人工无感；后续 DMA 章节会真正用到 `osSemaphoreRelease/Acquire`
- ISR 与任务共享 `rx_cmd` 时，读-改-写操作尽量在临界区外快照局部变量，但单字节原子操作在当前场景无实质风险

---

## 学到的

### 互斥锁
- **所有权**：谁 Lock 谁 Unlock，别人硬解会被拒
- **优先级继承**：High 任务等 Low 任务的锁时，RTOS 临时提升 Low 至 High，防止 Medium 间接饿死 High
- **临界区**：Lock→操作共享变量→Unlock，之间的代码越短越好，不包 `osDelay`
- **osWaitForever**：不设超时——锁持有时间极短（几微秒），不存在死锁风险

### 二进制信号量
- **仅一种**：`osSemaphoreNew(1, N, ...)` 第一个参数为 1
- **与互斥锁的本质区别**：无所有权、无优先级继承。二分信号量是"门铃"，互斥锁是"带身份证的门锁"
- **通知 vs 锁**：A Release → B Acquire（通知）；同任务 Lock → 同任务 Unlock（锁）

### 中断在 FreeRTOS 中的位置
- 硬件中断凌驾于所有任务之上，ISR 里不能调阻塞 API（osDelay/osMutexAcquire 等）
- `FromISR` 变种：`osSemaphoreReleaseFromISR`、`osMessageQueuePutFromISR`——不阻塞，ISR 安全
- UART 中断回调完全不变——`rx_cmd = ch` 仍在 ISR 内执行，BTCommandTask 任务层轮询读取

### 架构演进
| 版本 | 任务数 | 通信方式 | 新问题 |
|------|--------|---------|--------|
| D5 裸机 | 0（超循环） | 无 | 加功能改 while(1) |
| D6 FreeRTOS | 2（ADC+PWM） | 队列 | 单生产者单消费者 |
| D7 多任务 | 3（+BT命令） | 队列 + 互斥锁 | 多任务共享状态需锁保护 |

---

## 验证现象

| 手机发送 | 现象 |
|---------|------|
| A | 切回自动模式，灯光随环境亮度变化 |
| M | 切手动模式，保持当前亮度（默认 50%） |
| 1~9 | 手动模式，亮度 10%~90%（每档约 11%） |
| O | 手动模式熄灯 |
| 遮光/照光 | 仅在 AUTO 模式响应，MANUAL 模式不随光照变化 |

---

## 下一步

D8: DMA + ADC 连续采集，或嵌入式部署（Bootloader / OTA / IWDG）。
