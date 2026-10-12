# EBAZ4205 Zynq —— LED 控制 + 串口命令行（letter-shell）

基于 **EBAZ4205**（Zynq-7010）开发板的 PS + PL 协同示例：

> 在电脑串口终端里敲命令，**当场**控制板上的红色 LED —— 闪烁、调亮度，不用重新烧录。

- **PS（ARM Cortex-A9）**：移植 [letter-shell](https://github.com/NevermindZZT/letter-shell) 3.2.4，提供串口命令行
- **PL（FPGA）**：`led_ctl.v` 实现 1 kHz PWM 调光 + 精确周期闪烁
- **桥梁**：AXI GPIO，PS 写寄存器 → PL 拉引脚

---

## 一、硬件平台

| 项 | 说明 |
| --- | --- |
| 开发板 | EBAZ4205（原比特币矿机控制板） |
| 主芯片 | **XC7Z010-1CLG400I**（PS: 双核 Cortex-A9 + PL: Artix-7） |
| 内存 | 256 MB DDR3（16 bit） |
| 受控外设 | 板载**红色 LED** → FPGA 引脚 **W14**（LVCMOS33） |
| 串口 | PS **UART1 → MIO 24/25**，`115200 8N1` |
| PL 时钟 | PS `FCLK_CLK0` = **50 MHz** |
| PL 复位 | PS `FCLK_RESET0_N`（低有效） |

---

## 二、功能：可用命令

串口连上（115200 8N1）后能看到提示符 `letter:/$`，可敲：

| 命令 | 效果 |
| --- | --- |
| `led 0` | 灭 |
| `led 1` | 闪烁，**周期 1 秒**（1.0 Hz，亮 0.5s / 灭 0.5s） |
| `led 2` | 闪烁，**周期 2 秒**（0.5 Hz，亮 1.0s / 灭 1.0s） |
| `led 3` | 常亮，**亮度 1 档**（占空比约 33%） |
| `led 4` | 常亮，**亮度 2 档**（占空比约 67%） |
| `led 5` | 常亮，**亮度 3 档**（占空比 100%，最亮） |
| `help` | 列出所有已注册命令 |
| `Tab` | 命令自动补全 |

> 命令名和参数之间**必须有空格**；命令名**区分大小写**。

---

## 三、目录结构

```text
.
├── docs/
│   └── traindoc.pdf              设计说明文档
├── test/                         Vivado 工程
│   ├── test.xpr                  Vivado 工程文件
│   ├── test.srcs/
│   │   ├── sources_1/
│   │   │   ├── new/
│   │   │   │   └── led_ctl.v     ★ PL：LED 控制模块（PWM + 闪烁）
│   │   │   └── bd/design01/
│   │   │       └── design01.bd   ★ Block Design（PS + AXI GPIO + led_ctl）
│   │   └── constrs_1/new/
│   │       └── led_ctl.xdc       引脚约束：led_red → W14
│   └── test.sdk/
│       └── shell/                Xilinx SDK 应用工程
│           └── src/
│               ├── helloworld.c  ★ PS：shell 主程序 + led 命令
│               ├── lscript.ld    链接脚本（含 shellCommand 段）
│               ├── shell.c/.h    letter-shell 3.2.4 库
│               ├── shell_cfg.h   库配置（裸机模式）
│               ├── shell_ext.*   库扩展（参数解析）
│               └── platform.c/.h Xilinx 模板
└── .gitignore                    排除 Vivado / SDK 生成物
```

**★ 标记的 4 个文件是本项目自己写的**，其余为 Xilinx 生成或第三方库。

---

## 四、系统结构

```text
  PC 串口终端 (SecureCRT / Putty, 115200 8N1)
        │  敲入:  l e d   3  ⏎
        ▼
┌──────────────────────── PS (ARM Cortex-A9) ────────────────────────┐
│  UART1 (MIO24/25)                                                  │
│      ↕  shellRead() / shellWrite()          ← helloworld.c 实现     │
│  letter-shell 3.2.4                                                │
│      shellTask → 分词 → 查表 → 调用命令函数                          │
│      命令表来自 shellCommand 段 ← lscript.ld 提供首尾符号            │
│      ↓                                                             │
│  funcLed()  →  往 0x41200000 写 3                                  │
└────────────────────────────┬───────────────────────────────────────┘
                             │ AXI (M_AXI_GP0 → Interconnect → AXI GPIO)
┌────────────────────────────▼───────────────────────────────────────┐
│  PL (FPGA)                                                         │
│  AXI GPIO (0x41200000)  →  ctrl[31:0]                              │
│  led_ctl.v              →  按 ctrl[2:0] 产生 PWM / 闪烁 → led_red   │
└────────────────────────────┬───────────────────────────────────────┘
                             │ 引脚 W14（led_ctl.xdc 约束）
                             ▼
                         🔴 板载红色 LED
```

---

## 五、控制字约定（PL 侧寄存器）

AXI GPIO 基地址 **`0x41200000`**，32 bit 全输出。`ctrl[2:0]` 选择功能：

| `ctrl[2:0]` | 功能 | 说明 |
| --- | --- | --- |
| `0` | 灭 | 恒定输出 0 |
| `1` | 1 秒周期闪烁 | 计数器精确分频：半周期 25,000,000 @50MHz |
| `2` | 2 秒周期闪烁 | 半周期 50,000,000 @50MHz |
| `3` | 亮度 1 档 | PWM 占空比 ≈ 33%（16666 / 50000） |
| `4` | 亮度 2 档 | PWM 占空比 ≈ 67%（33332 / 50000） |
| `5` | 亮度 3 档 | PWM 占空比 100% |
| `6`、`7` | 保留 | 按"灭"处理 |

`ctrl[31:3]` 保留未用。

**PWM 参数**：频率 1 kHz（人眼看不到闪烁），周期 50,000 个时钟。

---

## 六、Block Design 结构

`design01.bd` 包含 5 个单元：

| 单元 | 作用 |
| --- | --- |
| `processing_system7_0` | Zynq PS（DDR3、UART1、FCLK_CLK0 50MHz、M_AXI_GP0） |
| `axi_interconnect_0` | AXI 互联（内含 AXI3→AXI4-Lite 协议转换器） |
| `axi_gpio_0` | **AXI-Lite 从机**：把 AXI 事务变成一根 32 位输出线 |
| `proc_sys_reset_0` | 把 PS 复位同步到 PL 时钟域，产生各类 `aresetn` |
| `led_ctl_0` | 自定义 LED 控制模块（module reference） |

**关键连线：**

```text
FCLK_CLK0          →  interconnect(ACLK/S00_ACLK/M00_ACLK)、axi_gpio(s_axi_aclk)、
                      proc_sys_reset(slowest_sync_clk)、led_ctl(clk)、M_AXI_GP0_ACLK
FCLK_RESET0_N      →  proc_sys_reset(ext_reset_in)
proc_sys_reset/interconnect_aresetn → interconnect(ARESETN)
proc_sys_reset/peripheral_aresetn   → interconnect(S00/M00_ARESETN)、
                                      axi_gpio(s_axi_aresetn)、led_ctl(rst_n)
M_AXI_GP0 → interconnect(S00_AXI) → M00_AXI → axi_gpio(S_AXI)
axi_gpio/gpio_io_o[31:0] → led_ctl(ctrl[31:0])
led_ctl/led_red          → 顶层端口 led_red
```

---

## 七、开发环境

| 工具 | 版本 |
| --- | --- |
| Vivado | **2018.3** |
| Xilinx SDK | **2018.3** |
| 目标器件 | `xc7z010clg400-1` |
| letter-shell | 3.2.4 |

---

## 八、如何重建 / 烧录运行

```text
① 用 Vivado 2018.3 打开          test/test.xpr

② 生成 BD 产物
   右键 design01.bd → Generate Output Products
   右键 design01.bd → Create HDL Wrapper

③ 生成比特流
   Run Synthesis → Run Implementation → Generate Bitstream

④ 导出硬件
   File → Export → Export Hardware（勾选 Include bitstream）
   → Launch SDK

⑤ SDK 里编译
   打开 test.sdk/shell 工程 → Project → Build Project

⑥ 上板
   Xilinx → Program FPGA                    ← 必须先烧 PL！
   右键 shell → Run As → Launch on Hardware

⑦ 打开串口终端
   COM 口：板子的 UART，参数 115200 8N1，无流控
   应看到：  letter shell 3.2.4 ready
             letter:/$

⑧ 敲命令
   led 3        ← 红灯变成 1 档亮度
```

> ⚠️ **必须先 Program FPGA**。AXI GPIO 在 PL 里，PL 没配置的话 PS 写 0x41200000
> 不会产生任何效果（灯不会动，但命令会正常回显）。

---

## 九、关键设计说明

### 9.1 PL 侧：`led_ctl.v`

- **闪烁**：用独立计数器做**精确分频**（不是取计数器某一位的近似值）
  - 1 秒档：数到 `CLK_HZ/2` 翻转 → 半周期 0.5 s
  - 2 秒档：数到 `CLK_HZ` 翻转 → 半周期 1.0 s
- **亮度**：1 kHz PWM，共用同一个计数器，两档之间切换无毛刺
- **输出寄存一拍**：避免组合逻辑毛刺直接打到引脚
- 时钟频率是 `parameter CLK_HZ`，换时钟只需改这一个参数

### 9.2 PS 侧：letter-shell 移植

只做了 **4 件事**：

| # | 做什么 | 在哪 |
| --- | --- | --- |
| 1 | 实现读写函数（接 PS UART1） | `helloworld.c` 的 `shellRead` / `shellWrite` |
| 2 | 初始化并驱动 shell | `main()`：`shellInit()` + `while(1) shellTask()` |
| 3 | 导出自己的命令 | `SHELL_EXPORT_CMD(..., led, funcLed, ...)` |
| 4 | 配置库为裸机模式 | `shell_cfg.h`：`SHELL_TASK_WHILE=0`、`SHELL_MAX_NUMBER=1` |

**读写函数的原型**必须与库匹配（letter-shell 3.1+ 是批量接口）：

```c
signed short shellRead (char *data, unsigned short len);
signed short shellWrite(char *data, unsigned short len);
```

**命令自动注册机制**：`SHELL_EXPORT_CMD` 宏把命令结构体放进一个名为
`shellCommand` 的自定义段，`shellInit()` 靠两个链接符号遍历该段得到命令表：

```c
extern const unsigned int _shell_command_start;
extern const unsigned int _shell_command_end;
```

### 9.3 为什么必须改 `lscript.ld`

`shellCommand` 段需要在链接脚本里**显式声明首尾**，否则：

- 不写 → 链接报错 `undefined reference to '_shell_command_start'`
- 写了但不用 `KEEP` → 开启 `--gc-sections` 时命令会被当垃圾清掉

所以 `test.sdk/shell/src/lscript.ld` 的 `.text` 段里加了这三行：

```ld
_shell_command_start = .;
KEEP (*(shellCommand))
_shell_command_end   = .;
```

> ⚠️ 链接脚本里**只能用 `/* */` 注释，不能用 `//`**，且建议**只用英文**——
> 用 `//` 或中文会报 `cannot find //` / `ignoring invalid character`。

---

## 十、常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 串口无任何输出 | COM 口/波特率不对，或程序没跑 | 确认 115200 8N1，确认已 Run |
| 敲 `led` 提示 command not found | 链接脚本的 `KEEP` 没生效 | 检查 `lscript.ld` 三行 |
| 命令有回显但灯不动 | PL 没烧 bitstream | 先 `Program FPGA` |
| 串口乱码 | 波特率不是 115200 | 改终端设置 |
| 按 Tab 双击无帮助 | 未实现 `SHELL_GET_TICK()` | 在 `shell_cfg.h` 里定义该宏 |

---

## 十一、参考资料

- letter-shell 官方仓库：<https://github.com/NevermindZZT/letter-shell>
- EBAZ4205 资料：<https://github.com/xjtuecho/EBAZ4205>
- 设计说明文档：[`docs/traindoc.pdf`](docs/traindoc.pdf)