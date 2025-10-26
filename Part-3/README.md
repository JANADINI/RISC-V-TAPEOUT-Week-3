
# OpenSTA Labs - Static Timing Analysis

## Introduction

This guide provides hands-on experience with **OpenSTA** for performing timing verification on digital designs, from basic examples to complete PVT corner analysis of VSDBabySoC.

**Learning Objectives:**
- Install and run OpenSTA via Docker
- Perform STA using interactive commands and TCL scripts
- Analyze synthesized netlists with timing libraries
- Conduct multi-corner timing verification
- Interpret timing reports and identify violations

---

## Table of Contents

- [Installation & Setup](#installation--setup)
- [Basic Example Analysis](#basic-example-analysis)
- [VSDBabySoC Timing Analysis](#vsdbabysoc-timing-analysis)
- [PVT Corner Analysis](#pvt-corner-analysis)
- [Summary](#summary)

---

## Installation & Setup

### Installing OpenSTA

OpenSTA runs in Docker for consistent, dependency-free execution.

**Prerequisites:**
```bash
sudo apt update
sudo apt install docker.io git -y
```

**Clone and Build:**
```bash
git clone https://github.com/parallaxsw/OpenSTA.git
cd OpenSTA
docker build --file Dockerfile.ubuntu22.04 --tag opensta .
```

**Launch OpenSTA:**
```bash
docker run -i -v /home/janadinisk:/home/janadinisk opensta
```

**Volume Mounting:** The `-v` flag mounts your host directory inside the container, allowing OpenSTA to access your design files using host paths.

---

## Basic Example Analysis

### Interactive Mode

**Step 1: Load Libraries and Design**
```bash
# Launch OpenSTA
docker run -i -v /home/janadinisk:/home/janadinisk opensta

# Inside OpenSTA prompt:
read_liberty /home/janadinisk/vsd/VLSI/OpenSTA/examples/nangate45_slow.lib.gz
read_verilog /home/janadinisk/vsd/VLSI/OpenSTA/examples/example1.v
link_design top
```

**Step 2: Define Constraints**
```bash
create_clock -name clk -period 10 {clk1 clk2 clk3}
set_input_delay -clock clk 0 {in1 in2}
```

**Step 3: Generate Reports**
```bash
# Setup timing (max delay)
report_checks

# Hold timing (min delay)
report_checks -path_delay min

# Both together
report_checks -path_delay min_max
```

---

### Understanding Reports

#### Setup Check (Max Path)
```
Startpoint: r2 (rising edge-triggered flip-flop clocked by clk)
Endpoint: r3 (rising edge-triggered flip-flop clocked by clk)
Path Type: max

Delay    Time   Description
---------------------------------------------------------
  0.00    0.00   clock clk (rise edge)
  0.23    0.23 v r2/Q (DFF_X1)           ← Launch
  0.08    0.31 v u1/Z (BUF_X1)           ← Buffer
  0.10    0.41 v u2/ZN (AND2_X1)         ← AND gate
  0.00    0.41 v r3/D (DFF_X1)           ← Capture
          0.41   data arrival time

 10.00   10.00   clock clk (rise edge)
 -0.16    9.84   library setup time
          9.84   data required time
---------------------------------------------------------
          9.43   slack (MET)             ← Pass!
```

**Analysis:** Data arrives at 0.41ns, must arrive by 9.84ns → 9.43ns slack (PASS)

---

#### Hold Check (Min Path)
```
Startpoint: in1 (input port)
Endpoint: r1 (flip-flop)
Path Type: min

  0.00    0.00   data arrival time
  0.01    0.01   data required time
---------------------------------------------------------
         -0.01   slack (VIOLATED)        ← Fail!
```

**Problem:** Zero delay path → data arrives too fast → hold violation

**Fix:** Add realistic input delay or insert buffer

---

### SPEF Analysis

**SPEF** (Standard Parasitic Exchange Format) includes real wire delays:

```bash
read_liberty /home/janadinisk/vsd/VLSI/OpenSTA/examples/nangate45_slow.lib.gz
read_verilog /home/janadinisk/vsd/VLSI/OpenSTA/examples/example1.v
link_design top
read_spef /home/janadinisk/vsd/VLSI/OpenSTA/examples/example1.dspef
create_clock -name clk -period 10 {clk1 clk2 clk3}
report_checks
```

**Impact:** Wire delays increase path delay from 0.41ns → 7.92ns (more realistic!)

---

### TCL Script Automation

**File: `min_max_analysis.tcl`**
```tcl
# Load corner libraries
read_liberty -max /home/janadinisk/vsd/VLSI/OpenSTA/examples/nangate45_slow.lib.gz
read_liberty -min /home/janadinisk/vsd/VLSI/OpenSTA/examples/nangate45_fast.lib.gz

# Read design
read_verilog /home/janadinisk/vsd/VLSI/OpenSTA/examples/example1.v
link_design top

# Apply constraints
create_clock -name clk -period 10 {clk1 clk2 clk3}
set_input_delay -clock clk 0 {in1 in2}

# Report
report_checks -path_delay min_max
```

**Run:**
```bash
docker run -i -v /home/janadinisk:/home/janadinisk opensta \
  /home/janadinisk/vsd/VLSI/OpenSTA/scripts/min_max_analysis.tcl
```

---

## VSDBabySoC Timing Analysis

### Design Overview

VSDBabySoC contains:
- **RISC-V CPU (rvmyth)** - 32-bit processor
- **PLL (avsdpll)** - Clock generator
- **DAC (avsddac)** - Digital-to-analog converter
- **Clock:** 11ns period (~91 MHz)

---

### TCL Script

**File: `vsdbabysoc_sta.tcl`**
```tcl
# Load standard cell library
read_liberty -min /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib 
read_liberty -max /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# Load analog IP libraries
read_liberty -min /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/avsdpll.lib
read_liberty -max /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/avsdpll.lib

read_liberty -min /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/avsddac.lib
read_liberty -max /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/avsddac.lib

# Read synthesized netlist
read_verilog /home/janadinisk/vsd/VLSI/VSDBabySoC/output/vsdbabysoc_synth.v

# Link design
link_design vsdbabysoc

# Apply SDC constraints
read_sdc /home/janadinisk/vsd/VLSI/VSDBabySoC/src/sdc/vsdbabysoc_synthesis.sdc

# Generate report
report_checks -path_delay min_max
```

---

### Common Issues & Fixes

**Issue 1: Liberty Syntax Error**
```
Error: syntax error line 74 in avsdpll.lib
//pin (GND#2) {
```

**Fix:** Replace `//` comments with `/* */`:
```liberty
/* 
pin (GND#2) {
  direction : input;
}
*/
```

**Issue 2: Fanout Warning**
```
Warning: default_fanout_load is 0.0
```

**Fix:** Benign warning - can be ignored.

---

### Timing Results

**Hold Path:**
```
Path: _9501_ → _10343_
Data Arrival: 0.27ns
Required: -0.03ns
Slack: 0.31ns (MET) ✅
```

**Setup Path:**
```
Path: _10458_ → _10023_
Data Arrival: 9.76ns
Required: 10.86ns
Slack: 1.11ns (MET) ✅
```

**Both hold and setup paths pass with good margin!**

---

### Advanced Reports

**Detailed Path Analysis:**
```bash
report_checks -digits 4 -fields {capacitance slew input_pins fanout}
```

Shows:
- **Capacitance** - Load at each node
- **Slew** - Signal transition time
- **Fanout** - Number of driven loads

**Power Analysis:**
```bash
report_power
```

Shows total power breakdown (dynamic + static).

---

## PVT Corner Analysis

### Understanding PVT

**PVT = Process, Voltage, Temperature**

| Factor | Corners | Impact |
|--------|---------|--------|
| **Process** | FF (fast), TT (typical), SS (slow) | Manufacturing variation |
| **Voltage** | High (1.95V), Nominal (1.8V), Low (1.62V) | Supply fluctuation |
| **Temperature** | -40°C, 25°C, 125°C | Operating environment |

**Corner Strategy:**
- **Setup checks:** Use slow corners (SS, low voltage, high temp)
- **Hold checks:** Use fast corners (FF, high voltage, low temp)

---

### Multi-Corner Script

**File: `sta_across_pvt.tcl`**
```tcl
# Define PVT libraries
set list_of_lib_files(1) "sky130_fd_sc_hd__tt_025C_1v80.lib"
set list_of_lib_files(2) "sky130_fd_sc_hd__ff_100C_1v95.lib"
set list_of_lib_files(3) "sky130_fd_sc_hd__ss_100C_1v40.lib"
set list_of_lib_files(4) "sky130_fd_sc_hd__ss_n40C_1v28.lib"
# ... (add all 13 corners)

# Load analog IPs
read_liberty /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/avsdpll.lib
read_liberty /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/avsddac.lib

# Create output directory
exec mkdir -p /home/janadinisk/vsd/VLSI/OpenSTA/output

# Loop through corners
for {set i 1} {$i <= [array size list_of_lib_files]} {incr i} {
    # Load corner library
    read_liberty /home/janadinisk/vsd/VLSI/skywater-pdk/timing/$list_of_lib_files($i)
    
    # Read and link design
    read_verilog /home/janadinisk/vsd/VLSI/VSDBabySoC/output/vsdbabysoc_synth.v
    link_design vsdbabysoc
    
    # Apply constraints
    read_sdc /home/janadinisk/vsd/VLSI/VSDBabySoC/src/sdc/vsdbabysoc_synthesis.sdc
    
    # Generate reports
    report_checks -path_delay min_max \
                  > /home/janadinisk/vsd/VLSI/OpenSTA/output/timing_$list_of_lib_files($i).txt
    
    exec echo "$list_of_lib_files($i)" >> /home/janadinisk/vsd/VLSI/OpenSTA/output/worst_slack.txt
    report_worst_slack -max -digits 4 >> /home/janadinisk/vsd/VLSI/OpenSTA/output/worst_slack.txt
    report_tns -digits 4 >> /home/janadinisk/vsd/VLSI/OpenSTA/output/tns.txt
    report_wns -digits 4 >> /home/janadinisk/vsd/VLSI/OpenSTA/output/wns.txt
}
```

**Run:**
```bash
docker run -i -v /home/janadinisk:/home/janadinisk opensta \
  /home/janadinisk/vsd/VLSI/OpenSTA/scripts/sta_across_pvt.tcl
```

---

### PVT Results Summary

| PVT Corner | TNS (ns) | WNS (ns) | Worst Setup Slack (ns) | Worst Hold Slack (ns) | Status |
|------------|----------|----------|------------------------|----------------------|--------|
| **tt_025C_1v80** | 0.00 | 0.00 | 1.11 | 0.31 | ✅ PASS |
| **ff_100C_1v95** | 0.00 | 0.00 | 3.84 | 0.20 | ✅ PASS |
| **ff_n40C_1v65** | 0.00 | 0.00 | 2.12 | 0.26 | ✅ PASS |
| **ss_100C_1v40** | -7521.42 | -13.04 | -13.04 | 0.91 | ❌ FAIL |
| **ss_100C_1v60** | -2909.84 | -6.28 | -6.28 | 0.64 | ❌ FAIL |
| **ss_n40C_1v28** | -36775.84 | -52.90 | -52.90 | 1.83 | ❌ FAIL |
| **ss_n40C_1v35** | -23278.99 | -33.20 | -33.20 | 1.35 | ❌ FAIL |
| **ss_n40C_1v40** | -17170.59 | -24.66 | -24.66 | 1.12 | ❌ FAIL |
| **ss_n40C_1v76** | -1905.43 | -3.96 | -3.96 | 0.50 | ❌ FAIL |

---

### Analysis

**Key Observations:**

1. **Fast Corners (FF):** All pass with large positive slack
   - Best case: 3.84ns slack at ff_100C_1v95
   - Hold timing safe with margins > 0.2ns

2. **Slow Corners (SS):** Setup violations occur
   - Worst case: -52.90ns WNS at ss_n40C_1v28
   - Indicates design cannot meet 11ns clock period at slow corners

**Violations at Slow Corners:**
- `ss_n40C_1v28` → Most critical (-52.9ns)
- `ss_n40C_1v35` → Severe (-33.2ns)
- `ss_100C_1v40` → Moderate (-13.0ns)

**Hold Timing:** All corners pass (0.2ns to 1.8ns positive slack)

---

### Fixing Setup Violations

**Option 1: Reduce Clock Frequency**
```tcl
# Change from 11ns to 15ns period
create_clock -period 15.0 [get_ports CLK]
```

**Option 2: Optimize Critical Paths**
- Upsize gates on critical paths
- Add pipeline stages
- Reduce logic depth
- Improve clock tree (reduce skew)

**Option 3: Use Faster Process Corner**
- Operate at higher voltage
- Control temperature environment
- Select faster transistor bins

---

### Key Metrics Explained

| Metric | Full Name | Meaning |
|--------|-----------|---------|
| **WNS** | Worst Negative Slack | Most critical timing violation |
| **TNS** | Total Negative Slack | Sum of all negative slacks |
| **Worst Max Slack** | - | Smallest positive slack (setup) |
| **Worst Min Slack** | - | Smallest positive slack (hold) |

**Sign-off Criteria:**
- ✅ WNS ≥ 0 (no setup violations)
- ✅ TNS = 0 (no accumulated violations)
- ✅ Worst Min Slack > 0 (no hold violations)

---

## Summary

### Key Takeaways

1. **OpenSTA Setup**
   - Docker-based installation for consistency
   - Volume mounting for file access
   - Interactive and batch mode operation

2. **Basic STA Flow**
   - Load libraries (Liberty files)
   - Read netlist
   - Apply constraints (clocks, I/O delays)
   - Generate timing reports

3. **Report Types**
   - `report_checks`: Detailed path analysis
   - `report_worst_slack`: Critical timing summary
   - `report_tns/wns`: Violation metrics
   - `report_power`: Power consumption

4. **VSDBabySoC Results**
   - Typical corner: PASS (1.11ns setup slack)
   - Slow corners: FAIL (up to -52.9ns violation)
   - Hold timing: PASS across all corners

5. **PVT Analysis Importance**
   - Identifies worst-case scenarios
   - Ensures robustness across conditions
   - Guides optimization efforts
   - Required for silicon sign-off

### Next Steps

- **Fix setup violations** in slow corners
- **Perform CTS** (Clock Tree Synthesis) for realistic clock delays
- **Run post-route STA** with extracted parasitics
- **Validate at multiple frequencies** for different use cases

---

## Quick Command Reference

### Essential OpenSTA Commands
```bash
# Load files
read_liberty [-min|-max] <lib_file>
read_verilog <netlist>
read_sdc <constraints>
read_spef <parasitic_file>

# Design setup
link_design <top_module>
create_clock -period <ns> [get_ports <clk>]
set_input_delay -clock <clk> <delay> [get_ports <port>]

# Timing reports
report_checks [-path_delay min|max|min_max]
report_worst_slack [-min|-max]
report_tns
report_wns

# Advanced
check_setup -verbose
report_power
```

---
