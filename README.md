# 电控组第二次培训作业

- **王语清**
- ***20263001319***

## 实现思路

PA0 / PA1 / PA2 配置为 GPIO 输出，低电平点亮。主循环中每一拍先熄灭其余两灯、
再点亮目标灯，循环切换即形成 500 ms 流水灯。

```c
HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_RESET);
HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET);
HAL_Delay(500);
```

## 加分项：非阻塞延时

`HAL_Delay()` 内部是 `while ((HAL_GetTick() - tickstart) < Delay);` 的空循环，
这 500 ms 内 CPU 无法处理其他任务。

改用 `HAL_GetTick()` 时间戳轮询：每轮循环只比较「当前时刻 − 上次切换时刻」，
未到 500 ms 立即返回，不阻塞主循环；到时则推进时间戳并切换下一拍。

```c
if ((uint32_t)(HAL_GetTick() - s_last_tick) >= 500u)
{
  s_last_tick += 500u;
  s_step = (uint8_t)((s_step + 1u) % 3u);
  LED_Show(s_step);
}
```

`01_Basic_Delay/` 为基础版，`02_NonBlocking/` 为非阻塞版（工程内无任何 `HAL_Delay`）。
