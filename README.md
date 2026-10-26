# 电控组第二次培训作业

- **王语清**
- ***20263001319***

> 使用前请核对姓名与学号，如有误请自行修改。
> 注意星号必须紧贴文字：`**姓名**` 是粗体，`***学号***` 是粗体加斜体，两侧不能有空格。

---

## 一、任务概述

使用 `STM32CubeMX + CLion` 完成 STM32F103C8T6 三路 GPIO 输出控制，实现三个 LED 的 500 ms 流水灯；
并在基础版之外额外实现了**非阻塞延时**方案（`HAL_GetTick` 时间戳轮询），摆脱 `HAL_Delay()` 死等。

仓库包含两个独立工程：

| 目录 | 内容 | 延时方式 |
|------|------|----------|
| `01_Basic_Delay/` | 基础任务 | `HAL_Delay(500)` 阻塞延时 |
| `02_NonBlocking/` | 加分项 | `HAL_GetTick()` 时间戳轮询，主逻辑无任何 `HAL_Delay` |

## 二、硬件与环境

### 2.1 硬件

- 开发板：STM32F103C8T6 最小系统板（蓝色板，LQFP48，64 KB Flash / 20 KB RAM）
- 下载器：ST-Link V2（SWD 四线：3.3V / SWDIO / SWCLK / GND）
- 外设：面包板 + 3 个 5 mm LED（红 / 绿 / 黄）

### 2.2 引脚分配

| User Label | 引脚 | 模式 | 有效电平 | 接线 |
|------------|------|------|----------|------|
| `LED0` | PA0 | GPIO_Output / Push-Pull / Low speed | 低电平点亮 | 3V3 → LED 阳极，阴极 → PA0 |
| `LED1` | PA1 | GPIO_Output / Push-Pull / Low speed | 低电平点亮 | 同上 |
| `LED2` | PA2 | GPIO_Output / Push-Pull / Low speed | 低电平点亮 | 同上 |

> 采用「共阳 + 低电平点亮」（灌电流）接法：三颗 LED 的阳极统一接 3V3，
> 阴极分别接三个 GPIO。GPIO 输出低电平时电流由 3V3 经 LED 流入引脚，灯亮；
> 输出高电平时两端等电位，灯灭。该接法与板载 PC13 的极性一致，三路代码完全统一。

### 2.3 软件环境

- STM32CubeMX 6.15.0（HAL 固件包 STM32Cube FW_F1 V1.8.7）
- CLion 2026.2（CMake + Ninja，GNU Arm Embedded Toolchain 10.3-2021.10）
- OpenOCD 0.12（`D:\OpenOCD`），烧录配置使用 `CFJ_STM32\stm32f1.cfg`
- 系统时钟：HSE 8 MHz → PLL ×9 → **HCLK 72 MHz**（APB1 = 36 MHz，APB2 = 72 MHz）

## 三、CubeMX 关键配置

1. `SYS → Debug = Serial Wire`（保留 SWD 调试口，避免烧录后无法再次连接）
2. `RCC → HSE = Crystal/Ceramic Resonator`，`LSE = Disable`，时钟树配至 72 MHz
3. PA0 / PA1 / PA2 配置为 `GPIO_Output`，**User Label 分别填 `LED0` / `LED1` / `LED2`**，
   输出电平默认 `High`（上电全灭），推挽输出、无上下拉、低速
4. `Code Generator` 不勾选「每个外设生成 .c/.h 对」，所有初始化保留在 `main.c`

CubeMX 依据 User Label 自动生成以下宏（`Core/Inc/main.h`）：

```c
#define LED0_Pin        GPIO_PIN_0
#define LED0_GPIO_Port  GPIOA
#define LED1_Pin        GPIO_PIN_1
#define LED1_GPIO_Port  GPIOA
#define LED2_Pin        GPIO_PIN_2
#define LED2_GPIO_Port  GPIOA
```

## 四、基础任务实现（HAL_Delay 版）

实现思路：在 `while(1)` 主循环中，每一拍先熄灭其余两灯、再点亮目标灯，然后 `HAL_Delay(500)` 等待 500 ms，
三拍循环即形成流水灯效果。

核心代码（`01_Basic_Delay/Core/Src/main.c`，`USER CODE BEGIN 3` 内）：

```c
/* 第 1 拍：只有 LED0 亮（低电平点亮） */
HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_RESET);
HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET);
HAL_Delay(500);

/* 第 2 拍：只有 LED1 亮 */
HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET);
HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET);
HAL_Delay(500);

/* 第 3 拍：只有 LED2 亮 */
HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_RESET);
HAL_Delay(500);
```

**缺点**：`HAL_Delay()` 内部是 `while ((HAL_GetTick() - tickstart) < Delay);` 的空循环，
这 500 ms 内 CPU 被完全占用，无法响应按键、串口、CAN 等任务，因此有了后面的加分项。

## 五、加分项：非阻塞延时实现

### 5.1 为什么 `HAL_Delay()` 是「死等」

```c
void HAL_Delay(uint32_t Delay)
{
  uint32_t tickstart = HAL_GetTick();
  while ((HAL_GetTick() - tickstart) < Delay) { }  /* CPU 在这里空转 500 ms */
}
```

非阻塞的核心思想是：**不「等」，而是「查时间到了没有」**——每轮循环只做一次比较，
未到期立即返回，把 CPU 让给其他任务。

### 5.2 实现原理

1. `HAL_GetTick()` 返回 SysTick 累加的开机毫秒数（1 ms 精度，`uint32_t`）。
2. 用静态变量 `s_last_tick` 记录上一次切换灯的时刻。
3. 每轮主循环计算 `now - s_last_tick`，**≥ 500 才切换**，否则立即返回，不阻塞。
4. 切换时执行 `s_last_tick += 500`（而非 `= now`），避免主循环偶尔被拖慢造成的累计漂移。
5. `HAL_GetTick()` 约 49.7 天溢出一次，采用**无符号减法**比较可自动正确处理回绕，
   因此必须写成 `(uint32_t)(now - last) >= 500`，而不能写成 `now >= last + 500`。

### 5.3 核心代码

```c
/* USER CODE BEGIN 0 */
#define LED_STEP_MS     500u

static uint32_t s_last_tick = 0;
static uint8_t  s_step      = 0;

static void LED_AllOff(void)
{
  HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_SET);
  HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET);
  HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET);
}

static void LED_Show(uint8_t step)
{
  LED_AllOff();
  switch (step)
  {
    case 0:  HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_RESET); break;
    case 1:  HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET); break;
    default: HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_RESET); break;
  }
}
/* USER CODE END 0 */
```

```c
/* USER CODE BEGIN 3 */
/* 核心：只判断「时间到了没」，没到就立刻返回 —— 不阻塞 */
if ((uint32_t)(HAL_GetTick() - s_last_tick) >= LED_STEP_MS)
{
  s_last_tick += LED_STEP_MS;               /* 累加而非 = now，避免累计漂移 */
  s_step = (uint8_t)((s_step + 1u) % 3u);   /* 0 → 1 → 2 → 0 */
  LED_Show(s_step);
}
/* 这 500 ms 里 CPU 是空闲的，可以放按键扫描、串口处理等任务 */
/* USER CODE END 3 */
```

### 5.4 与 `HAL_Delay` 的对比

| 维度 | `HAL_Delay` | `HAL_GetTick` 轮询 |
|------|-------------|--------------------|
| CPU 占用 | 100% 空转 | ≈ 0（比较后立即返回） |
| 计时精度 | ±1 ms | ±1 ms + 主循环抖动 |
| 可否并行多任务 | 否 | 是（可在主循环插入任意任务） |
| 代码复杂度 | 最低 | 低 |
| 适用场景 | 简单演示 | 多数中小项目 |

## 六、验收视频

| 文件 | 对应工程 | 内容 |
|------|----------|------|
| `videos/01_Basic_500ms_流水灯.mp4` | `01_Basic_Delay` | CubeMX 引脚标签 → CLion 编译通过 → 启动烧录 → `Programming Finished` → 板上 500 ms 流水 |
| `videos/02_NonBlocking_500ms_流水灯.mp4` | `02_NonBlocking` | 同上，且代码中无 `HAL_Delay` |

## 七、遇到的问题与解决

1. **ST-Link 驱动未安装，设备管理器显示黄色感叹号**
   使用队伍环境包里的 WinUSB 驱动（`LinkScope\openocd\drivers\ST-Link\stlink_dbg_winusb.inf`）安装后，
   设备从 `Error` 状态变为 `OK`，服务为 `WinUSB`。

2. **OpenOCD 报 `Error: init mode failed (unable to connect to the target)`**
   原因是 SWD 四根线的顺序与板子丝印不一致，且板子未独立供电。
   按丝印逐根核对（3.3V→3V3、SWDIO→DIO、SWCLK→CLK、GND→GND）并给板子插上 USB 后连通，
   读到 `SWD DPIDR 0x1ba01477` 与 `flash size = 64 KiB`。

3. **CubeMX 生成的链接脚本导致 GCC 10 链接失败**
   报 `non constant or forward reference address expression for section .ARM.extab`。
   原因是新版 CubeMX 的 `.ld` 使用了 `(READONLY)` 关键字（仅 GCC 11+ 支持），
   删除全部 `(READONLY)` 后链接正常。

4. **CLion 未自动生成 CMake 工程、调试时调用了 x86 版 GDB**
   手动补齐 `CMakeLists.txt` / `CMakePresets.json` / `cmake/` 脚手架，
   并在工具链设置中把调试器指定为 `arm-none-eabi-gdb.exe`，
   运行配置使用 OpenOCD + `CFJ_STM32\stm32f1.cfg`。

---

*本工程为 RoboMaster 2027 电控组第二次招新培训作业。*