# Xilinx 7 Series FPGA Device Selection Guide

## Overview

Xilinx 7 Series FPGAs are based on 28nm High-K Metal Gate (HKMG) process technology, offering three device families with different positioning to cover the full range from low-cost to ultra-high-performance applications.

### Three Series Positioning

| Series | Positioning | Logic Cell Range | Application Scenarios |
|--------|-------------|------------------|----------------------|
| **Artix-7** | Lowest cost and power | 16K - 215K | Consumer electronics, industrial control, communications infrastructure |
| **Kintex-7** | Best price/performance | 70K - 480K | Mid-range communications, video processing, medical imaging |
| **Virtex-7** | Highest performance and capacity | 585K - 2000K | High-end networking, HPC, radar signal processing |

---

## Detailed Resource Comparison

### Artix-7 Series (Cost-Sensitive)

**Features:**
- Optimized for cost and power, small form factor packages
- Suitable for high-volume applications
- Supports DDR3 interface up to 1066 Mb/s

**Device Specifications:**

| Device | Logic Cells | Slices | FF | BRAM (36Kb) | DSP | Max I/O |
|--------|-------------|--------|-----|-------------|-----|---------|
| XC7A15T | 16,640 | 2,600 | 20,800 | 25 | 45 | 250 |
| XC7A35T | 33,280 | 5,200 | 41,600 | 50 | 90 | 300 |
| XC7A50T | 52,160 | 8,150 | 65,200 | 75 | 120 | 300 |
| XC7A75T | 75,520 | 11,800 | 94,400 | 105 | 180 | 300 |
| XC7A100T | 101,440 | 15,850 | 126,800 | 135 | 240 | 300 |
| XC7A200T | 215,360 | 33,650 | 269,200 | 365 | 740 | 500 |

**Key Features:**
- Up to 16 GTP transceivers (6.6 Gb/s)
- Supports PCIe Gen2 x4
- XADC analog interface
- Speed grades: -1, -2, -3 (-1 slowest, -3 fastest)

---

### Kintex-7 Series (Price/Performance)

**Features:**
- 2x better price/performance than previous generation
- Higher DSP performance and transceiver bandwidth
- Supports DDR3 interface up to 1866 Mb/s

**Device Specifications:**

| Device | Logic Cells | Slices | FF | BRAM (36Kb) | DSP | GTX Transceivers |
|--------|-------------|--------|-----|-------------|-----|------------------|
| XC7K70T | 65,600 | 10,250 | 82,000 | 135 | 240 | 8 |
| XC7K160T | 162,240 | 25,350 | 202,800 | 325 | 600 | 8 |
| XC7K325T | 326,080 | 50,950 | 407,600 | 445 | 840 | 16 |
| XC7K355T | 356,160 | 55,650 | 445,200 | 715 | 1440 | 24 |
| XC7K410T | 406,720 | 63,550 | 508,400 | 795 | 1540 | 16 |
| XC7K420T | 416,960 | 65,150 | 521,200 | 835 | 1680 | 32 |
| XC7K480T | 477,760 | 74,650 | 597,200 | 955 | 1920 | 32 |

**Key Features:**
- Up to 32 GTX transceivers (12.5 Gb/s)
- Supports PCIe Gen2 x8
- Stronger DSP performance (up to 2,845 GMAC/s)
- Industrial temperature support

---

### Virtex-7 Series (High-Performance)

**Features:**
- Highest system performance and capacity
- Stacked Silicon Interconnect (SSI) technology for ultra-large devices
- Up to 28.05 Gb/s transceiver rate

**Main Device Specifications:**

| Device | Logic Cells | Slices | FF | BRAM (36Kb) | DSP | GTX/GTH |
|--------|-------------|--------|-----|-------------|-----|---------|
| XC7V585T | 582,720 | 91,050 | 728,400 | 795 | 1260 | 36 GTX |
| XC7V2000T | 1,954,560 | 305,400 | 2,443,200 | 1,292 | 2160 | 36 GTX |
| XC7VX330T | 326,400 | 51,000 | 408,000 | 750 | 1120 | 28 GTX |
| XC7VX415T | 412,160 | 64,400 | 515,200 | 880 | 2160 | 48 GTH |
| XC7VX485T | 485,760 | 75,900 | 607,200 | 1,030 | 2800 | 56 GTH |
| XC7VX690T | 693,120 | 108,300 | 866,400 | 1,470 | 3600 | 80 GTH |
| XC7VX980T | 979,200 | 153,000 | 1,224,000 | 1,500 | 3600 | 72 GTH |
| XC7VX1140T | 1,139,200 | 178,000 | 1,424,000 | 1,880 | 3360 | 96 GTH |
| XC7VH580T | 580,480 | 90,700 | 725,600 | 940 | 1680 | 48 GTH + 8 GTZ |
| XC7VH870T | 876,160 | 136,900 | 1,095,200 | 1,410 | 2520 | 72 GTH + 16 GTZ |

**Key Features:**
- GTH transceivers support 13.1 Gb/s, GTZ supports 28.05 Gb/s
- Supports PCIe Gen3 x8
- Highest performance: 5,335 GMAC/s DSP performance
- Stacked Silicon Interconnect (SSI) technology for ultra-large capacity

---

## Selection Decision Tree

### Step 1: Determine Application Type

```
Application Requirements
├── Cost-sensitive + Low Power → Artix-7
├── Balanced Performance/Cost → Kintex-7
└── Ultra-high Performance/Capacity → Virtex-7
```

### Step 2: Estimate Resource Requirements

**Common Resource Estimation Methods:**

1. **Logic Resources (Slices/LUTs)**
   - Simple control logic: 100-500 LUTs per functional module
   - Complex algorithms: 1000-5000 LUTs per functional module
   - Processor soft core: MicroBlaze approximately 1500-3000 LUTs

2. **Memory Resources (BRAM)**
   - Video line buffer: (pixels per line × bit width) ÷ 36Kb
   - FIFO buffer: (depth × bit width) ÷ 36Kb
   - Data cache: Calculate based on data volume

3. **DSP Resources**
   - FIR filter: 1 DSP slice per tap
   - FFT: Calculate based on number of points and parallelism
   - Image processing: Estimate based on algorithm complexity

4. **Transceiver Requirements**
   - Gigabit Ethernet: 1 Gb/s × quantity
   - PCIe: Gen1(2.5G), Gen2(5G), Gen3(8G)
   - Fibre Channel: 1/2/4/8/10/16 Gb/s

### Step 3: Consider Future Expansion

**Recommended Headroom:**
- Logic resources: Actual usage ≤ 70% of available resources
- BRAM: Actual usage ≤ 80% of available resources
- DSP: Reserve 20-30% based on algorithm characteristics
- I/O: Consider debugging and testing requirements

---

## Typical Application Selection Recommendations

### Application Scenario Reference Table

| Application Scenario | Recommended Series | Typical Device | Key Considerations |
|---------------------|-------------------|----------------|-------------------|
| **Industrial Control** | Artix-7 | XC7A35T/50T | Low cost, low power, many I/Os |
| **Machine Vision** | Kintex-7 | XC7K325T | Strong DSP, good DDR3 support |
| **Software Radio** | Kintex-7/Virtex-7 | XC7K410T/XC7VX690T | Many transceivers, strong DSP |
| **100G Networking** | Virtex-7 | XC7VX690T/VX980T | 28G transceivers, large capacity |
| **Datacenter Acceleration** | Virtex-7 | XC7VX1140T | Maximum capacity, PCIe Gen3 |
| **Medical Devices** | Artix-7/Kintex-7 | XC7A100T/XC7K160T | Low power, high reliability |
| **Test & Measurement** | Kintex-7/Virtex-7 | XC7K325T/XC7VX485T | High-speed acquisition, large bandwidth |

---

## Package and Speed Grades

### Package Types

| Package Code | Type | Pitch | Application Scenarios |
|-------------|------|-------|----------------------|
| CP | Wire-bond CSP | 0.5mm | Extremely small form factor |
| CS | Wire-bond CSP | 0.8mm | Small consumer electronics |
| FT | Wire-bond Fine-pitch | 1.0mm | Medium density |
| SB | Lidless Flip-chip | 0.8mm | High performance + low cost |
| FB | Lidless Flip-chip | 1.0mm | High performance |
| FF/FG | Flip-chip | 1.0mm | Highest performance |
| FL/FH | Flip-chip | 1.0mm | Virtex-7 specific |

### Speed Grade Descriptions

**Artix-7/Kintex-7:**
- `-1`: Slowest, lowest power
- `-2`: Medium performance (most commonly used)
- `-3`: Highest performance
- `-1L/-2L`: Low power versions

**Virtex-7:**
- `-1`: Base performance
- `-2`: Standard performance
- `-3`: Highest performance
- `-2G`: SSI devices, support 12.5G/13.1G/28.05G transceivers

---

## Quick Selection Reference Table

### Find by Resource Requirements

**Small Designs (<50K Logic Cells):**
- Artix-7: XC7A15T ~ XC7A50T
- Kintex-7: XC7K70T

**Medium Designs (50K-200K Logic Cells):**
- Artix-7: XC7A75T ~ XC7A200T
- Kintex-7: XC7K160T ~ XC7K325T

**Large Designs (200K-500K Logic Cells):**
- Kintex-7: XC7K355T ~ XC7K480T
- Virtex-7: XC7VX330T ~ XC7VX485T

**Extra Large Designs (>500K Logic Cells):**
- Virtex-7: XC7V585T and above
- Or consider SSI devices (XC7V2000T, XC7VX1140T, etc.)

---

## Reference Materials

- DS180: 7 Series FPGAs Overview
- 7-series-product-selection-guide.pdf
- UG470-UG483: 7 Series User Guides

**Note:** All data is based on Xilinx official datasheets, this guide is for reference only.
