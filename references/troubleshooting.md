# FPGA Design Troubleshooting Guide

## Timing Issues

### **Timing Closure Failure**

#### **Id**
timing-closure

#### **Severity**
critical

#### **Summary**
Design doesn't meet timing constraints, unreliable operation

#### **Symptoms**
- Negative slack in timing report
- Works at low temperature, fails when warm
- Works on some units, fails on others

#### **Why**
Timing closure means all paths meet setup/hold requirements.

Common causes of failure:
- Long combinational paths (too much logic between FFs)
- High fanout signals (driving many loads)
- Clock skew/uncertainty
- Missing or incorrect constraints

Negative slack = path too slow = unreliable operation.

#### **Gotcha**

```verilog
// WRONG: Long combinational path
always @(*) begin
    // 20 levels of logic - will never meet 100MHz timing
    result = ((a * b) + (c * d)) * ((e + f) / g) + h;
end
```

#### **Solution**

```verilog
// CORRECT: Pipeline long operations
always @(posedge clk) begin
    // Stage 1
    mult1 <= a * b;
    mult2 <= c * d;

    // Stage 2
    sum1 <= mult1 + mult2;
    sum2 <= e + f;

    // Stage 3
    result <= sum1 + (sum2 * h);  // Simplified
end

// Reduce fanout with register duplication
// Let synthesis tool handle with: set_max_fanout 32

// Add proper timing constraints
// create_clock -period 10.0 [get_ports clk]
```

---

## Logic Errors

### **Metastability from Missing CDC Synchronizer**

#### **Id**
cdc-metastability

#### **Severity**
critical

#### **Summary**
Signals crossing clock domains without synchronization cause random failures

#### **Symptoms**
- Random bit flips in data
- FSM enters invalid state
- Works on some boards, fails on others
- Failures increase with temperature

#### **Why**
When a signal changes near a clock edge, the flip-flop may enter a metastable state - neither 0 nor 1.

This takes time to resolve (settling time). If another flip-flop samples before resolution, you get random values.

Single-bit: Use 2-FF synchronizer
Multi-bit: Use async FIFO with Gray code pointers
Pulse: Convert to toggle, synchronize, edge-detect

CDC bugs are the #1 cause of "random" FPGA failures.

#### **Gotcha**

```verilog
// WRONG: Direct connection across clock domains
always @(posedge clk_b) begin
    data_b <= data_a;  // data_a is in clk_a domain!
    // Metastability! Random values!
end
```

#### **Solution**

```verilog
// CORRECT: Use 2-FF synchronizer for single bit
(* ASYNC_REG = "TRUE" *)
reg [1:0] sync_chain;

always @(posedge clk_b) begin
    sync_chain <= {sync_chain[0], signal_a};
end
assign signal_b = sync_chain[1];

// For multi-bit data: use async FIFO
async_fifo #(.DATA_WIDTH(8)) fifo (
    .wr_clk(clk_a), .wr_data(data_a),
    .rd_clk(clk_b), .rd_data(data_b)
);
```

---

### **Unintentional Latch Inference**

#### **Id**
latch-inference

#### **Severity**
critical

#### **Summary**
Incomplete if/case creates latch instead of flip-flop

#### **Symptoms**
- Synthesis warning: 'latch inferred'
- Timing analysis fails
- Unexpected behavior after synthesis

#### **Why**
In combinational logic, if a signal isn't assigned in all branches, synthesis infers a latch to hold the previous value.

Latches are:
- Hard to analyze timing for
- Not available in all FPGA architectures
- Often indicate a design error

Almost always, you wanted a flip-flop or a complete assignment.

#### **Gotcha**

```verilog
// WRONG: Missing else creates latch
always @(*) begin
    if (enable)
        data_out = data_in;
    // What happens when enable=0? Latch!
end

// WRONG: Incomplete case creates latch
always @(*) begin
    case (sel)
        2'b00: y = a;
        2'b01: y = b;
        // Missing 2'b10, 2'b11 cases - latch!
    endcase
end
```

#### **Solution**

```verilog
// CORRECT: Assign default at start
always @(*) begin
    data_out = 8'h00;  // Default value
    if (enable)
        data_out = data_in;
end

// CORRECT: Complete case with default
always @(*) begin
    case (sel)
        2'b00: y = a;
        2'b01: y = b;
        default: y = 8'h00;  // Catch-all
    endcase
end

// CORRECT: Use full_case/parallel_case pragmas carefully
// (* full_case *) only if you guarantee coverage
```

---

### **Asynchronous Reset Release Glitch**

#### **Id**
reset-glitch

#### **Severity**
high

#### **Summary**
Releasing reset asynchronously causes metastability

#### **Symptoms**
- FSM starts in wrong state after reset
- Different behavior on different resets
- Works most of the time, occasionally fails

#### **Why**
Asserting reset asynchronously is fine - it immediately resets.
But RELEASING reset at an arbitrary time can violate recovery/removal timing on flip-flops.

Solution: Assert asynchronously, release synchronously.
This gives the reset assertion benefit (immediate) while ensuring clean release aligned to clock.

#### **Gotcha**

```verilog
// WRONG: Fully asynchronous reset
always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        state <= IDLE;
    else
        state <= next_state;
end
// Reset release can cause metastability!
```

#### **Solution**

```verilog
// CORRECT: Reset synchronizer
// Async assert, sync release

module reset_sync (
    input  wire clk,
    input  wire rst_n_async,
    output wire rst_n_sync
);
    reg [1:0] sync;

    always @(posedge clk or negedge rst_n_async) begin
        if (!rst_n_async)
            sync <= 2'b00;  // Async assert
        else
            sync <= {sync[0], 1'b1};  // Sync release
    end

    assign rst_n_sync = sync[1];
endmodule

// Use synchronized reset in design
reset_sync rst_sync (.clk(clk), .rst_n_async(rst_n), .rst_n_sync(rst_n_safe));
```

---

### **Simulation-Synthesis Mismatch**

#### **Id**
simulation-synthesis-mismatch

#### **Severity**
high

#### **Summary**
Design works in simulation but fails on FPGA

#### **Symptoms**
- Testbench passes, hardware fails
- Adding signals to debug changes behavior
- Different results from simulation and implementation

#### **Why**
Common causes:
1. Non-synthesizable constructs (initial blocks, delays)
2. Incomplete sensitivity lists
3. Blocking vs non-blocking assignment confusion
4. X-propagation differences
5. Timing assumptions in testbench

Simulation is behavior model; synthesis creates actual hardware.

#### **Gotcha**

```verilog
// WRONG: Initial blocks don't synthesize
initial begin
    count = 0;  // Only works in simulation!
end

// WRONG: Incomplete sensitivity list
always @(a or b) begin
    y = a & b & c;  // c not in sensitivity list!
    // Simulation: y updates on a or b change
    // Synthesis: combinational logic, updates on any change
end

// WRONG: Blocking in sequential, non-blocking in combinational
always @(posedge clk) begin
    a = b;  // Should be <=
    c = a;  // Gets NEW value of a (race condition)
end
```

#### **Solution**

```verilog
// CORRECT: Use reset instead of initial
always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        count <= 0;
    else
        count <= count + 1;
end

// CORRECT: Use @(*) for combinational
always @(*) begin  // All signals in sensitivity list
    y = a & b & c;
end

// CORRECT: Non-blocking for sequential
always @(posedge clk) begin
    a <= b;  // Non-blocking
    c <= a;  // Gets OLD value of a
end
```

---

## Resource Optimization

### **FPGA Resource Exhaustion**

#### **Id**
resource-exhaustion

#### **Severity**
high

#### **Summary**
Design uses more resources than available

#### **Symptoms**
- Synthesis fails with 'resource exceeded'
- Timing degrades as utilization increases
- Can't fit design even at lower clock speed

#### **Why**
FPGAs have limited:
- LUTs (logic)
- FFs (registers)
- BRAM (block RAM)
- DSP slices (multipliers)

Over ~70-80% utilization, place-and-route struggles.
Common causes: unintended resource usage, inefficient coding.

#### **Solution**

```verilog
// Check resource usage in synthesis reports

// Efficient resource usage:
// 1. Use BRAM instead of distributed RAM for large memories
(* ram_style = "block" *) reg [7:0] mem [0:1023];

// 2. Share multipliers via time-division
always @(posedge clk) begin
    case (phase)
        0: product <= a * b;  // Reuse same DSP
        1: product <= c * d;
    endcase
end

// 3. Use inference patterns tools recognize
// Let synthesis optimize instead of manual optimization

// 4. Reduce bit widths where possible
reg [7:0] counter;  // Not reg [31:0] if you only count to 100
```

---

## Debugging Techniques

### **ILA (Integrated Logic Analyzer) Usage**

#### **Description**
Xilinx ILA is an embedded logic analyzer for debugging FPGA designs in hardware.

#### **Basic Usage**

```tcl
# XDC constraints for ILA
set_property MARK_DEBUG true [get_nets {signal_name}]

# Or instantiate ILA in Verilog
ila_0 your_instance_name (
    .clk(clk),                  // input wire clk
    .probe0(signal_to_probe)    // input wire [7:0] probe0
);
```

#### **Best Practices**

1. **Plan debug signals early**: Add debug ports during design phase
2. **Use trigger conditions**: Set up complex trigger conditions to catch rare events
3. **Limit probe width**: Each probe bit consumes BRAM for capture buffer
4. **Use cross-triggering**: Trigger multiple ILA cores simultaneously
5. **Add debug ILA during development, remove for production**

#### **Common Debug Patterns**

```verilog
// Add debug output for state machines
(* mark_debug = "true" *) reg [3:0] debug_state;
assign debug_state = state;

// Debug counter for tracking events
(* mark_debug = "true" *) reg [15:0] event_counter;
always @(posedge clk)
    if (trigger_event)
        event_counter <= event_counter + 1;

// Debug flags for status monitoring
(* mark_debug = "true" *) wire debug_fifo_full;
(* mark_debug = "true" *) wire debug_fifo_empty;
assign debug_fifo_full = fifo_full;
assign debug_fifo_empty = fifo_empty;
```

---

## Static Analysis Rules

### **Case Statement Without Default**

#### **Id**
missing-default-case

#### **Severity**
warning

#### **Type**
regex

#### **Pattern**
- `case\s*\([^)]+\)[^}]*endcase(?!.*default)`
- `case\s*\([^)]+\)(?![\s\S]{0,500}default)`

#### **Message**
Case statement without default may infer latch.

#### **Fix Action**
Add 'default:' clause with explicit assignment

#### **Applies To**
- **/*.v
- **/*.sv

---

### **Blocking Assignment in Sequential Logic**

#### **Id**
blocking-in-sequential

#### **Severity**
warning

#### **Type**
regex

#### **Pattern**
- `always\s*@\s*\(\s*posedge[^)]+\)[^;]*\b\w+\s*=\s*(?!.*<=)`

#### **Message**
Use non-blocking (<=) for sequential logic to avoid race conditions.

#### **Fix Action**
Replace = with <= in always @(posedge clk) blocks

#### **Applies To**
- **/*.v
- **/*.sv

---

### **Direct Clock Domain Crossing**

#### **Id**
direct-cdc-connection

#### **Severity**
error

#### **Type**
regex

#### **Pattern**
- `always\s*@\s*\(\s*posedge\s+clk_b[^}]*\b(\w+_a)\b(?!.*sync)`

#### **Message**
Signal appears to cross clock domains without synchronizer.

#### **Fix Action**
Add 2-FF synchronizer or async FIFO for CDC

#### **Applies To**
- **/*.v
- **/*.sv

---

### **Initial Block in Synthesizable Code**

#### **Id**
initial-block

#### **Severity**
warning

#### **Type**
regex

#### **Pattern**
- `^\s*initial\s+begin(?![\s\S]*\.tb\.)`

#### **Message**
Initial blocks don't synthesize. Use reset instead.

#### **Fix Action**
Replace with synchronous reset: always @(posedge clk) if (!rst_n)

#### **Applies To**
- **/*.v
- **/*.sv

---

### **Incomplete Sensitivity List**

#### **Id**
incomplete-sensitivity

#### **Severity**
warning

#### **Type**
regex

#### **Pattern**
- `always\s*@\s*\([^*][^)]*\)\s*begin[^}]*\b(\w+)\b(?![^}]*@.*\1)`

#### **Message**
Consider using @(*) for combinational logic to auto-include all signals.

#### **Fix Action**
Replace explicit sensitivity list with always @(*)

#### **Applies To**
- **/*.v

---

### **Synchronizer Without ASYNC_REG Attribute**

#### **Id**
missing-async-reg

#### **Severity**
info

#### **Type**
regex

#### **Pattern**
- `reg\s+\[\d+:\d\]\s+\w*sync\w*(?!.*ASYNC_REG)`

#### **Message**
Synchronizer registers should have ASYNC_REG attribute for proper placement.

#### **Fix Action**
Add (* ASYNC_REG = "TRUE" *) before register declaration

#### **Applies To**
- **/*.v
- **/*.sv

---

### **Asynchronous Reset Without Synchronizer**

#### **Id**
async-reset-no-sync

#### **Severity**
warning

#### **Type**
regex

#### **Pattern**
- `negedge\s+rst_n(?!.*sync)`

#### **Message**
Asynchronous reset should be synchronized for clean release.

#### **Fix Action**
Use reset synchronizer: async assert, sync release pattern

#### **Applies To**
- **/*.v
- **/*.sv

---

### **High Fanout Signal**

#### **Id**
high-fanout-signal

#### **Severity**
info

#### **Type**
regex

#### **Pattern**
- `\.(\w+)\(enable\)[^;]*\.(\w+)\(enable\)[^;]*\.(\w+)\(enable\)`

#### **Message**
High fanout signal may cause timing issues.

#### **Fix Action**
Consider registering signal at each destination or using synthesis directives

#### **Applies To**
- **/*.v
- **/*.sv

---

### **Missing Clock Definition**

#### **Id**
missing-clock-constraint

#### **Severity**
warning

#### **Type**
regex

#### **Pattern**
- `get_ports.*clk(?!.*create_clock)`

#### **Message**
Clock port should have create_clock constraint.

#### **Fix Action**
Add: create_clock -period <ns> [get_ports clk]

#### **Applies To**
- **/*.xdc
- **/*.sdc

---

### **IO Without Delay Constraint**

#### **Id**
unconstrained-io

#### **Severity**
info

#### **Type**
regex

#### **Pattern**
- `get_ports.*data(?!.*set_input_delay|set_output_delay)`

#### **Message**
Data ports should have input/output delay constraints.

#### **Fix Action**
Add set_input_delay/set_output_delay for all data ports

#### **Applies To**
- **/*.xdc
- **/*.sdc

---

### **Non-Blocking in Combinational Logic**

#### **Id**
non-blocking-combinational

#### **Severity**
warning

#### **Type**
regex

#### **Pattern**
- `always\s*@\s*\(\s*\*\s*\)[^}]*<=`

#### **Message**
Use blocking (=) for combinational logic.

#### **Fix Action**
Replace <= with = in always @(*) blocks

#### **Applies To**
- **/*.v
- **/*.sv
