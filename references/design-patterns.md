# 设计模式库

基于实际项目经验的设计模式集合。

## 1. 多级流水线模式

### 适用场景
- 高速数据处理（视频、网络、信号处理）
- 组合逻辑延迟超过时钟周期
- 需要提高工作频率

### 模式结构
```systemverilog
module pipeline_example #(
    parameter DATA_WIDTH = 8,
    parameter STAGES = 5
)(
    input  wire                  clk,
    input  wire                  rst_n,
    input  wire [DATA_WIDTH-1:0] data_in,
    input  wire                  data_valid_in,
    output wire [DATA_WIDTH-1:0] data_out,
    output wire                  data_valid_out
);

    // 流水线寄存器数组
    reg [DATA_WIDTH-1:0] pipe_reg [0:STAGES-1];
    reg                  valid_reg [0:STAGES-1];
    
    // Stage 0: 输入采样
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            pipe_reg[0] <= {DATA_WIDTH{1'b0}};
            valid_reg[0] <= 1'b0;
        end else begin
            pipe_reg[0] <= data_in;
            valid_reg[0] <= data_valid_in;
        end
    end
    
    // Stage 1-N: 中间处理（示例：简单传递，实际可插入运算）
    genvar i;
    generate
        for (i = 1; i < STAGES; i = i + 1) begin : gen_pipeline
            always @(posedge clk or negedge rst_n) begin
                if (!rst_n) begin
                    pipe_reg[i] <= {DATA_WIDTH{1'b0}};
                    valid_reg[i] <= 1'b0;
                end else begin
                    pipe_reg[i] <= pipe_reg[i-1];
                    valid_reg[i] <= valid_reg[i-1];
                end
            end
        end
    endgenerate
    
    // 输出
    assign data_out = pipe_reg[STAGES-1];
    assign data_valid_out = valid_reg[STAGES-1];
    
endmodule
```

### 使用要点
- 每级延迟控制在目标时钟的60-70%
- 保持数据有效信号同步传递
- 复位时所有级清零

---

## 2. 带符号算术运算模式

### 问题背景
无符号数与有符号系数相乘时的符号处理容易出错。

### 解决方案
```systemverilog
module signed_arithmetic #(
    parameter DATA_WIDTH = 8,
    parameter COEF_WIDTH = 9  // 系数位宽（包含符号位）
)(
    input  wire                       clk,
    input  wire                       rst_n,
    input  wire [DATA_WIDTH-1:0]      data_in,      // 无符号输入
    input  wire signed [COEF_WIDTH-1:0] coef,       // 有符号系数
    output wire signed [DATA_WIDTH+COEF_WIDTH:0] result  // 扩展位宽结果
);

    // 关键：无符号转有符号
    wire signed [DATA_WIDTH:0] data_signed;
    assign data_signed = $signed({1'b0, data_in});  // 高位补0转有符号
    
    // 乘法：结果位宽 = DATA_WIDTH + 1 + COEF_WIDTH
    wire signed [DATA_WIDTH+COEF_WIDTH:0] mult_result;
    assign mult_result = data_signed * coef;
    
    // 寄存输出
    reg signed [DATA_WIDTH+COEF_WIDTH:0] result_reg;
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            result_reg <= 'd0;
        else
            result_reg <= mult_result;
    end
    
    assign result = result_reg;
    
endmodule
```

### 关键技巧
1. **符号扩展**：无符号数转有符号时高位补0
2. **位宽计算**：乘积位宽 = 被乘数位宽 + 乘数位宽
3. **饱和处理**：防止结果溢出

---

## 3. 双触发器同步器（CDC）

### 适用场景
- 单比特信号跨时钟域传输
- 亚稳态概率需要降到最低

### 实现代码
```systemverilog
module cdc_sync_2ff #(
    parameter STAGES = 2  // 默认2级，高速可用3级
)(
    input  wire clk_dst,    // 目的时钟
    input  wire rst_n,
    input  wire async_in,   // 异步输入
    output wire sync_out    // 同步输出
);

    // Xilinx约束：将触发器放置在一起
    (* ASYNC_REG = "TRUE" *)
    reg [STAGES-1:0] sync_chain;
    
    always @(posedge clk_dst or negedge rst_n) begin
        if (!rst_n) begin
            sync_chain <= {STAGES{1'b0}};
        end else begin
            sync_chain <= {sync_chain[STAGES-2:0], async_in};
        end
    end
    
    assign sync_out = sync_chain[STAGES-1];
    
endmodule
```

### 注意事项
- 仅适用于单比特信号
- 多比特信号必须使用FIFO或握手协议
- ASYNC_REG约束确保工具将触发器放在一起

---

## 4. 简单同步FIFO

### 适用场景
- 跨时钟域数据传输
- 数据缓冲（速率匹配）

### 实现代码
```systemverilog
module simple_fifo #(
    parameter DATA_WIDTH = 8,
    parameter DEPTH = 16,
    parameter ADDR_WIDTH = $clog2(DEPTH)
)(
    input  wire                  clk,
    input  wire                  rst_n,
    
    // 写接口
    input  wire [DATA_WIDTH-1:0] wr_data,
    input  wire                  wr_en,
    output wire                  full,
    
    // 读接口
    output wire [DATA_WIDTH-1:0] rd_data,
    input  wire                  rd_en,
    output wire                  empty
);

    // 存储器
    reg [DATA_WIDTH-1:0] mem [0:DEPTH-1];
    
    // 读写指针
    reg [ADDR_WIDTH:0] wr_ptr;  // 额外1bit用于判断满/空
    reg [ADDR_WIDTH:0] rd_ptr;
    
    // 写操作
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            wr_ptr <= 'd0;
        else if (wr_en && !full)
            wr_ptr <= wr_ptr + 1'b1;
    end
    
    always @(posedge clk) begin
        if (wr_en && !full)
            mem[wr_ptr[ADDR_WIDTH-1:0]] <= wr_data;
    end
    
    // 读操作
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            rd_ptr <= 'd0;
        else if (rd_en && !empty)
            rd_ptr <= rd_ptr + 1'b1;
    end
    
    assign rd_data = mem[rd_ptr[ADDR_WIDTH-1:0]];
    
    // 状态判断
    assign full  = (wr_ptr[ADDR_WIDTH] != rd_ptr[ADDR_WIDTH]) && 
                   (wr_ptr[ADDR_WIDTH-1:0] == rd_ptr[ADDR_WIDTH-1:0]);
    assign empty = (wr_ptr == rd_ptr);
    
endmodule
```

---

## 5. AXI-Stream接口模板

### 基本接口定义
```systemverilog
module axis_interface #(
    parameter DATA_WIDTH = 32,
    parameter USER_WIDTH = 1,
    parameter DEST_WIDTH = 1
)(
    input  wire                   clk,
    input  wire                   rst_n,
    
    // AXI-Stream Slave（输入）
    input  wire [DATA_WIDTH-1:0]  s_axis_tdata,
    input  wire                   s_axis_tvalid,
    output wire                   s_axis_tready,
    input  wire                   s_axis_tlast,  // 可选
    input  wire [USER_WIDTH-1:0]  s_axis_tuser,  // 可选
    
    // AXI-Stream Master（输出）
    output wire [DATA_WIDTH-1:0]  m_axis_tdata,
    output wire                   m_axis_tvalid,
    input  wire                   m_axis_tready,
    output wire                   m_axis_tlast,
    output wire [USER_WIDTH-1:0]  m_axis_tuser
);

    // 简单的透传示例
    assign m_axis_tdata  = s_axis_tdata;
    assign m_axis_tvalid = s_axis_tvalid;
    assign s_axis_tready = m_axis_tready;
    assign m_axis_tlast  = s_axis_tlast;
    assign m_axis_tuser  = s_axis_tuser;
    
    // 实际应用中在此插入处理逻辑
    
endmodule
```

### 握手机制要点
1. **TVALID由发送方控制**，数据有效时拉高
2. **TREADY由接收方控制**，可以接收时拉高
3. **数据传输发生在两者都高时的时钟上升沿**
4. **TVALID一旦拉高，必须保持直到握手完成**

---

*这些模式都经过实际项目验证。*
