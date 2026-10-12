# 供电配置 BUG 详解 —— 为什么 `PWR_DIRECT_SMPS_SUPPLY` 会让调试口消失

> **现象**：OpenOCD 连不上，报 `Error connecting DP: cannot read IDR`。按复位、换速率、换复位策略、连 JTAG 全都无效 —— 但 **BOOT0 拉高后复位，SWD 立刻连上**。
>
> **结论**：[main.c:249](CM7/Core/Src/main.c#L249) 那一行 `HAL_PWREx_ConfigSupply(PWR_DIRECT_SMPS_SUPPLY)` 把 `PWR_CR3` 的 **bit1（LDOEN）从 1 改成了 0**，内核供电来源被切掉。芯片跑固件几毫秒内就没了电 —— 连调试逻辑一起。改回 `PWR_LDO_SUPPLY` 即可。
>
> 本文只讲**代码**。硬件为什么可以排除放在 §1 里一句带过，因为它本身就是一条代码层面的推理。

---

## 目录

| 节 | 内容 |
|---|---|
| [§1](#1-先立一条判据为什么普通崩溃解释不了这个现象) | 先立一条判据：为什么普通崩溃解释不了这个现象 |
| [§2](#2-排查固件里谁动了电源域) | 排查：固件里谁动了电源域 |
| [§3](#3-第-249-行为什么真的会写进硬件) | 第 249 行为什么**真的**会写进硬件 |
| [§4](#4-写下去之后为什么芯片会死) | 写下去之后为什么芯片会死 |
| [§5](#5-为什么按复位救不回来为什么-boot01-能救) | 为什么"按复位"救不回来，为什么 BOOT0=1 能救 |
| [§6](#6-为什么这个-bug-特别安静) | 为什么这个 bug 特别安静 |
| [§7](#7-改法与烧录顺序) | 改法与烧录顺序 |
| [§8](#8-判据可证伪) | 判据（可证伪） |
| [§9](#9-复发陷阱cubemx-会把它写回去) | 复发陷阱：CubeMX 会把它写回去 |
| [§10](#10-源码索引) | 源码索引（哪条结论由哪一行证明） |

---

## 1. 先立一条判据：为什么普通崩溃解释不了这个现象

`cannot read IDR` 这句话的意思很具体：**调试端口（DAP）自己没有应答**。不是"连上了但读不了 flash"，是**连门都没敲开**。

这条判据的价值在于 —— **CPU 跑死，DAP 照样应答。**

| 固件干了什么 | DAP 还活着吗 |
|---|---|
| 死循环（`while(1)`） | ✅ 活着 |
| HardFault / 跑飞 / 跳到地址 0 | ✅ 活着 |
| 野指针写坏 RAM | ✅ 活着 |
| 栈溢出 | ✅ 活着 |
| 时钟配错，CPU 卡在等某个标志 | ✅ 活着 |
| **供电域塌了（VCORE 没了）** | ❌ **死** |
| **芯片被按住不停复位** | ❌ **死** |

原因：DAP 在**独立的电源/时钟域**里，它只依赖调试时钟（由 `DBGMCU_CR` 的 `DBG_CKD1EN`/`DBG_CKD3EN` 门控，**复位默认是开的**）。CPU 崩了不会顺手把自己的调试时钟关掉。

> **所以：看到 `cannot read IDR`，第一反应不该是"我的逻辑写错了"，而是"我的固件动了电源或复位域"。**

这一条把搜索范围从"整个工程"压到"电源/复位相关的那几行"。下面的排查就是顺着这条走。

---

## 2. 排查：固件里谁动了电源域

全工程搜一遍所有能碰电源域、复位域、调试域的调用（工程自己的源码，不含 `Drivers/` 和 `Middlewares/`）：

```bash
grep -rn 'ConfigSupply\|PWR_CR\|PWR_REGULATOR\|VOS' CM7/Core/Src CM7/Core/Inc BSP/
grep -rn 'DBGMCU\|HAL_DBGMCU'                       CM7/Core/Src CM7/Core/Inc BSP/
grep -rn 'HAL_PWR\|__WFI\|__WFE\|PWR_CPUCR\|EnterSTOP\|EnterSTANDBY' CM7/Core/Src CM7/Core/Inc BSP/
```

结果：**全部命中都在同一个函数里** —— [`SystemClock_Config()`](CM7/Core/Src/main.c#L242)，而且只有三行：

| 行 | 代码 | 读到的寄存器 | 判据 |
|---|---|---|---|
| [249](CM7/Core/Src/main.c#L249) | `HAL_PWREx_ConfigSupply(PWR_DIRECT_SMPS_SUPPLY);` | `PWR_CR3` | **会改硬件** ← 就是它 |
| [253](CM7/Core/Src/main.c#L253) | `__HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE3);` | `PWR->D3CR` | **不能**导致本现象 |
| [255](CM7/Core/Src/main.c#L255) | `while(!__HAL_PWR_GET_FLAG(PWR_FLAG_VOSRDY)) {}` | `PWR->D3CR` | **不能**导致本现象 |

`DBGMCU`、低功耗模式、`PWR_CPUCR` 全部**零引用**。工程里没有任何一行代码去关调试时钟、也没有任何一行进低功耗模式。

### 为什么 253 / 255 可以排除（两次独立论证，任一条成立就够了）

**论证一：VOS 只在"允许的档位之间"选电压，不可能让内核断电。**

`__HAL_PWR_VOLTAGESCALING_CONFIG` 展开后是 `MODIFY_REG(PWR->D3CR, PWR_D3CR_VOS, ...)`（[stm32h7xx_hal_pwr.h:292](Drivers/STM32H7xx_HAL_Driver/Inc/stm32h7xx_hal_pwr.h#L292)）。`PWR_D3CR_VOS` 是 `PWR->D3CR` 的 bit[15:14] —— [位定义](Drivers/CMSIS/Device/ST/STM32H7xx/Include/stm32h757xx.h#L18285)。四个可能的取值就是四个电压档位，`SCALE3` 是最低档（64 MHz 远在它的频率上限之内）。**它改的是"内核吃 0.9 V 还是 1.1 V"，不是"内核吃不吃电"。**

**论证二（更强）：就算它真卡住或配错，也解释不了本现象。**

回到 §1 的判据 —— 卡住 ≠ 调试口消失。所以这两行**无论怎么错都不可能是原因**，不需要再深究。

> **这一步的推理方式值得记下来：不是"我看了这两行觉得没问题"，而是"这两行的错误模式不在候选集合里"。**

### 顺带一个副作用（不影响结论，但解释了这个 bug 为什么更安静）

`HAL_PWREx_ConfigSupply` 在写完 `PWR_CR3` **之后**还有一段等待：

```c
MODIFY_REG (PWR->CR3, PWR_SUPPLY_CONFIG_MASK, SupplySource);   /* ← 写已经发生了 */
tickstart = HAL_GetTick ();
while (__HAL_PWR_GET_FLAG (PWR_FLAG_ACTVOSRDY) == 0U)
{
  if ((HAL_GetTick() - tickstart) > PWR_FLAG_SETTING_DELAY) { return HAL_ERROR; }
}
```

**关键：寄存器在进入这个循环之前就已经被改了。** 后面等不等得到、返回 `HAL_OK` 还是 `HAL_ERROR`，都改变不了"已经写下去"这个事实。而 `main.c:249` 没接返回值 —— 所以连这个 `HAL_ERROR` 都没人看。

---

## 3. 第 249 行为什么**真的**会写进硬件

这是整个 BUG 的技术核心。`HAL_PWREx_ConfigSupply` 开头有一个"配置是否已锁"的守卫（[stm32h7xx_hal_pwr_ex.c:325](Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_pwr_ex.c#L325)）：

```c
#if defined (PWR_FLAG_SCUEN)
  if (__HAL_PWR_GET_FLAG (PWR_FLAG_SCUEN) == 0U)
#else
  if ((PWR->CR3 & (PWR_CR3_SMPSEN | PWR_CR3_LDOEN | PWR_CR3_BYPASS)) != (PWR_CR3_SMPSEN | PWR_CR3_LDOEN))
#endif
  {
    if ((PWR->CR3 & PWR_SUPPLY_CONFIG_MASK) != SupplySource)
    {
      return HAL_ERROR;          /* ← 锁住了，改不了 */
    }
    else
    {
      return HAL_OK;             /* ← 锁住了，但请求值跟现状一样，不用改 */
    }
  }

  MODIFY_REG (PWR->CR3, PWR_SUPPLY_CONFIG_MASK, SupplySource);   /* ← 真正写的地方 */
```

**走哪个分支，取决于 `PWR_FLAG_SCUEN` 有没有定义。**

### 第 1 步：`PWR_FLAG_SCUEN` 到底有没有定义

这一条查起来容易骗人 —— grep `PWR_FLAG_SCUEN` 会命中，但它住在 `#else` 里：

```c
/* stm32h7xx_hal_pwr.h:175 */
#if defined (SMPS)
#define PWR_FLAG_SMPSEXTRDY (0x11U)
#else
#define PWR_FLAG_SCUEN      (0x11U)      /* ← 这一支不生效 */
#endif /* defined (SMPS) */
```

而 `SMPS` **是定义了的**：

```c
/* stm32h757xx.h:218 */
#define SMPS       /*!< Switched mode power supply feature */
```

> **⇒ `PWR_FLAG_SCUEN` 没有定义。走 `#else` 分支。**

> **教训**：判断一个宏的"值"，先看它所在的 `#if` 分支是否生效。**grep 命中 ≠ 定义生效。**

### 第 2 步：`#else` 那个条件成不成立

代入各常量的值（[stm32h757xx.h:18203-18210](Drivers/CMSIS/Device/ST/STM32H7xx/Include/stm32h757xx.h#L18203)）：

| 常量 | 值 |
|---|---|
| `PWR_CR3_SMPSEN` | `0x04`（bit2） |
| `PWR_CR3_LDOEN` | `0x02`（bit1） |
| `PWR_CR3_BYPASS` | `0x01`（bit0） |

守卫变成：

```c
if ((PWR->CR3 & (0x04 | 0x02 | 0x01)) != (0x04 | 0x02))
   if ((PWR->CR3 & 0x07) != 0x06)
```

**夹子实测**：`PWR_CR3 = 0x05000046`（地址 `0x5802480C`）

```
0x05000046
      0x46 = 0100 0110b
               │││└─ bit0 BYPASS    = 0
               ││└── bit1 LDOEN     = 1  ← LDO 开着
               │└─── bit2 SMPSEN    = 1  ← SMPS 也开着
               └──── bit3 SMPSEXTHP = 0
              bit4/5  SMPSLEVEL     = 00
              bit6                    = 1
         bit24 USB33DEN            = 1
         bit26 USB33RDY（只读状态）  = 1
```

`0x46 & 0x07 = 0x06` == `(SMPSEN|LDOEN)` → **条件为假 → 整个守卫块被跳过 → 直接执行 `MODIFY_REG`。**

### 第 3 步：`MODIFY_REG` 具体改了哪一位

```c
/* PWR_SUPPLY_CONFIG_MASK（SMPS 分支，stm32h7xx_hal_pwr_ex.h:248）*/
= SMPSLEVEL | SMPSEXTHP | SMPSEN | LDOEN | BYPASS
= 0x30     | 0x08      | 0x04  | 0x02  | 0x01
= 0x3F                            ← 掩码只盖 bit[5:0]，bit6 不在里面

/* SupplySource */
= PWR_DIRECT_SMPS_SUPPLY
= PWR_CR3_SMPSEN
= 0x04
```

逐位算：

```
新值 = (0x05000046 & ~0x3F) | 0x04
     =  0x05000040 | 0x04
     =  0x05000044
```

**前后对照：**

| | `PWR_CR3` 值 | bit1 LDOEN | bit2 SMPSEN |
|---|---|---|---|
| 改之前（复位默认） | `0x05000046` | **1** | 1 |
| 改之后（固件跑过） | `0x05000044` | **0** | 1 |
| 差异 | **只有 bit1** | 1 → 0 | 不变 |

> **这就是全部了：这一行对 `PWR_CR3` 的净效果，是恰好一个 bit —— bit1（LDOEN）从 1 变成 0。**
>
> （bit6 之所以保留，是因为掩码 `0x3F` 盖不到它 —— 这也顺便说明算式要对齐掩码，不能只看"不想要哪些位"。）

### 第 4 步：为什么这个守卫**故意**放它过去

不是 HAL 写漏了，是设计意图。RST 之后 `PWR_CR3` 的默认值是"SMPS 和 LDO 都开着"（`0x06`）—— 这个状态的含义是 **"还没人配置过供电"**。所以：

| `PWR_CR3 & 0x07` | 含义 | 守卫的态度 |
|---|---|---|
| `0x06`（两个都开） | **没配置过**（复位默认） | **放行，允许写** |
| 其它任何值 | 已经明确配置过 | 拦住，返回 `HAL_ERROR` |

也就是说 —— **HAL 自己认为"现在可以写"，硬件也同意。** 这一写是真实生效的，不是被静默忽略。

---

## 4. 写下去之后为什么芯片会死

`PWR_CR3` 的 bit[2:0] 是**内核供电来源选择**。HAL 给了三个名字（[stm32h7xx_hal_pwr_ex.h:235-245](Drivers/STM32H7xx_HAL_Driver/Inc/stm32h7xx_hal_pwr_ex.h#L235)）：

| 宏 | 值 | 二进制 | HAL 注释 |
|---|---|---|---|
| `PWR_LDO_SUPPLY` | `PWR_CR3_LDOEN` = `0x02` | `010` | *Core domains supplied from the LDO* |
| `PWR_DIRECT_SMPS_SUPPLY` | `PWR_CR3_SMPSEN` = `0x04` | `100` | *Core domains supplied from the SMPS only* |
| `PWR_EXTERNAL_SOURCE_SUPPLY` | `PWR_CR3_BYPASS` = `0x01` | `001` | 外部直接供电 |

**注意 `PWR_DIRECT_SMPS_SUPPLY` 注释里的 "only"。** 它的语义不是"打开 SMPS"，而是"**内核只由 SMPS 供电**"—— 也就是说 **LDO 被关掉**。`MODIFY_REG` 会把掩码内的位全清再写值，所以写 SMPSEN 的同时 **LDOEN 被清零**。

### 链条

```
main.c:169  SystemClock_Config()
              └─ main.c:249  HAL_PWREx_ConfigSupply(PWR_DIRECT_SMPS_SUPPLY)
                    └─ MODIFY_REG(PWR->CR3, 0x3F, 0x04)
                          └─ PWR_CR3:  0x05000046  →  0x05000044
                                        bit1 LDOEN: 1 → 0
                                             ↓
                          LDO 关闭 → 内核的 VCORE 失去来源
                                             ↓
                    CPU 停 + 调试逻辑停（它们在同一个电源域）
                                             ↓
                          DAP 不应答 → cannot read IDR
```

### 时机：为什么"一上电就死"

`SystemClock_Config()` 在 [main.c:169](CM7/Core/Src/main.c#L169) 被调用，**在任何外设初始化之前**：

```
main.c:161  HAL_Init();
main.c:169  SystemClock_Config();     ← ★ 249 行在这里
main.c:198  MX_GPIO_Init();           ← 连 GPIO 都还没配
main.c:199  MX_DMA_Init();
main.c:200  MX_DMA2D_Init();
main.c:201  MX_FMC_Init();
main.c:206  sdram_init();
main.c:214  ETH_Init();
main.c:223  osKernelStart();
```

**所以崩溃点在开机的头几毫秒内、在任何外设（LED、串口、屏幕）之前。** 这解释了为什么现象是"什么反应都没有"：不是程序跑飞了，是**它根本没跑到能给你反馈的那一步**。

### 为什么"LDO 才是对的"

| 证据 | 说明 |
|---|---|
| [`main.c.orig:212`](CM7/Core/Src/main.c.orig#L212)（09-28 的备份）写的是 `HAL_PWR_LDO_SUPPLY` | **这块板用 LDO 单独供电是跑通过的** |
| 芯片复位默认 `PWR_CR3 = 0x46`（SMPSEN 和 LDOEN 都开） | LDO 通路在复位时就是开着的 |
| [main.c:249](CM7/Core/Src/main.c#L249)（10-09 18:58）改成 `PWR_DIRECT_SMPS_SUPPLY` | 新换上来的**未验证**选项 |

**⇒ LDO 是被验证过的选项；`DIRECT_SMPS` 是那天新换上去、从没在这块板上跑通过的选项。** 修改时间也和"烧录开始失败"的时间重合。

---

## 5. 为什么"按复位"救不回来，为什么 BOOT0=1 能救

### 按复位为什么没用

`BOOT0 = 0` 时复位 → CPU 从 `0x08000000` 开始跑 → **又跑到 `main.c:249`** → 又把 `LDOEN` 清零 → 又死。**复位会把固件从头再跑一遍，所以它永远治不好自己。**

> 这也解释了为什么之前"换速率 / 换复位策略 / 连 JTAG"那一整套都没用 —— 那些手段都在试图**穿过**一个已经断电的芯片。

### BOOT0=1 为什么能救

`BOOT0 = 1` + 复位 → 芯片执行 **ST 出厂 ROM bootloader**，你的固件**一行都不执行** → `PWR_CR3` 保持复位默认 `0x46` → SWD 正常。

这是整个 BUG 最有用的一个性质：**它给你留了一条不需要调试口也能进去的路。**

> 顺带一句：`PWR_CR3` 被写过之后，HAL 认为配置已锁（`0x04 & 0x07 = 0x04 ≠ 0x06` → 守卫生效 → 后续调用返回 `HAL_ERROR`）。**复位会不会把 `PWR_CR3` 还原成 `0x46`，本文不下断言** —— 保险做法是**断电重上电**，别只按复位。

---

## 6. 为什么这个 bug 特别安静

它一路躲过了所有常规排查，原因有三层：

| # | 静默点 | 说明 |
|---|---|---|
| 1 | **返回值没人接** | `HAL_PWREx_ConfigSupply` 返回 `HAL_StatusTypeDef`，[main.c:249](CM7/Core/Src/main.c#L249) 直接丢弃。函数内部的"锁住了改不了"路径也是 `return HAL_ERROR`，两个失败路径全都无声 |
| 2 | **编译、链接零警告** | 宏名合法、参数合法、链接符号齐全。`make` 一声不响 |
| 3 | **这个名字看起来更"高级"** | `SMPS`（开关电源）听起来比 `LDO`（线性稳压）更先进，容易让人觉得"改成 SMPS 是优化"。但 HAL 的语义是 `SMPS only` —— **把 LDO 关掉**，不是在已有基础上加一层 |

**而且它的表现极具误导性**：报错发生在调试器上（`cannot read IDR`），凶手指向的却是"调试器/接线/芯片"。**故障现象的位置和根因的位置完全无关** —— 这是嵌入式里最难查的一类。

> **可复用的判据**：当"换个工具/换根线/换个速率"都改不了症状时，试着反过来问 —— **"什么样的代码错误能造成这个现象？"** §1 那张表就是把这句话变成可执行的筛选条件。

---

## 7. 改法与烧录顺序

### 要改的一行

[main.c:249](CM7/Core/Src/main.c#L249)，一个词：

```c
   HAL_PWREx_ConfigSupply(PWR_DIRECT_SMPS_SUPPLY);
→  HAL_PWREx_ConfigSupply(PWR_LDO_SUPPLY);
```

改完 `PWR_CR3` 的变化：

```
0x05000046  (复位默认：SMPSEN=1 LDOEN=1)
0x05000044  （错的那版：SMPSEN=1 LDOEN=0）  ← 现在跑的是这个
0x05000042  （改回 LDO：SMPSEN=0 LDOEN=1）  ← 和 09-28 那版一致，跑通过
```

### 烧录顺序（关键：不按这个顺序你烧不进去）

因为**固件一跑起来调试口就死**，所以必须让芯片**先停在 bootloader 里**再烧：

| 步 | 动作 | 为什么 |
|---|---|---|
| 1 | **BOOT0 = 1**，按复位 | 芯片停在 ROM bootloader，固件不跑 → **SWD 活着** |
| 2 | 烧录 | 此时调试口可用 |
| 3 | **BOOT0 跳回 0**，按复位 | 这次固件用 LDO，调试口应该活着 |
| 4 | 再跑一次 OpenOCD | 见 §8 判据 |

> **⚠️ 第 3 步别忘。** BOOT0 停在 1 的话，下次上电芯片会蹲在 ROM bootloader 里等你烧写，**你的程序永远不会跑**。

---

## 8. 判据（可证伪）

改完、按上面顺序烧完，跑：

```bash
openocd -s <scripts> -f interface/cmsis-dap.cfg -f target/stm32h7x_dual_bank.cfg \
        -c "init" -c "shutdown"
```

| 结果 | 结论 |
|---|---|
| 出现 `Info : SWD DPIDR 0x6ba02477` + `Cortex-M7 r1p1 processor detected` | ✅ **确诊**：原因就是这一行 |
| 仍然 `cannot read IDR` | ❌ **本文结论被推翻** → 回到 §1 重新筛，下一个候选是"芯片被不停复位"（`IWDG1` 在 `.ioc` 里标着 `I`，值得查） |

**还想更直接的证法（可选，需谨慎）**：趁 SWD 通着（BOOT0=1），手动模拟固件那一写，再连一次看口还在不在：

```bash
# 在 openocd 的 init 之后：
mdw 0x5802480C          # 先看，应为 0x05000046
mww 0x5802480C 0x05000044   # 就是固件那一写的值
mdw 0x5802480C          # 回读，确认写进去了
shutdown
# 然后重新跑一次 openocd：口没了 → 就是它
```

> **这会在运行中改一颗电源寄存器。** 出岔子的代价是芯片失去内核供电 —— 恢复办法是**断电重上电**（不是按复位）。

---

## 9. 复发陷阱：CubeMX 会把它写回去

**这是修完之后最容易再踩的一脚。**

| 事实 | 证据 |
|---|---|
| `HAL_PWREx_ConfigSupply(...)` 在 `SystemClock_Config()` 里 | [main.c:242](CM7/Core/Src/main.c#L242) 起的整个函数 |
| 该函数里**没有任何 `USER CODE` 标记** | `USER CODE` 块都在 `main()` 的 `while(1)` 尾部，离这里很远 |
| **⇒ 整个 `SystemClock_Config()` 都是 CubeMX 管理区** | 下次 Generate 会**整段重写** |
| `.ioc` 里**没有任何供电相关的键** | 全文搜 `supply` 的唯一命中是 `PA8.GPIO_PuPd=GPIO_PULLDOWN`（"PU**LLDO**WN" 里恰好含 "LDO"，误命中） |

**⇒ 这个值来自 CubeMX 的界面设置/默认值，不在 `.ioc` 里明写。下次 Generate 一定会把它写回 `PWR_DIRECT_SMPS_SUPPLY`。**

两个办法，选一个：

| 办法 | 做法 | 代价 |
|---|---|---|
| **A（推荐）** | 在 CubeMX 里找到供电设置，改成 LDO，让 `.ioc` 记住它 | 要翻 CubeMX 界面（H7 一般在 Clock Configuration 页的供电/VOS 区域，或 PWR 的配置项里） |
| **B（保底）** | **每次 Generate 之后**手动改回这一行 | 靠记性；建议在 `.ioc` 旁边贴个备注 |

> 参考：`main.c.orig`（09-28）保留了 **LDO** 那一版，可以随时拿来对照。**没有 git 的情况下，`.orig` 就是你的 diff。**

---

## 10. 源码索引

| 结论 | 证据（文件:行） |
|---|---|
| 出问题的那一行 | [main.c:249](CM7/Core/Src/main.c#L249) |
| 09-28 那版是 LDO | [main.c.orig:212](CM7/Core/Src/main.c.orig#L212) |
| 守卫的 `#if/#else` 结构 | [stm32h7xx_hal_pwr_ex.c:325](Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_pwr_ex.c#L325) |
| 真正写寄存器的那句 | [stm32h7xx_hal_pwr_ex.c:347](Drivers/STM32H7xx_HAL_Driver/Src/stm32h7xx_hal_pwr_ex.c#L347) |
| `PWR_FLAG_SCUEN` 在**不生效**的 `#else` 里 | [stm32h7xx_hal_pwr.h:175-179](Drivers/STM32H7xx_HAL_Driver/Inc/stm32h7xx_hal_pwr.h#L175) |
| `SMPS` 已定义 | [stm32h757xx.h:218](Drivers/CMSIS/Device/ST/STM32H7xx/Include/stm32h757xx.h#L218) |
| `SMPSEN`/`LDOEN`/`BYPASS` 位 | [stm32h757xx.h:18203-18210](Drivers/CMSIS/Device/ST/STM32H7xx/Include/stm32h757xx.h#L18203) |
| `PWR_SUPPLY_CONFIG_MASK` 两个分支 | [stm32h7xx_hal_pwr_ex.h:247](Drivers/STM32H7xx_HAL_Driver/Inc/stm32h7xx_hal_pwr_ex.h#L247) |
| 三个供电源常量 + HAL 注释 | [stm32h7xx_hal_pwr_ex.h:235-245](Drivers/STM32H7xx_HAL_Driver/Inc/stm32h7xx_hal_pwr_ex.h#L235) |
| `PWR_TypeDef` 偏移（**CR3 @0x0C**、D3CR @0x18） | [stm32h757xx.h:1351 附近的 PWR_TypeDef](Drivers/CMSIS/Device/ST/STM32H7xx/Include/stm32h757xx.h#L1340) |
| `PWR_BASE = D3_AHB1PERIPH_BASE + 0x4800` ⇒ **`PWR_CR3 = 0x5802480C`** | [stm32h757xx.h:2404](Drivers/CMSIS/Device/ST/STM32H7xx/Include/stm32h757xx.h#L2404) |
| VOS 在 `PWR->D3CR`（bit[15:14]），不在 `PWR_CR1` | [stm32h757xx.h:18285](Drivers/CMSIS/Device/ST/STM32H7xx/Include/stm32h757xx.h#L18285) + [stm32h7xx_hal_pwr.h:292](Drivers/STM32H7xx_HAL_Driver/Inc/stm32h7xx_hal_pwr.h#L292) |
| `PWR_FLAG_VOSRDY` 读的是 `PWR->D3CR` | [stm32h7xx_hal_pwr.h:382](Drivers/STM32H7xx_HAL_Driver/Inc/stm32h7xx_hal_pwr.h#L382) |
| `SystemClock_Config` 在任何外设初始化之前 | [main.c:169](CM7/Core/Src/main.c#L169) vs [main.c:198](CM7/Core/Src/main.c#L198) |

### 夹子实测值（BOOT0=1，ROM bootloader 运行时）

| 地址 | 寄存器 | 值 | 解读 |
|---|---|---|---|
| `0x58024800` | `PWR_CR1` | `0xF000C000` | — |
| `0x58024804` | `PWR_CSR1` | `0x00004000` | `ACTVOS`(bit[15:14]) = `0b01`；`ACTVOSRDY`(bit13) = 0 |
| `0x58024808` | `PWR_CR2` | `0x00000000` | — |
| `0x5802480C` | **`PWR_CR3`** | **`0x05000046`** | **SMPSEN=1, LDOEN=1**（能工作的状态） |
| `0x58024810` | `PWR_CPUCR` | `0x00000000` | — |

---

## 附：一句话记住它

> **`PWR_DIRECT_SMPS_SUPPLY` 里的 `only` 是关键字 —— 它不是在"打开 SMPS"，而是在"关掉 LDO"。**
>
> **而关掉 LDO 之后，内核靠什么供电，是硬件问题 —— 代码这边能确定的只有一件事：以前用 LDO 跑通过，这次换成了没跑通过的。**
