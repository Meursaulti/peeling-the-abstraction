# 项目概述

CS61C Project 3 主要实现一个支持 RV32I 指令子集的 RISC-V CPU。

项目从单周期 CPU 开始，通过模块化设计逐步搭建完整数据通路与控制单元，而后完成二级流水线改造，并通过课程提供的测试框架验证实现正确性。

---

# 实现流程

## 单周期 CPU

通过 Logisim 模块化搭建各独立部件并利用课程提供的测试框架验证功能正确性，而后将各模块集成为完整数据通路，并通过 ROM 实现控制单元。

主要实现模块包括：

* PC
* Register File
* ALU
* Immediate Generator
* Branch Comparator
* Data Memory
* Instruction Memory
* ROM Control Unit

最终实现课程要求的 RV32I 指令子集。

---

## 指令集验证

在 CPU 功能完成后，通过编写汇编程序验证各类指令实现是否符合预期。

主要覆盖：

* R-Type
* I-Type
* S-Type
* B-Type
* U-Type
* J-Type

并结合课程提供的测试框架进行自动化测试。

验证内容包括：

* 算术运算
* 逻辑运算
* 数据访存
* 条件分支
* 无条件跳转
* 立即数处理

确保各类指令均能按照 RV32I ISA 规范正确执行。

---

## 二级流水线改造

在单周期 CPU 的基础上完成二级流水线改造。

流水线划分如下：

```text
IF | EX
```

其中：

### IF 阶段

* PC 更新
* 指令获取

### EX 阶段

* 指令译码
* 寄存器读取
* ALU 运算
* 数据访存
* 写回

项目最终采用课程要求的 Flush 方案解决控制冒险。

---

# 关于控制冒险的一些思考

在完成项目后，我对控制冒险的处理方式有一些额外思考。

课程实现采用的是：

```text
Branch Taken
    ↓
Flush IF阶段指令
    ↓
产生一个 Bubble
```

这种方式简单可靠，但会带来额外 CPI 开销。

## 我的想法

由于本项目采用的是：

```text
IF | EX
```

二级流水线。

先明确一个概念，在本实现中 DMEM 的延迟长于 IMEM。因为 DMEM 不仅需要完成存储器访问，还需要经过部分加载器逻辑，因此整体实现相较于 IMEM 更复杂，延迟也更高。

流水线运行过程中，IF 阶段主要完成：

```text
PC → IMEM → 流水线寄存器
```

而 EX 阶段需要经过：

```text
Register File
→ ALU
→ DMEM
→ Write Back
```

因此 EX 阶段的关键路径明显长于 IF 阶段。

换句话说，IF 阶段实际上存在一定的时序余量。

## 设想方案

当执行 B-Type 指令时：

1. ALU 在 EX 阶段计算出 Branch Target
2. 将 Branch Target 直接转发
3. 通过 MUX 将 Branch Target 转发至流水线 PC 寄存器的输入以及 PC + 4 加法器的输入
4. 同时将 Branch Target 转发至 IMEM 地址端

虽说看似增加了不少多路复用器，但从下图与课程原版的对比来看，本质上只是删除了一个复用器并增加了一个复用器。

如图：

<img width="880" height="971" alt="image" src="https://github.com/user-attachments/assets/a8c5efc9-ce92-4ca4-b341-baa9c585e944" />

此时距离 EX 阶段结束仍存在 DMEM 等后续路径。

根据：

```text
DMEM 延迟 > IMEM 延迟
```

这一关系，如果能够在 ALU 计算完成后立刻更新取指地址并启动下一次取指，那么理论上可以在 EX 阶段结束之前完成正确指令的获取。

## 说明

以上内容仅为个人推导。

对应的数据通路已经完成设计，并完成了电路层面的搭建，但由于时间、精力以及个人水平所限，我并未进一步完成该方案的功能测试与时序验证。

---

# 部分电路图展示

## 寄存器文件

<img width="1958" height="1244" alt="屏幕截图 2026-05-28 213956" src="https://github.com/user-attachments/assets/08a3c9ac-44bc-40c9-8455-df2a236fc31d" />

## 立即数生成器

<img width="1403" height="671" alt="屏幕截图 2026-05-28 214015" src="https://github.com/user-attachments/assets/8e120fa3-d361-4e00-8c7f-0384d5816a64" />

## 部分加载器

<img width="863" height="573" alt="屏幕截图 2026-05-28 214027" src="https://github.com/user-attachments/assets/414db242-218c-4521-b6d5-e5c39d4edd68" />

## 二级流水线数据通路（课程版本）

<img width="1960" height="977" alt="image" src="https://github.com/user-attachments/assets/2a8c0186-992b-44a8-98a9-36fb689a3ba5" />

## 二级流水线数据通路（提前取值版本）

<img width="1874" height="863" alt="屏幕截图 2026-06-08 151603" src="https://github.com/user-attachments/assets/51eabb9c-17c9-448c-8e02-9dc679728518" />

---

以上内容仅记录项目实现过程与个人思考。
