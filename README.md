# 芯片“锁死”后通过 BOOT0 进入系统 Bootloader 修复的原理与经验（STM32H7 / GD32）

## 背景

- 芯片：`<STM32H7 / GD32 / 其他>`
- 调试器：`<CMSIS-DAP / ST-Link / J-Link>`
- 工具链：`<xPack OpenOCD 0.12.0-7 / STM32CubeProgrammer / ...>`
- 系统：`<Windows / Linux / macOS>`

## 现象

使用 OpenOCD 下载程序时失败，终端进程被终止，退出代码为 1：

```text
终端进程“...\openocd.exe -s ...\scripts -f interface/cmsis-dap.cfg -f target/stm32h7x_dual_bank.cfg -c 'set DUAL_CORE 1' -c init -c 'reset halt' -c 'program ... verify' -c 'reset run' -c shutdown”已终止，退出代码: 1。
