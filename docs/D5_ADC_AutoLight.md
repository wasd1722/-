# D5: ADC 光敏自动调光

## 目标
通过 ADC 采集光敏传感器电压，根据环境亮度自动调节 RGB 灯 PWM 占空比，实现"天黑自动亮灯"。理解 ADC 分辨率、采样时间、转换时间、自动标定。

## 硬件
- 芯片：STM32F103C8T6
- RGB 灯：PA9(B)、PA10(G)、PA11(R)，共阴极，高电平点亮
- 光敏模块：AO→PA0(ADC1_IN0)，VCC→3.3V，GND→GND
- ADC 时钟：PCLK2 /6 = 12MHz（≤14MHz 合规）
- TIM1 高级定时器，Channel2/3/4 分别对应 B/G/R

## 环境
- STM32CubeMX + Keil MDK + ST-Link
- 时钟：HSE 8MHz → PLL 9倍频 → SYSCLK 72MHz

---

## CubeMX 配置

### 新增 ADC1
| 参数 | 值 | 说明 |
|------|-----|------|
| IN0 (PA0) | ADC1_IN0 | 光敏模块 AO 输出 |
| Clock Prescaler | PCLK2 divided by 6 | 72/6=12MHz，≤14MHz 合规 |
| Resolution | 12 bits | 满精度 |
| Data Alignment | Right alignment | 低 12 位有效 |
| Scan Conversion Mode | Disabled | 单通道 |
| Continuous Conversion Mode | Enabled | 启动后自动反复采 |
| Sampling Time (Channel 0) | 239.5 Cycles | 高阻抗源最保险档位 |

### 保留 TIM1（D4 配置）
| 参数 | 值 |
|------|-----|
| Clock Source | Internal Clock |
| Prescaler (PSC) | 71 |
| Counter Period (ARR) | 999 |
| Channel2 (PA9) | PWM Generation CH2 - B |
| Channel3 (PA10) | PWM Generation CH3 - G |
| Channel4 (PA11) | PWM Generation CH4 - R |
| PWM Mode | PWM Mode 1 |
| Pulse | 0 |
| Polarity | High |

---

## 关键代码

### 全局变量（USER CODE BEGIN PV）
```c
uint32_t adc_value = 0;    // ADC 原始值（0~4095）
uint16_t pwm_duty = 0;     // PWM 占空比（0~999）
```

### 启动 ADC（USER CODE BEGIN 2）
```c
/* D4 已有的 PWM 启动保留 */
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_2);  // PA9 - B
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_3);  // PA10 - G
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_4);  // PA11 - R

/* 新增 ADC */
HAL_ADCEx_Calibration_Start(&hadc1);   // 上电自校准
HAL_ADC_Start(&hadc1);                 // 启动连续转换
```

### 主循环——含自动标定（USER CODE BEGIN 3）
```c
while (1)
{
    adc_value = HAL_ADC_GetValue(&hadc1);

    /* 自动标定：持续追踪 ADC 的最小和最大值 */
    static uint32_t adc_min = 4095;
    static uint32_t adc_max = 0;
    if (adc_value < adc_min) adc_min = adc_value;
    if (adc_value > adc_max) adc_max = adc_value;

    /* 用实际范围映射，而不是全量程 0~4095 */
    if (adc_max > adc_min)
        pwm_duty = (uint16_t)((adc_value - adc_min) * 999 / (adc_max - adc_min));
    else
        pwm_duty = 0;

    /* 三通道同步 → 白光自动调亮度 */
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, pwm_duty);  // R
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, pwm_duty);  // G
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, pwm_duty);  // B

    HAL_Delay(20);   // 20ms 刷新，省 CPU
}
```

---

## 踩坑记录

### 现象1：遮光灯变暗（映射方向反了）
- 预期：遮住光敏模块 → 环境暗 → 灯变亮
- 实际：遮住灯反而变暗
- 原因：光敏模块"暗 → AO 电压高"，`pwm_duty = 999 - (adc_value * 999 / 4095)` 取反方向错了
- 解决：去掉 `999 -`，改为 `pwm_duty = adc_value * 999 / 4095`，暗→高ADC→高PWM→灯亮

### 现象2：亮度变化不明显（全量程映射浪费）
- 现象：遮光和光照之间亮度变化极小，像没反应
- 根因：光敏模块实际输出只覆盖 ADC 量程的一小段（比如 1500~3500），用 0~4095 线性映射时，有效区间只占 PWM 的一小截
- 解决：**自动标定**——代码运行时持续追踪 `adc_min` 和 `adc_max`，PWM 在这段实际区间内线性展开。上电后遮几下、照几下，标定自动跑完，变化幅度瞬间拉大

### 现象3：CubeMX 默认 ADC 时钟超标
- 现象：生成代码后发现 ADCCLK = 36MHz（PCLK2 /2）
- 问题：STM32F103 数据手册要求 12 位精度下 ADCCLK ≤ 14MHz，36MHz 远超上限，有效位数 ENOB 下降
- 解决：CubeMX Clock Configuration → ADC Prescaler 改为 /6 → 12MHz

---

## 学到的

### ADC 原理
- 12 位 = 4096 台阶，LSB = Vref / 4096 ≈ 0.806mV（Vref=3.3V 时）
- 量化误差 ±0.5 LSB，原理性不可消除
- SAR ADC 内部有采样保持电容，必须充到接近 Vin 才能准确采样

### 采样时间与源阻抗
- τ = (R_source + R_ADC) × C_sample
- 12 位精度需约 10τ，采样时间不够 → 电容未充满 → 读数偏低
- 239.5 Cycles 是最长档位，高阻抗源也扛得住
- 源阻抗 = 分压电阻并联值（Thevenin 等效）

### 转换时间
- 总转换时间 = 采样时间 + 12.5 Cycles（SAR 硬件固定开销）
- 239.5 + 12.5 = 252 Cycles ÷ 12MHz = **21μs**
- 1.5 + 12.5 = 14 Cycles ÷ 12MHz ≈ **1.17μs**（快 18 倍）
- ADCCLK 必须 ≤ 14MHz，常用 72MHz /6 = 12MHz

### 自动标定
- 传感器实际范围 ≠ 理论量程，线性映射浪费精度
- 动态追踪 min/max 可以自适应不同环境和模块
- 比手动串口打印省事，适合不需要绝对精度的场景

### 工程规范
- `uint32_t` 存 ADC 值，避免窄类型在 8 位 MCU 上溢出
- `HAL_ADCEx_Calibration_Start` 上电后首次读取前必须调用
- `adc_max > adc_min` 检查防止除以零
- `static` 变量保持标定状态，不死循环复位

---

## 验证现象

| 动作 | 灯状态 | 说明 |
|------|--------|------|
| 上电后遮光→照光→遮光（来回几次） | 标定跑完 | adc_min、adc_max 收敛 |
| 手遮住光敏模块 | 灯变亮 | 暗→高 ADC→高 PWM |
| 手机手电筒照模块 | 灯变暗 | 亮→低 ADC→低 PWM |
| 变化幅度 | 明显 | 自动标定在有效范围展开 |

---

## 下一步

D6: FreeRTOS 集成，把 ADC 采集和 PWM 调光拆成两个独立任务（ADCTask + PWMTask），用队列传递数据。
