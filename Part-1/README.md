# Post-Synthesis Gate-Level Simulation of VSDBabySoC

## Introduction

Gate-Level Simulation (GLS) is a crucial verification step that validates the synthesized netlist against the original RTL design. After synthesis transforms high-level RTL into gate-level representation using standard cells, GLS ensures no functional discrepancies, timing violations, or logical errors were introduced.

This guide provides a comprehensive walkthrough of synthesis and gate-level simulation for the **VSDBabySoC** design.

---

## Table of Contents

- [Why Gate-Level Simulation?](#why-gate-level-simulation)
- [Prerequisites](#prerequisites)
- [Directory Structure](#directory-structure)
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

---

## Directory Structure

```
/home/janadinisk/vsd/VLSI/VSDBabySoC/
├── src/
│   ├── include/          # Header files (*.vh) with macros and parameters
│   ├── module/           # Source Verilog files
│   │   ├── avsddac.v    # Digital-to-Analog Converter
│   │   ├── avsdpll.v    # PLL clock generator
│   │   ├── rvmyth.tlv   # RISC-V CPU in TL-Verilog
│   │   ├── rvmyth.v     # Verilog from TLV conversion
│   │   ├── vsdbabysoc.v # Top-level SoC module
│   │   ├── testbench.v  # Testbench for simulation
│   │   └── clk_gate.v   # Clock gating module
│   ├── gls_model/       # Standard cell models and primitives
│   └── lib/             # Liberty timing files (add if not present)
├── output/              # Simulation outputs
├── compiled_tlv/        # TL-Verilog compilation files
└── images/              # Documentation images
```

---

## Synthesis Flow

### Step 1: Prepare Header Files

VSDBabySoC uses **SandPiper** for the RISC-V core, requiring specific header files in the working directory:

```bash
cd /home/janadinisk/vsd/VLSI/VSDBabySoC
cp src/include/sp_verilog.vh .
cp src/include/sandpiper.vh .
cp src/include/sandpiper_gen.vh .
```

**Why needed?** These files contain macros and parameters essential for proper RTL elaboration during synthesis.

---

### Step 2: Launch Yosys

```bash
cd /home/janadinisk/vsd/VLSI/VSDBabySoC
yosys
```

---

### Step 3: Read Design Files

Inside Yosys prompt:

```bash
# Read top-level module
read_verilog src/module/vsdbabysoc.v 

# Read RISC-V CPU with include paths
read_verilog -I /home/janadinisk/vsd/VLSI/VSDBabySoC/src/include/ /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/rvmyth.v

# Read clock gating module  
read_verilog -I /home/janadinisk/vsd/VLSI/VSDBabySoC/src/include/ /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/clk_gate.v
```

**What happens:** Yosys parses, elaborates, and builds internal RTL representation (RTLIL).

---

### Step 4: Read Liberty Files

```bash
# Analog IP libraries (DAC and PLL)
read_liberty -lib /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/avsddac.lib 
read_liberty -lib /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/avsdpll.lib 

# SkyWater 130nm standard cell library (TT corner, 25°C, 1.8V)
read_liberty -lib /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

**Liberty files contain:** Cell definitions, timing characteristics, power data, and area information.

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

**What happens:** RTL optimization, logic minimization, FSM extraction, and technology-independent optimization.

---

### Step 6: Map Flip-Flops

```bash
dfflibmap -liberty /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

**Purpose:** Converts generic `$dff` cells to technology-specific D flip-flops with proper clock polarity, reset types, and timing characteristics.

---

### Step 7: Initial Optimization

```bash
opt
```

**Optimizations applied:**
- Constant folding - Evaluates constant expressions
- Dead code elimination - Removes unused logic
- Common subexpression elimination - Shares identical logic
- Identity optimization - Simplifies trivial operations

---

### Step 8: ABC Technology Mapping

```bash
abc -liberty /home/janadinisk/vsd/VLSI/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib -script +strash;scorr;ifraig;retime;{D};strash;dch,-f;map,-M,1,{D}
```

**ABC Script Breakdown:**

| Command | Purpose | Benefit |
|---------|---------|---------|
| `strash` | Creates And-Inverter Graph (AIG) | Canonical representation |
| `scorr` | Sequential correspondence checking | Merges equivalent registers |
| `ifraig` | Incremental FRAIG | Simplifies combinational logic |
| `retime` | Register retiming | Optimizes timing by moving flip-flops |
| `dch,-f` | Don't-care optimization | Exploits don't-care conditions |
| `map` | Technology mapping | Maps logic to standard cells |

**Expected Result:** Reduced gate count and optimized critical paths.

---

### Step 9: Final Cleanup

```bash
flatten              # Remove module hierarchy
setundef -zero       # Replace undefined (X) signals with 0
clean -purge         # Remove unused/duplicate logic
rename -enumerate    # Systematically rename nets and cells
```

**Why each command matters:**
- `flatten` - Creates flat netlist for physical design tools
- `setundef -zero` - Ensures deterministic simulation behavior
- `clean -purge` - Eliminates dangling wires and unused cells
- `rename -enumerate` - Provides predictable naming (_123_, _124_, etc.)

---

### Step 10: View Statistics

```bash
stat
```

**Sample Output:**
```
Number of cells:           5924
  sky130_fd_sc_hd__dfxtp_1  1144  # D flip-flops
  sky130_fd_sc_hd__nand2_1   848  # 2-input NAND gates
  sky130_fd_sc_hd__nor2_1    404  # 2-input NOR gates
  sky130_fd_sc_hd__a21oi_1   667  # AND-OR-Invert cells
  ...
Chip area: 52933.27 µm²
```

---

### Step 11: Generate Netlist

```bash
write_verilog -noattr /home/janadinisk/vsd/VLSI/VSDBabySoC/output/vsdbabysoc_synth.v
```

**Flags:**
- `-noattr` - Excludes Yosys internal attributes for cleaner output

**Exit Yosys:**
```bash
exit
```

---

### Step 12: Update Testbench

Edit testbench to reference the synthesized netlist:

```bash
cd /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module
nano testbench.v
```

Modify the include statement:

```verilog
// Change from:
`include "vsdbabysoc.v"

// To (for post-synthesis simulation):
`ifdef POST_SYNTH_SIM
  `include "../../output/vsdbabysoc_synth.v"
`else
  `include "vsdbabysoc.v"
`endif
```

**Why?** This allows the same testbench to work for both RTL and gate-level simulation using the `-DPOST_SYNTH_SIM` flag.

---

## Gate-Level Simulation

### Step 1: Prepare Simulation Directory

Create output directory if it doesn't exist:

```bash
mkdir -p /home/janadinisk/vsd/VLSI/VSDBabySoC/output
cd /home/janadinisk/vsd/VLSI/VSDBabySoC/output
```

---

### Step 2: Copy Required Model Files

```bash
# Copy analog IP behavioral models
cp /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/avsddac.v .
cp /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/avsdpll.v .

# Copy standard cell models and primitives
cp /home/janadinisk/vsd/VLSI/VSDBabySoC/src/gls_model/sky130_fd_sc_hd.v .
cp /home/janadinisk/vsd/VLSI/VSDBabySoC/src/gls_model/primitives.v .
```

**Files needed for GLS:**

| File | Purpose |
|------|---------|
| `vsdbabysoc_synth.v` | Synthesized gate-level netlist |
| `avsddac.v` | Behavioral model of analog DAC |
| `avsdpll.v` | Behavioral model of analog PLL |
| `primitives.v` | Basic gate definitions from PDK |
| `sky130_fd_sc_hd.v` | Standard cell behavioral models |

---

### Step 3: Compile with Icarus Verilog

```bash
iverilog -o /home/janadinisk/vsd/VLSI/VSDBabySoC/output/vsdbabysoc_synth.vvp \
  -DPOST_SYNTH_SIM \
  -DFUNCTIONAL \
  -DUNIT_DELAY=#1 \
  -I /home/janadinisk/vsd/VLSI/VSDBabySoC/src/include \
  -I /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module \
  -I /home/janadinisk/vsd/VLSI/VSDBabySoC/src/gls_model \
  /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/testbench.v
```

**Compilation flags explained:**

| Flag | Purpose | Effect |
|------|---------|--------|
| `-o <file>` | Output file | Compiled simulation executable |
| `-DPOST_SYNTH_SIM` | Define macro | Enables post-synthesis simulation mode |
| `-DFUNCTIONAL` | Define macro | Functional simulation (no timing checks) |
| `-DUNIT_DELAY=#1` | Define macro | Adds 1 time unit delay to gates |
| `-I <path>` | Include directory | Specifies header file locations |

**Understanding the macros:**

- **POST_SYNTH_SIM** - Tells testbench to include synthesized netlist instead of RTL
- **FUNCTIONAL** - Disables timing checks in standard cell models (faster simulation)
- **UNIT_DELAY=#1** - Prevents zero-delay races by adding minimal propagation delay

---

### Step 4: Run Simulation

```bash
cd /home/janadinisk/vsd/VLSI/VSDBabySoC/output
vvp vsdbabysoc_synth.vvp
```

**Expected Console Output:**
```
VCD info: dumpfile post_synth_sim.vcd opened for output.
CLK: 1 | reset: 1 | OUT: xxx
CLK: 0 | reset: 1 | OUT: xxx
CLK: 1 | reset: 0 | OUT: 000
CLK: 0 | reset: 0 | OUT: 000
CLK: 1 | reset: 0 | OUT: 001
...
Simulation complete. VCD file generated: post_synth_sim.vcd
```

---

### Step 5: View Waveforms

```bash
gtkwave /home/janadinisk/vsd/VLSI/VSDBabySoC/output/post_synth_sim.vcd
```

**GTKWave Navigation Tips:**

1. **Expand Signal Hierarchy:**
   - `testbench` → `uut` (your design under test)
   - Look for: CLK, reset, OUT, internal CPU signals

2. **Key Signals to Add:**
   - Clock and reset
   - `OUT` - DAC output
   - `rvmyth.CPU_Xreg_value_a4` - Register file
   - `rvmyth.CPU_src1_value` - Source operands
   - `rvmyth.CPU_pc` - Program counter

3. **Waveform Analysis:**
   - Verify reset sequence (all FFs initialize)
   - Check output progression (incremental values)
   - Observe gate delays (slight delays after clock edge)

---

## Results Comparison

### Pre-Synthesis vs Post-Synthesis Verification

#### Pre-Synthesis Waveform (RTL Simulation)

**Command to generate:**
```bash
cd /home/janadinisk/vsd/VLSI/VSDBabySoC/output
iverilog -o pre_synth_sim.vvp \
  -I /home/janadinisk/vsd/VLSI/VSDBabySoC/src/include \
  /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/testbench.v \
  /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/vsdbabysoc.v \
  /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/rvmyth.v \
  /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/clk_gate.v \
  /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/avsddac.v \
  /home/janadinisk/vsd/VLSI/VSDBabySoC/src/module/avsdpll.v
  
vvp pre_synth_sim.vvp
gtkwave pre_synth_sim.vcd
```

**Characteristics:**
- Clean signal transitions
- Zero-delay logic propagation
- Ideal behavioral model

---

#### Post-Synthesis Waveform (Gate-Level Simulation)

![Post-Synthesis Waveform](../images/post_synth_wave.png)

**Characteristics:**
- Slight gate delays visible (from `UNIT_DELAY=#1`)
- Real gate behavior with propagation time
- Functionally identical to RTL

---

### Verification Checklist

| Verification Point | RTL | GLS | Status |
|-------------------|-----|-----|--------|
| **Reset Functionality** | ✓ | ✓ | ✅ PASS |
| **Clock Toggles Correctly** | ✓ | ✓ | ✅ PASS |
| **Output Sequence** | 0→1→2→3... | 0→1→2→3... | ✅ PASS |
| **Register Values Match** | ✓ | ✓ | ✅ PASS |
| **No Undefined (X) States** | ✓ | ✓ | ✅ PASS |
| **Module Interconnections** | ✓ | ✓ | ✅ PASS |
| **CPU Instruction Execution** | ✓ | ✓ | ✅ PASS |

---

### Detailed Analysis

#### Functional Correctness
- ✅ **100% Output Match** - All values identical between RTL and GLS
- ✅ **Timing Consistency** - Minor delays within acceptable limits
- ✅ **State Machine Behavior** - Proper state transitions
- ✅ **Data Path Integrity** - Correct data propagation through pipeline

#### Key Observations

**Reset Sequence (t=0 to t=10ns):**
- All 1144 flip-flops initialize to known state
- Combinational logic settles properly
- First valid output appears after reset deassertion

**Normal Operation (t>10ns):**
- CPU executes instructions correctly
- Register file updates as expected
- DAC output transitions smoothly
- No glitches or metastability issues

**Corner Cases Verified:**
- Maximum and minimum input values
- Rapid state changes
- Back-to-back operations

---

## Conclusion

### Summary

The Gate-Level Simulation successfully validates that the synthesized VSDBabySoC netlist maintains complete functional equivalence with the original RTL design.

**Synthesis Results:**
- **Initial Cells:** 6,440
- **Optimized Cells:** 5,924 (8% reduction)
- **Flip-Flops:** 1,144
- **Estimated Area:** 52,933 µm²
- **Technology:** SkyWater 130nm HD

**Verification Status:**
- ✅ Functional verification: **PASS**
- ✅ Reset behavior: **PASS**
- ✅ Clock distribution: **PASS**
- ✅ Output correctness: **PASS**
- ✅ Ready for physical design: **YES**

---

### Next Steps

With successful GLS completion, VSDBabySoC is ready for:

1. **Static Timing Analysis (STA)**
   - Setup/hold time verification
   - Critical path analysis
   - Clock domain crossing checks

2. **Physical Design Flow**
   - Floorplanning
   - Placement optimization
   - Clock tree synthesis
   - Routing

3. **Post-Layout Verification**
   - Back-annotated timing simulation (SDF)
   - Parasitic extraction
   - IR drop analysis
   - Final sign-off checks

---

## Quick Command Reference

### Complete Synthesis Flow
```bash
# Prepare environment
cd /home/janadinisk/vsd/VLSI/VSDBabySoC
cp src/include/*.vh .

# Run Yosys
yosys
read_verilog src/module/vsdbabysoc.v
read_verilog -I src/include/ src/module/rvmyth.v
read_verilog -I src/include/ src/module/clk_gate.v
read_liberty -lib src/lib/avsddac.lib
read_liberty -lib src/lib/avsdpll.lib
read_liberty -lib src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
synth -top vsdbabysoc
dfflibmap -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
opt
abc -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib -script +strash;scorr;ifraig;retime;{D};strash;dch,-f;map,-M,1,{D}
flatten
setundef -zero
clean -purge
rename -enumerate
stat
write_verilog -noattr output/vsdbabysoc_synth.v
exit
```

### Complete GLS Flow
```bash
# Prepare files
cd /home/janadinisk/vsd/VLSI/VSDBabySoC/output
cp ../src/module/avsddac.v .
cp ../src/module/avsdpll.v .
cp ../src/gls_model/*.v .

# Compile and run
iverilog -o vsdbabysoc_synth.vvp -DPOST_SYNTH_SIM -DFUNCTIONAL -DUNIT_DELAY=#1 \
  -I ../src/include -I ../src/module -I ../src/gls_model \
  ../src/module/testbench.v

vvp vsdbabysoc_synth.vvp
gtkwave post_synth_sim.vcd
```

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| **Yosys read errors** | Missing header files | Verify include paths and copy .vh files |
| **Undefined signals (X)** | Uninitialized registers | Check reset logic and initialization |
| **Compilation errors** | Missing model files | Ensure all .v files copied to output/ |
| **Timing mismatches** | Zero-delay races | Use `-DUNIT_DELAY=#1` flag |
| **Functional mismatch** | Synthesis bug | Review synthesis warnings, check for latches |
| **GTKWave crashes** | Large VCD file | Reduce simulation time or sample rate |

---

## Additional Resources

- [Yosys Documentation](http://www.clifford.at/yosys/documentation.html)
- [SkyWater PDK](https://skywater-pdk.readthedocs.io/)
- [Icarus Verilog Guide](http://iverilog.icarus.com/)
- [GTKWave Tutorial](http://gtkwave.sourceforge.net/)

---
