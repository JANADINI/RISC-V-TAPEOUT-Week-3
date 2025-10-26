
# Post-Synthesis Gate-Level Simulation of VSDBabySoC

## Introduction

Gate-Level Simulation (GLS) is a crucial verification step that validates the synthesized netlist against the original RTL design. After synthesis transforms high-level RTL into gate-level representation using standard cells, GLS ensures no functional discrepancies, timing violations, or logical errors were introduced.

This guide provides a comprehensive walkthrough of synthesis and gate-level simulation for the **VSDBabySoC** design.

---

## Table of Contents

- [Why Gate-Level Simulation?](#why-gate-level-simulation)
- [Prerequisites](#prerequisites)
- [Synthesis Flow](#synthesis-flow)
- [Gate-Level Simulation](#gate-level-simulation)
- [Results Comparison](#results-comparison)
- [Conclusion](#conclusion)

---

## Why Gate-Level Simulation?

GLS validates several critical aspects:

| Aspect | Purpose |
|--------|---------|
| **Functional Verification** | Confirms synthesized netlist matches RTL behavior |
| **Timing Validation** | Identifies setup/hold violations and race conditions |
| **Reset Check** | Ensures proper initialization from unknown states |
| **Clock Distribution** | Verifies correct clock/reset propagation |
| **Pre-Layout Sanity** | Final check before physical design stages |

---

## Prerequisites

### Required Tools
- **Yosys** (v0.58+) - Synthesis tool
- **Icarus Verilog** - Verilog simulator
- **GTKWave** - Waveform viewer

### Design Files Location
```
~/Desktop/VSDBabySoC/
├── src/
│   ├── module/       # RTL files
│   ├── include/      # Header files
│   ├── lib/         # Liberty files
│   └── gls_model/   # Gate-level models
```

---

## Synthesis Flow

### Step 1: Prepare Header Files

VSDBabySoC uses **SandPiper** for the RISC-V core, requiring specific header files:

```bash
cd ~/Desktop/VSDBabySoC
cp src/include/sp_verilog.vh .
cp src/include/sandpiper.vh .
cp src/include/sandpiper_gen.vh .
```

**Why needed?** These files contain macros and parameters for proper RTL elaboration.

---

### Step 2: Launch Yosys

```bash
yosys
```

---

### Step 3: Read Design Files

Inside Yosys prompt:

```bash
# Read top-level module
read_verilog src/module/vsdbabysoc.v 

# Read RISC-V CPU with include paths
read_verilog -I ~/Desktop/VSDBabySoC/src/include/ ~/Desktop/VSDBabySoC/src/module/rvmyth.v

# Read clock gating module  
read_verilog -I ~/Desktop/VSDBabySoC/src/include/ ~/Desktop/VSDBabySoC/src/module/clk_gate.v
```

**What happens:** Yosys parses, elaborates, and builds internal RTL representation.

---

### Step 4: Read Liberty Files

```bash
# Analog IP libraries
read_liberty -lib ~/Desktop/VSDBabySoC/src/lib/avsddac.lib 
read_liberty -lib ~/Desktop/VSDBabySoC/src/lib/avsdpll.lib 

# SkyWater 130nm standard cell library (TT corner, 25°C, 1.8V)
read_liberty -lib ~/Desktop/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

**Liberty files contain:** Cell definitions, timing data, power info, and area specifications.

---

### Step 5: Synthesize Design

```bash
synth -top vsdbabysoc
```

**Expected Output:**
```
=== vsdbabysoc ===
Number of wires:          6551
Number of cells:          6440
  avsddac                    1
  avsdpll                    1
  sky130_fd_sc_hd__*      6438
```

---

### Step 6: Map Flip-Flops

```bash
dfflibmap -liberty ~/Desktop/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

**Purpose:** Maps generic flip-flops to technology-specific D flip-flops.

---

### Step 7: Optimize

```bash
opt
```

**Optimizations applied:**
- Constant folding
- Dead code elimination
- Common subexpression elimination
- Identity simplification

---

### Step 8: ABC Technology Mapping

```bash
abc -liberty ~/Desktop/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib -script +strash;scorr;ifraig;retime;{D};strash;dch,-f;map,-M,1,{D}
```

**ABC Script Breakdown:**

| Command | Purpose |
|---------|---------|
| `strash` | Creates And-Inverter Graph (AIG) |
| `scorr` | Merges equivalent registers |
| `ifraig` | Simplifies combinational logic |
| `retime` | Redistributes registers for timing |
| `dch,-f` | Don't-care optimization |
| `map` | Maps to library cells |

---

### Step 9: Final Cleanup

```bash
flatten              # Remove hierarchy
setundef -zero       # Replace X with 0
clean -purge         # Remove unused logic
rename -enumerate    # Systematic renaming
```

---

### Step 10: View Statistics

```bash
stat
```

**Sample Output:**
```
Number of cells:           5924
  sky130_fd_sc_hd__dfxtp_1  1144  # Flip-flops
  sky130_fd_sc_hd__nand2_1   848
  sky130_fd_sc_hd__nor2_1    404
  ...
Chip area: 52933.27 µm²
```

---

### Step 11: Generate Netlist

```bash
write_verilog -noattr ~/Desktop/vsdbabysoc_synth.v
```

**Exit Yosys:**
```bash
exit
```

---

### Step 12: Update Testbench

Edit testbench to use synthesized netlist:

```bash
cd ~/Desktop/VSDBabySoC/src/module
nano testbench.v
```

Change:
```verilog
`include "vsdbabysoc.synth.v"    // Old
```
To:
```verilog
`include "vsdbabysoc_synth.v"    // New
```

---

## Gate-Level Simulation

### Step 1: Copy Required Files

```bash
cd ~/Desktop
cp ~/Desktop/VSDBabySoC/src/module/avsddac.v .
cp ~/Desktop/VSDBabySoC/src/module/avsdpll.v .
cp ~/Desktop/VSDBabySoC/src/gls_model/sky130_fd_sc_hd.v .
cp ~/Desktop/VSDBabySoC/src/gls_model/primitives.v .
```

**Files needed:**
- `vsdbabysoc_synth.v` - Synthesized netlist
- `avsddac.v`, `avsdpll.v` - Analog IP models
- `primitives.v` - Basic gate definitions
- `sky130_fd_sc_hd.v` - Standard cell models

---

### Step 2: Compile with iverilog

```bash
iverilog -o ~/Desktop/vsdbabysoc_synth.vvp \
  -DPOST_SYNTH_SIM \
  -DFUNCTIONAL \
  -DUNIT_DELAY=#1 \
  -I ~/Desktop/VSDBabySoC/src/include \
  -I ~/Desktop/VSDBabySoC/src/module \
  -I ~/Desktop/VSDBabySoC/src/gls_model \
  ~/Desktop/VSDBabySoC/src/module/testbench.v
```

**Flags explained:**

| Flag | Purpose |
|------|---------|
| `-DPOST_SYNTH_SIM` | Enables post-synthesis mode in testbench |
| `-DFUNCTIONAL` | Functional simulation (no timing checks) |
| `-DUNIT_DELAY=#1` | Adds minimal gate delay (1 time unit) |
| `-I <path>` | Include directories for headers |

---

### Step 3: Run Simulation

```bash
vvp ~/Desktop/vsdbabysoc_synth.vvp
```

**Expected Output:**
```
VCD info: dumpfile post_synth_sim.vcd opened for output.
CLK: 1 | reset: 1 | OUT: xxx
CLK: 0 | reset: 1 | OUT: xxx
CLK: 1 | reset: 0 | OUT: 000
CLK: 0 | reset: 0 | OUT: 000
...
```

---

### Step 4: View Waveforms

```bash
gtkwave ~/Desktop/post_synth_sim.vcd
```

**Key signals to observe:**
- `CLK` - Clock signal
- `reset` - Reset signal
- `OUT` - DAC output
- `rvmyth.CPU_Xreg_value` - Register file
- Internal CPU signals

---

## Results Comparison

### Pre-Synthesis vs Post-Synthesis

#### Pre-Synthesis Waveform (RTL Simulation)
![Pre-Synthesis Waveform](images/pre_synth_wave.png)

**Characteristics:**
- Clean signal transitions
- Zero-delay logic propagation
- Ideal behavior

#### Post-Synthesis Waveform (Gate-Level Simulation)
![Post-Synthesis Waveform](images/post_synth_wave.png)

**Characteristics:**
- Slight gate delays visible (from `UNIT_DELAY=#1`)
- Functionally identical outputs
- Real gate behavior

### Verification Checklist

| Check | RTL | GLS | Status |
|-------|-----|-----|--------|
| Reset functionality | ✓ | ✓ | ✅ PASS |
| Clock toggles | ✓ | ✓ | ✅ PASS |
| Output sequence | 0,1,2,3... | 0,1,2,3... | ✅ PASS |
| Register values | ✓ | ✓ | ✅ PASS |
| No undefined states | ✓ | ✓ | ✅ PASS |

### Analysis

- ✅ **Functional Match:** 100% - All outputs identical between RTL and GLS
- ✅ **Timing Behavior:** Minimal delays within acceptable limits  
- ✅ **Module Interconnections:** CPU, DAC, and PLL work correctly
- ✅ **Ready for Next Steps:** STA, Place & Route, Post-Layout Simulation

---

## Conclusion

The Gate-Level Simulation successfully validates that the synthesized VSDBabySoC netlist matches the RTL design's functionality. 

**Key Results:**
- **Total Cells:** 5,924 (optimized from 6,440)
- **Flip-Flops:** 1,144
- **Estimated Area:** ~52,933 µm²
- **Functional Verification:** ✅ PASS

**Design is now ready for:**
1. Static Timing Analysis (STA)
2. Floorplanning and Placement
3. Clock Tree Synthesis (CTS)
4. Routing
5. Post-layout verification

---

## Quick Command Reference

### Synthesis
```bash
yosys
read_verilog <files>
read_liberty -lib <lib_files>
synth -top vsdbabysoc
dfflibmap -liberty <lib>
opt
abc -liberty <lib> -script <script>
flatten; setundef -zero; clean -purge; rename -enumerate
write_verilog -noattr output.v
```

### Gate-Level Simulation
```bash
iverilog -o output.vvp -DPOST_SYNTH_SIM -DFUNCTIONAL -DUNIT_DELAY=#1 -I <includes> testbench.v
vvp output.vvp
gtkwave output.vcd
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Synthesis errors | Check header files and include paths |
| Undefined signals (X) | Verify reset logic and initialization |
| Timing mismatches | Add `UNIT_DELAY` or use SDF annotation |
| Compilation errors | Ensure all model files are copied |

---
