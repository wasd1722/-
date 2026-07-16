# D1: RGB 灯控制

## 硬件
- 芯片：STM32F103C8T6
- RGB 灯：PA9(蓝 B)、PA10(绿 G)、PA11(红 R)
- 共阴：高电平点亮

## 目标
通过 HAL 库控制 RGB 灯显示不同颜色。

## 关键代码
```c
void RGB_SetColor(uint8_t r, uint8_t g, uint8_t b)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_11, r ? GPIO_PIN_SET : GPIO_PIN_RESET);  // R - PA11
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_10, g ? GPIO_PIN_SET : GPIO_PIN_RESET);  // G - PA10
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9,  b ? GPIO_PIN_SET : GPIO_PIN_RESET);  // B - PA9
}

// 主循环：循环显示红、绿、蓝
while (1)
{
    RGB_SetColor(1, 0, 0);  // 红
    HAL_Delay(500);
    RGB_SetColor(0, 1, 0);  // 绿
    HAL_Delay(500);
    RGB_SetColor(0, 0, 1);  // 蓝
    HAL_Delay(500);
}
