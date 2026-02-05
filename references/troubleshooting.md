# 问题排查与解决

实际项目中遇到的问题及解决方案。

## 时序问题

### 问题1：关键路径延迟过大

**现象**：
- Vivado报告WNS（Worst Negative Slack）为负数
- 设计无法实现目标频率（如100MHz）

**诊断步骤**：
1. 查看时序报告，找到延迟最大的路径
2. 分析该路径上的逻辑级数
3. 检查是否有复杂的组合逻辑块

**解决方案**：

**方案A：插入流水线（推荐）**
```systemverilog
// 优化前：组合逻辑太长
assign result = (a * coef_a) + (b * coef_b) + (c * coef_c);

// 优化后：分两级完成
always @(posedge clk) begin
    sum_ab <= (a * coef_a) + (b * coef_b);  // 第1级
    result <= sum_ab + (c * coef_c);         // 第2级
end
```

**方案B：逻辑重定时**
```tcl
# 在XDC约束文件中添加
set_property RETIMING true [get_cells -hierarchical *]
```

**方案C：使用DSP Slice**
```systemverilog
// 让工具自动使用DSP
(* use_dsp = "yes" *) 
reg [17:0] mult_result;
```

---

### 问题2：建立时间违例（Setup Violation）

**原因**：
- 信号从源寄存器到目的寄存器传播太慢
- 时钟偏斜（Clock Skew）过大
- 组合逻辑延迟过大

**排查方法**：
```tcl
# 查看具体路径 report_timing -from [get_pins src_reg/C] -to [get_pins dst_reg/D] -setup
```

**解决策略**：
1. **减少逻辑级数**：将复杂运算拆分到多个周期
2. **优化布线**：手动布局关键模块
3. **降低时钟频率**：如果允许的话
4. **使用时钟偏斜优化**：`set_property CLOCK_DELAY_GROUP ...`

---

### 问题3：保持时间违例（Hold Violation）

**原因**：
- 信号传播太快，在时钟沿之前就到达
- 通常发生在同一时钟域内的短路径

**解决策略**：
1. **添加延迟单元**：
   ```tcl
   set_property DELAY_VALUE 4 [get_cells -hierarchical *delay_cell*]
   ```
   
2. **使用LUT作为延迟线**：
   ```systemverilog
   (* dont_touch = "true" *)
   wire delayed_signal;
   assign delayed_signal = ~ (~signal);  // 两个LUT延迟
   ```

---

## 逻辑错误

### 问题4：符号扩展错误

**现象**：
- 有符号数运算结果错误
- 高位填充不正确导致负数变正数

**案例分析**：RGB转YUV中的错误
```systemverilog
// 错误代码：
wire signed [17:0] result;
assign result = data * coef;  // data无符号，coef有符号
// 结果：高bit被错误填充

// 正确代码：
wire signed [8:0] data_signed;
assign data_signed = $signed({1'b0, data});  // 先转有符号
wire signed [17:0] result;
assign result = data_signed * coef;  // 再相乘
```

---

### 问题5：异步复位导致的亚稳态

**现象**：
- 系统偶尔出现随机错误
- 复位释放后状态机进入非法状态

**原因**：
- 异步复位信号在时钟沿附近释放
- 复位信号传播延迟不一致

**解决方案**：
```systemverilog
// 推荐：同步复位
always @(posedge clk) begin  // 注意：没有negedge rst_n
    if (!rst_n) begin
        state <= IDLE;
    end else begin
        state <= next_state;
    end
end

// 如果必须用异步复位，使用复位同步器：
reg rst_sync1, rst_sync2;
always @(posedge clk or negedge rst_async_n) begin
    if (!rst_async_n) begin
        rst_sync1 <= 1'b0;
        rst_sync2 <= 1'b0;
    end else begin
        rst_sync1 <= 1'b1;
        rst_sync2 <= rst_sync1;  // 同步后的复位
    end
end

wire rst_n_sync = rst_sync2;
```

---

### 问题6：跨时钟域数据丢失

**现象**：
- 多比特信号直接用2FF同步器
- 数据偶尔出错

**原因**：
- 2FF只能同步单bit信号
- 多bit信号每个bit延迟可能不同

**正确做法**：

**方法A：使用FIFO（推荐）**
```systemverilog
// 异步FIFO自动处理CDC
async_fifo #(
    .DATA_WIDTH(32),
    .DEPTH(16)
) u_fifo (
    .wr_clk(clk_a),
    .rd_clk(clk_b),
    // ... 其他端口
);
```

**方法B：握手协议**
```systemverilog
// 发送方
always @(posedge clk_a) begin
    if (ready_to_send) begin
        data_out <= data;
        valid_out <= 1'b1;
    end
    if (ack_in)  // 收到应答
        valid_out <= 1'b0;
end

// 接收方（经过2FF同步后）
always @(posedge clk_b) begin
    valid_sync <= {valid_sync[0], valid_out};
    if (valid_sync[1]) begin  // 检测到有效
        data_captured <= data_sync;  // 捕获数据
        ack_out <= 1'b1;  // 发送应答
    end
end
```

---

## 资源问题

### 问题7：LUT资源不足

**现象**：
- 综合报错：LUT utilization > 100%
- 或者无法实现（Place & Route失败）

**优化方法**：

**1. 简化组合逻辑**
```systemverilog
// 低效：复杂条件
assign out = (condition1 & condition2) | (condition3 & ~condition4) | ... ;

// 高效：使用case语句
case ({condition1, condition2, condition3, condition4})
    4'b1100: out = ...;
    // ...
endcase
```

**2. 使用BRAM代替分布式RAM**
```systemverilog
// 小数组自动用LUTRAM
reg [7:0] small_array [0:15];  // 用16个LUT

// 大数组强制用BRAM
(* ram_style = "block" *)
reg [7:0] large_array [0:1023];  // 用1个BRAM
```

**3. 共享资源**
```systemverilog
// 优化前：两个独立的乘法器
wire [15:0] result1 = a * b;
wire [15:0] result2 = c * d;

// 优化后：时分复用（如果允许延迟）
reg [15:0] mult_result;
reg [1:0] state;
always @(posedge clk) begin
    case (state)
        0: mult_result <= a * b;
        1: mult_result <= c * d;
    endcase
end
```

---

### 问题8：BRAM资源不足

**优化方法**：

**1. 减小存储深度**
```systemverilog
// 原来：1024深度，实际只需512
reg [7:0] mem [0:1023];  // 使用28个BRAM（假设36Kb块）

// 优化：减小到512
reg [7:0] mem [0:511];   // 使用14个BRAM
```

**2. 使用半双工访问**
```systemverilog
// 如果读和写不同时发生，可以共享地址线
// 节省一半BRAM资源
```

**3. 数据压缩**
- 只存储有效数据位
- 使用编码减少存储量

---

## 调试技巧

### 使用ILA定位问题

**步骤1：标记调试信号**
```systemverilog
(* mark_debug = "true" *)
wire [7:0] debug_data;
(* mark_debug = "true" *)
wire debug_valid;
```

**步骤2：在Vivado中添加ILA**
```tcl
# 综合后打开 synthesized design
# Tools -> Set up Debug
# 选择要观察的信号
# 设置触发条件（如 error_flag == 1）
# 设置采样深度（建议4096或8192）
```

**步骤3：运行时捕获**
```
1. 生成bitstream并下载到FPGA
2. 打开Hardware Manager
3. 设置触发条件
4. 等待触发或强制触发
5. 分析波形
```

---

### 常见调试场景

**场景1：输出偶尔错误**
- 怀疑：时序问题
- 方法：ILA捕获输出和中间结果
- 检查：是否在错误时钟沿输出

**场景2：状态机死锁**
- 怀疑：状态转换条件错误
- 方法：ILA监控状态寄存器
- 检查：是否进入未定义状态

**场景3：数据流断断续续**
- 怀疑：握手信号问题
- 方法：ILA捕获valid/ready信号
- 检查：握手时序是否正确

---

*这些问题都是在实际项目中遇到并解决的。*
