---
fpga-hardware-design-and-review-guide
description: >
  Personal FPGA hardware design guide based on real project experience. 
  Covers pipeline design, timing optimization, SystemVerilog coding patterns, 
  and practical debugging techniques. Use this skill when:
  (1) designing FPGA modules with timing constraints,
  (2) implementing video processing or data path designs,
  (3) optimizing for resource utilization and timing closure,
  (4) reviewing RTL code for hardware implementation,
  (5) debugging synthesis and implementation issues.
license: MIT
compatibility: Works with Claude Code, Cursor, and AI coding assistants.
metadata:
  author: peng
  version: "1.0"
  based_on: Real project experience (RGB-to-YUV converter, video processing)
allowed-tools: Read Write Edit Bash
---

# FPGA Hardware Design Guide

基于实际项目经验的FPGA硬件设计指南。

## 核心设计理念

### 1. 流水线架构优先

在处理高速数据流（如视频、网络包）时，采用多级流水线设计：
- **单级处理**：组合逻辑延迟过大，容易出现时序违例
- **多级流水线**：每级插入寄存器，分散延迟，提高时钟频率
- **典型应用**：RGB转YUV、图像滤波、协议解析

**实际案例**：RGB转YUV转换器采用5级流水线
- Stage 0: 输入寄存（同步输入信号）
- Stage 1: 乘法运算（系数*像素值）
- Stage 2: 部分累加（R*coef_r + G*coef_g）
- Stage 3: 最终累加（+ B*coef_b）
- Stage 4: 移位和饱和处理（结果截断到8bit）

### 2. 位宽管理的艺术

**计算位宽的原则**：
```
乘法位宽 = 输入位宽 + 系数位宽 + 1（符号位）
累加位宽 = 乘法位宽 + log2(累加个数) + 1（保护位）
```

**经验法则**：
- 8bit无符号 * 9bit有符号系数 = 18bit有符号结果
- 3个18bit数相加 = 20bit（留2bit保护位防止溢出）
- 右移8位后 = 8bit最终结果

### 3. 同步设计的铁律

**必须遵守的规则**：
1. **所有触发器使用同一时钟域**（除非明确需要CDC）
2. **同步复位优于异步复位**（避免亚稳态传播）
3. **输入信号必须打两拍**（跨时钟域或外部输入）
4. **组合逻辑输出必须寄存**（避免毛刺传播）

**实际教训**：
- 异步复位在时钟不稳定时会导致不可预测行为
- 未寄存的组合输出在布局布线后可能产生毛刺
- 跨时钟域信号直接用会导致亚稳态

## 时序收敛实战技巧

### 延迟分析与优化

**识别关键路径**：
1. 查看综合报告中的 `Worst Negative Slack (WNS)`
2. 分析 `Total Negative Slack (TNS)` 分布
3. 定位延迟最大的逻辑级数

**优化策略**：
1. **插入流水线寄存器**（最有效）
   - 在组合逻辑中间插入FF
   - 每级延迟 < 目标时钟周期的70%
   
2. **逻辑重定时**（Retiming）
   - 使用 `set_property RETIMING true` 
   - 让工具自动移动寄存器位置
   
3. **关键信号优化**
   - 对关键路径使用 `set_property HIGH_PRIORITY true`
   - 手动布局关键模块 `set_property LOC ...`

**实际数据**：
- 原始设计：关键路径15ns，目标10ns（不满足）
- 插入2级流水线：关键路径7ns（满足+余量30%）
- 延迟代价：2个时钟周期（可接受）

## 资源优化策略

### LUT优化

**减少LUT使用的方法**：
1. **使用case语句代替if-else链**（综合为LUT更高效）
2. **避免复杂的三目运算符嵌套**
3. **利用DSP Slice代替LUT实现乘法**

**实例对比**：
```systemverilog
// 低效：多层嵌套if
if (condition1) out = a;
else if (condition2) out = b;
else if (condition3) out = c;
// 使用 ~20 LUTs

// 高效：case语句
case ({condition1, condition2, condition3})
    3'b100: out = a;
    3'b010: out = b;
    3'b001: out = c;
    default: out = d;
endcase
// 使用 ~8 LUTs
```

### BRAM使用技巧

**何时使用BRAM**：
- 存储深度 > 16（通常）
- 需要双端口访问
- 大容量查找表（>1KB）

**何时使用分布式RAM**：
- 小容量存储（<16深度）
- 需要异步读取
- 节省BRAM资源

**代码示例**：
```systemverilog
// 自动推断为BRAM（36Kb块）
reg [7:0] mem [0:1023];  // 8Kbits
always @(posedge clk) begin
    if (we) mem[addr] <= din;
    dout <= mem[addr];  // 同步读
end

// 小容量自动使用LUTRAM
reg [7:0] small_mem [0:15];  // 128bits
```

### DSP Slice优化

**充分利用DSP48E1**：
- 25×18乘法器（支持有符号/无符号）
- 48位累加器
- 预加器（用于对称FIR滤波）

**避免DSP浪费**：
- 不要用小位宽乘法（<8bit），LUT更高效
- 级联DSP时利用专用走线（ACIN/ACOUT）
- 使用CE和SCLR控制节省功耗

## 调试与验证方法

### 仿真策略

**三级验证体系**：
1. **行为级仿真**（前综合）
   - 验证算法正确性
   - 使用理想延迟模型
   
2. **综合后仿真**（Post-Synthesis）
   - 验证综合结果功能正确
   - 检查时序粗略估计
   
3. **实现后仿真**（Post-Implementation）
   - 包含实际布线延迟
   - 最接近真实硬件

**测试平台编写要点**：
```systemverilog
// 1. 自检查测试
initial begin
    // 施加激励
    apply_stimulus();
    
    // 等待处理
    repeat(10) @(posedge clk);
    
    // 检查结果
    if (dout !== expected) begin
        $error("Test failed! Expected %h, got %h", expected, dout);
        $finish;
    end
    $display("Test passed!");
end

// 2. 覆盖率检查
covergroup cg @(posedge clk);
    coverpoint state {
        bins idle = {IDLE};
        bins busy = {BUSY};
        bins done = {DONE};
    }
endgroup
```

### 板上调试技巧

**使用ILA（Integrated Logic Analyzer）**：
1. 标记关键信号为 `mark_debug`
2. 设置触发条件（如错误标志、特定状态）
3. 捕获数据到Vivado分析

**使用VIO（Virtual Input/Output）**：
- 实时修改参数（如滤波器系数）
- 监控内部状态寄存器
- 无需重新编译即可调试

**实际调试案例**：
- 问题：YUV输出偶尔出现错误值
- 方法：ILA捕获乘法中间结果
- 发现：符号扩展错误导致高位溢出
- 解决：修正有符号数扩展逻辑

## 参考文档

**详细设计模式**：查看 `references/design-patterns.md`
- CDC同步器设计
- FIFO实现
- AXI-Stream接口

**常见问题排查**：查看 `references/troubleshooting.md`
- 时序违例诊断流程
- 亚稳态处理
- 资源冲突解决

**器件选型指南**：查看 `references/device-selection.md`
- 根据资源需求选型
- 封装和速度等级选择
- 成本优化建议

## 黄金法则

1. **先功能后优化** — 先让设计正确工作，再优化时序和资源
2. **早约束晚放松** — 早期严格时序约束，后期根据情况放宽
3. **寄存一切边界** — 模块输入输出必须寄存，避免时序耦合
4. **文档即代码** — 清晰的注释和文档比复杂的设计更重要
5. **测试驱动开发** — 先写测试平台，再实现功能

---

*本指南基于实际项目经验编写，持续更新中。*
