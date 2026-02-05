---
name:fpga-hardware-design-and-review-guide
description: >
  Comprehensive FPGA hardware design guide based on real-world project experience.
  Covers pipeline architecture, timing optimization, SystemVerilog coding patterns,
  CDC handling, and practical debugging techniques. Use this skill when:
  (1) designing FPGA modules with timing constraints,
  (2) implementing video processing or data path designs,
  (3) optimizing for resource utilization and timing closure,
  (4) reviewing RTL code for hardware implementation,
  (5) debugging synthesis or implementation issues.
license: MIT
compatibility: Works with Claude Code, Cursor, and AI coding assistants.
metadata:
  author: peng
  version: "1.0"
  based_on: Real project experience (RGB-to-YUV converter, video processing)
allowed-tools: Read Write Edit Bash
---

# FPGA Hardware Design Guide

A practical FPGA hardware design guide based on real-world project experience.

## Core Design Philosophy

### 1. Pipeline Architecture First

When processing high-speed data streams (video, network packets), adopt multi-stage pipeline design:
- **Single-stage processing**: Combinational logic delay too large, prone to timing violations
- **Multi-stage pipeline**: Insert registers at each stage, distribute delay, increase clock frequency
- **Typical applications**: RGB-to-YUV conversion, image filtering, protocol parsing

**Real Case**: RGB-to-YUV converter with 5-stage pipeline
- Stage 0: Input register (synchronize input signals)
- Stage 1: Multiply operation (coefficient * pixel value)
- Stage 2: Partial accumulation (R*coef_r + G*coef_g)
- Stage 3: Final accumulation (+ B*coef_b)
- Stage 4: Shift and saturation (truncate result to 8-bit)

### 2. The Art of Bit-Width Management

**Bit-width calculation principles**:
```
Multiplication bit-width = input bit-width + coefficient bit-width + 1 (sign bit)
Accumulation bit-width = multiplication bit-width + log2(number of additions) + 1 (guard bit)
```

**Rules of thumb**:
- 8-bit unsigned * 9-bit signed coefficient = 18-bit signed result
- 3 numbers of 18-bit addition = 20-bit (leave 2 guard bits to prevent overflow)
- After right-shifting 8 bits = 8-bit final result

### 3. Iron Rules of Synchronous Design

**Rules that must be followed**:
1. **All flip-flops use the same clock domain** (unless CDC is explicitly needed)
2. **Synchronous reset preferred over asynchronous reset** (avoid metastability propagation)
3. **Input signals must be registered for two cycles** (cross-clock domain or external inputs)
4. **Combinational logic outputs must be registered** (avoid glitch propagation)

**Lessons learned**:
- Asynchronous reset leads to unpredictable behavior when clock is unstable
- Unregistered combinational outputs may produce glitches after place-and-route
- Direct use of cross-clock domain signals causes metastability

## Timing Closure Practical Techniques

### Delay Analysis and Optimization

**Identifying critical paths**:
1. Check `Worst Negative Slack (WNS)` in synthesis report
2. Analyze `Total Negative Slack (TNS)` distribution
3. Locate logic levels with maximum delay

**Optimization strategies**:
1. **Insert pipeline registers** (most effective)
   - Insert FF in the middle of combinational logic
   - Each stage delay < 70% of target clock period
   
2. **Logic retiming**
   - Use `set_property RETIMING true`
   - Let tool automatically move register positions
   
3. **Critical signal optimization**
   - Use `set_property HIGH_PRIORITY true` for critical paths
   - Manual placement for critical modules `set_property LOC ...`

**Real data**:
- Original design: critical path 15ns, target 10ns (not met)
- After inserting 2 pipeline stages: critical path 7ns (met + 30% margin)
- Latency cost: 2 clock cycles (acceptable)

## Resource Optimization Strategies

### LUT Optimization

**Methods to reduce LUT usage**:
1. **Use case statements instead of if-else chains** (more efficient LUT synthesis)
2. **Avoid complex nested ternary operators**
3. **Use DSP Slices instead of LUTs for multiplication**

**Comparison example**:
```systemverilog
// Inefficient: nested if-else
if (condition1) out = a;
else if (condition2) out = b;
else if (condition3) out = c;
// Uses ~20 LUTs

// Efficient: case statement
case ({condition1, condition2, condition3})
    3'b100: out = a;
    3'b010: out = b;
    3'b001: out = c;
    default: out = d;
endcase
// Uses ~8 LUTs
```

### BRAM Usage Techniques

**When to use BRAM**:
- Storage depth > 16 (typically)
- Dual-port access required
- Large lookup tables (>1KB)

**When to use distributed RAM**:
- Small storage (<16 depth)
- Asynchronous read needed
- Save BRAM resources

**Code example**:
```systemverilog
// Automatically inferred as BRAM (36Kb block)
reg [7:0] mem [0:1023];  // 8Kbits
always @(posedge clk) begin
    if (we) mem[addr] <= din;
    dout <= mem[addr];  // Synchronous read
end

// Small capacity automatically uses LUTRAM
reg [7:0] small_mem [0:15];  // 128bits
```

### DSP Slice Optimization

**Fully utilize DSP48E1**:
- 25×18 multiplier (supports signed/unsigned)
- 48-bit accumulator
- Pre-adder (for symmetric FIR filters)

**Avoid DSP waste**:
- Don't use DSP for small multiplications (<8bit), LUTs are more efficient
- Use dedicated routing (ACIN/ACOUT) when cascading DSPs
- Use CE and SCLR controls to save power

## Debugging and Verification Methods

### Simulation Strategy

**Three-level verification system**:
1. **Behavioral simulation** (pre-synthesis)
   - Verify algorithm correctness
   - Use ideal delay models
   
2. **Post-synthesis simulation**
   - Verify synthesis result functionality
   - Check rough timing estimates
   
3. **Post-implementation simulation**
   - Include actual routing delays
   - Closest to real hardware

**Testbench writing essentials**:
```systemverilog
// 1. Self-checking test
initial begin
    // Apply stimulus
    apply_stimulus();
    
    // Wait for processing
    repeat(10) @(posedge clk);
    
    // Check results
    if (dout !== expected) begin
        $error("Test failed! Expected %h, got %h", expected, dout);
        $finish;
    end
    $display("Test passed!");
end

// 2. Coverage check
covergroup cg @(posedge clk);
    coverpoint state {
        bins idle = {IDLE};
        bins busy = {BUSY};
        bins done = {DONE};
    }
endgroup
```

### On-board Debugging Techniques

**Using ILA (Integrated Logic Analyzer)**:
1. Mark critical signals as `mark_debug`
2. Set trigger conditions (e.g., error flags, specific states)
3. Capture data to Vivado for analysis

**Using VIO (Virtual Input/Output)**:
- Modify parameters in real-time (e.g., filter coefficients)
- Monitor internal status registers
- Debug without recompilation

**Real debugging case**:
- Issue: YUV output occasionally shows wrong values
- Method: ILA captured multiplication intermediate results
- Finding: Sign extension error caused high-bit overflow
- Solution: Fixed signed number extension logic

## Reference Documentation

**Detailed design patterns**: See `references/design-patterns.md`
- CDC synchronizer design
- FIFO implementation
- AXI-Stream interface

**Common issues troubleshooting**: See `references/troubleshooting.md`
- Timing violation diagnosis process
- Metastability handling
- Resource conflict resolution

**Device selection guide**: See `references/device-selection.md`
- Selection based on resource requirements
- Package and speed grade selection
- Cost optimization suggestions

## Golden Rules

1. **Function first, optimization second** — Make the design work correctly first, then optimize timing and resources
2. **Constrain early, relax late** — Strict timing constraints early, relax based on situation later
3. **Register all boundaries** — Module inputs and outputs must be registered to avoid timing coupling
4. **Documentation is code** — Clear comments and documentation are more important than complex designs
5. **Test-driven development** — Write testbench first, then implement functionality

---

*This guide is based on real-world project experience and is continuously updated.*
