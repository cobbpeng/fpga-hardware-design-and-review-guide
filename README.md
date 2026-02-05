# FPGA Hardware Design Guide

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A comprehensive FPGA hardware design guide based on real-world project experience, covering pipeline design, timing optimization, SystemVerilog coding patterns, and practical debugging techniques.

## Overview

This guide differs from traditional FPGA tutorials. It's built from hands-on project experience, providing:
- Proven design patterns tested in real projects
- Practical solutions to common problems
- Debugging techniques learned from mistakes
- Actionable device selection advice

## Key Features

✨ **Based on Real Experience** - All content comes from actual projects, not theory  
✨ **Independent & Original** - Self-contained system, no dependency on other skills  
✨ **Practice-Oriented** - Concrete code examples and solutions  
✨ **Continuously Updated** - Growing with new project experiences  

## Main Content

### 🎯 SKILL.md - Core Guide
- Pipeline architecture design principles
- The art of bit-width management
- Practical timing closure techniques
- Resource optimization strategies
- Debugging and verification methods

### 📚 Reference Documentation

#### `design-patterns.md`
Practical design pattern templates:
- Multi-stage pipeline pattern
- Signed arithmetic operations
- Cross-clock domain synchronizer (CDC)
- Simple synchronous FIFO
- AXI-Stream interface

#### `troubleshooting.md`
Problem diagnosis handbook:
- Timing violation diagnosis and solutions
- Logic error case studies
- Resource optimization methods
- ILA debugging techniques

#### `device-selection.md`
Device selection guide:
- Artix-7 / Kintex-7 / Virtex-7 comparison
- Resource estimation methods
- Real selection cases
- Cost optimization tips

## When to Use

Use this skill when you need to:
1. Design FPGA modules with timing constraints
2. Implement video processing or data path designs
3. Optimize resource utilization and achieve timing closure
4. Review RTL code for hardware implementation quality
5. Debug synthesis or implementation issues

## Installation

```bash
npx skills add https://github.com/cobbpeng/fpga-hardware-design-and-review-guide --skill fpga-hardware-design-guide
```

## Usage Examples

### Example 1: Pipeline Design
> "I need to design a high-speed FIR filter with 5 pipeline stages. How should I organize the code?"

**The skill will provide:**
- Complete pipeline code template
- Explanation of each stage's function
- Timing optimization suggestions

### Example 2: Timing Issues
> "Synthesis report shows critical path delay of 15ns, target is 10ns. How to optimize?"

**The skill will provide:**
- Analysis of possible causes
- 3 optimization solutions (pipeline insertion, retiming, manual placement)
- Specific Tcl constraint commands

### Example 3: Device Selection
> "Building a 4-channel SDR at 50MSPS per channel, which FPGA should I choose?"

**The skill will provide:**
- Resource calculation (logic, DSP, BRAM)
- Specific recommendation (e.g., XC7K160T)
- Selection rationale and margin considerations

## Project Case Studies

Experience in this guide comes from these actual projects:
- **RGB-to-YUV Converter** - 5-stage pipeline design, bit-width management
- **VGA Controller** - Video timing generation, frame buffer management
- **FIR Filter** - DSP optimization, coefficient storage
- **AXI Interface** - Bus protocol, handshake handling

## Comparison with Other Skills

| Feature | This Skill | Generic FPGA Skills |
|---------|-----------|---------------------|
| Content Source | Real project experience | Official documentation |
| Code Examples | Verified implementations | Theoretical examples |
| Problem Solving | Actual pitfalls encountered | Generic advice |
| Selection Guide | Case-by-case analysis | Parameter comparison tables |

## Contributing

Issues and Pull Requests are welcome!

If you have:
- Problems and solutions from real projects
- Unique design techniques
- Device selection experiences

Please contribute to this guide.

## Author

**peng** - FPGA Hardware Design Engineer

Specializing in:
- Xilinx 7 Series FPGAs
- Video/Image Processing
- Digital Signal Processing
- High-speed Interface Design

## License

MIT License - See [LICENSE](LICENSE) file for details

## Disclaimer

This guide is written based on personal experience for reference only. Please refer to official Xilinx documentation for actual projects.

---

*Continuously updating...*
