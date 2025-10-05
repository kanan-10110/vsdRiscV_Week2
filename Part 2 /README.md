# VSDBabySoC Post-Synthesis Simulation Guide

<div align="center">

![VLSI](https://img.shields.io/badge/VLSI-Design-blue)
![Verilog](https://img.shields.io/badge/Verilog-HDL-green)
![Yosys](https://img.shields.io/badge/Yosys-Synthesis-orange)
![Status](https://img.shields.io/badge/Status-Active-success)

*A comprehensive guide for post-synthesis simulation and verification of VSDBabySoC*

</div>

---

## 📑 Table of Contents

- [Folder Organization](#-folder-organization)
- [Initial Configuration](#-initial-configuration)
- [Converting TLV to Standard Verilog](#-converting-tlv-to-standard-verilog)
- [Running Simulations](#-running-simulations)
- [Critical Signals for Waveform Analysis](#-critical-signals-for-waveform-analysis)
- [Waveform Analysis Expectations](#-waveform-analysis-expectations)
- [Sample Waveform Pattern](#-sample-waveform-pattern)
- [Suggested GTKWave Organization](#-suggested-gtkwave-organization)
- [Verification Success Criteria](#-verification-success-criteria)
- [Common Issues and Solutions](#-common-issues-and-solutions)
- [Key Takeaways](#-key-takeaways)
- [Required Software Versions](#-required-software-versions)

---

## 📁 Folder Organization

```txt
VSDBabySoC/
├── src/
│   ├── include/      # Verilog headers (*.vh)
│   ├── module/       # Design files (Verilog + TLV)
│   │   ├── vsdbabysoc.v   # Main integration module
│   │   ├── rvmyth.v       # Processor core
│   │   ├── avsdpll.v      # Phase-locked loop
│   │   ├── avsddac_stub.v      # Digital-to-analog converter
│   │   └── testbench.v    # Verification environment
└── output/           # Generated results
```

---

## 🚀 Initial Configuration

### 📥 Repository Download

```bash
cd VLSI
git clone https://github.com/manili/VSDBabySoC.git
cd VSDBabySoC
```

---

## 🔄 Converting TLV to Standard Verilog

The **RVMYTH** processor uses **TL-Verilog (.tlv)** format, requiring transformation to standard Verilog for simulation compatibility.

```bash
# Update package manager
sudo apt update
sudo apt install python3-venv python3-pip

# Initialize isolated Python workspace
python3 -m venv sp_env
source sp_env/bin/activate

# Add SandPiper conversion utility
pip install pyyaml click sandpiper-saas

# Execute TLV to Verilog transformation
sandpiper-saas -i ./src/module/*.tlv -o rvmyth.v --bestsv --noline -p verilog --outdir ./src/module/
```

This produces `rvmyth.v` in your module directory.

---

## 🎯 Running Simulations

### 🔹 Gate-Level Verification Workflow

Gate-level verification requires a synthesized netlist (vsdbabysoc.synth.v) generated through Yosys synthesis.

#### Step 1: Activate Virtual Environment

```bash
source sp_env/bin/activate   
```

> **🧠 Purpose:** Enables the Python virtual workspace named sp_env. Virtual workspaces provide isolated environments with dedicated Python interpreters and libraries (yosys, cocotb, matplotlib, etc.) separate from system-level installations.

#### Step 2: Launch Yosys

```bash
yosys
```

#### Step 3: Load Design Source Files

Load design source files into Yosys for processing and transformation:

```tcl
read_verilog -I ./src/include ./src/module/clk_gate.v
read_verilog -I ./src/include ./src/module/rvmyth.v
read_verilog -I ./src/include ./src/module/avsdpll_stub.v
read_verilog -I ./src/include ./src/module/avsddac_stub.v
read_verilog -I ./src/include ./src/module/vsdbabysoc.v
```

#### Step 4: Execute Synthesis

Execute synthesis targeting vsdbabysoc as the primary module:

```tcl
synth -top vsdbabysoc
```

#### Step 5: Prepare Output Directory

Exit Yosys and prepare output location:

```bash
exit
mkdir -p output/synthesized
```

#### Step 6: Export Netlist

Restart Yosys and export netlist:

```bash
yosys
```

```tcl
write_verilog output/synthesized/vsdbabysoc.synth.v
exit
```

**Yosys performs:**
- Creates gate-level netlist using basic components (gates, registers, primitives)
- Optionally flattens module hierarchy based on synthesis configuration
- Exports netlist to .v format

**Resulting folder layout:**

```txt
output/
 └── synthesized/
      └── vsdbabysoc.synth.v
```

#### Step 7: Gate-Level Verification

Close Yosys and begin gate-level verification:

```bash
iverilog -o output/post_synth_sim/post_synth_sim.out -DPOST_SYNTH_SIM \
    -I src/include -I src/module -I src/gls_model \
    src/module/testbench.v \
    output/synthesized/vsdbabysoc.synth.v \
    src/module/avsddac_stub.v
```

**⚠️ Encountered error message:**

```
src/gls_model/sky130_fd_sc_hd.v:67667: syntax error
src/gls_model/sky130_fd_sc_hd.v:67667: error: Invalid module item.
```

**To resolve**, identify the problematic line in sky130_fd_sc_hd.v:

```bash
nl -ba src/gls_model/sky130_fd_sc_hd.v | sed -n '67650,67680p'
```

Edit the file using vi:

```bash
vi sky130_fd_sc_hd.v
```

After correction, save and retry:

```bash
iverilog -o output/post_synth_sim/post_synth_sim.out -DPOST_SYNTH_SIM \
    -I src/include -I src/module -I src/gls_model \
    src/module/testbench.v \
    output/synthesized/vsdbabysoc.synth.v \
    src/module/avsddac_stub.v
```

Extract waveform database using:

```bash
vvp output/post_synth_sim/post_synth_sim.out
```
    
#### Step 8: Visualize Waveforms

Visualize gate-level waveforms with GTKWave:

```bash
gtkwave post_synth_sim.vcd
```

---

## 🔍 Critical Signals for Waveform Analysis

When analyzing `post_synth_sim.vcd` in **GTKWave**, monitor these essential signals:

| Signal | Function | Expected Pattern |
|---------|--------------|-------------------|
| `reset` | System initialization control | Initially high, transitions low to begin operation |
| `REF` | PLL reference frequency | Consistent square wave input |
| `CLK` | PLL-generated system clock | Stable periodic signal after lock acquisition |
| `RV_TO_DAC[9:0]` | Processor-to-DAC data bus | Active transitions showing computation results |
| `OUT` | DAC output signal | Mirrors `RV_TO_DAC` behavior |
| `ENb_VCO`, `ENb_CP` | PLL enable controls | Fixed values controlling PLL functionality |
| `VCO_IN` | PLL frequency input | Clock source for multiplication |

---

## 🧠 Waveform Analysis Expectations

Proper operation exhibits these **four distinct stages**:

### 🔹 Stage 1 — Initialization
- `reset` signal active
- `CLK` potentially inactive or unstable
- `RV_TO_DAC` and `OUT` remain quiet

### 🔹 Stage 2 — Clock Generation
- Following `reset` release, PLL (`avsdpll`) stabilizes
- `REF` and `VCO_IN` achieve synchronization
- `CLK` establishes steady oscillation

### 🔹 Stage 3 — Processor Activity
- `rvmyth` CPU begins instruction processing
- `RV_TO_DAC` exhibits active transitions
- `OUT` tracks `RV_TO_DAC` changes

### 🔹 Stage 4 — Normal Operation
- `CLK` maintains consistency
- Data signals change rhythmically
- Unknown (`X`) or floating (`Z`) states absent

---

## 📊 Sample Waveform Pattern

Simplified representation of expected **GTKWave** observations:

```
| Signal     |   Time → → →
|-------------|------------------------------------------
| reset       | ────────┐__________
| REF         | ──▁──▁──▁──▁──▁──▁──▁──▁──
| CLK         | ______▁______▁______▁______▁
| RV_TO_DAC   | 000000 → transitions → changing values
| OUT         | tracks RV_TO_DAC changes
```

---

## 🧩 Suggested GTKWave Organization

Arrange signals hierarchically for clarity:

```
vsdbabysoc/
 ├── reset
 ├── REF
 ├── CLK
 ├── RV_TO_DAC[9:0]
 ├── OUT
 ├── ENb_VCO
 ├── ENb_CP
 └── VCO_IN
```

**Adding procedure:**
1. Launch GTKWave
2. Select **SST → Search Signals**
3. Transfer signals to waveform display
4. Apply **"Zoom Fit"** or zoom controls for complete timeline view

---

## ✅ Verification Success Criteria

Correct implementation demonstrates:

- ✔️ Clean compilation without errors from `iverilog` or `vvp`
- ✔️ `CLK` begins periodic operation post-reset
- ✔️ `RV_TO_DAC` shows activity following initialization
- ✔️ `OUT` indicates DAC functionality
- ✔️ Absence of persistent `X` or `Z` conditions in GTKWave

---

## ⚠️ Common Issues and Solutions

| Problem | Root Cause | Resolution |
|-------|----------------|-----|
| `Unknown module avsddac_stub` | DAC stub not included | Add `src/module/avsddac_stub.v` to compilation |
| `No waveform transitions` | Clock failure or reset issue | Verify PLL configuration, reset behavior |
| `X or Z states continue` | Connection problems or undriven nets | Check module port connections |
| `CLK irregular` | PLL misconfiguration | Confirm `ENb_VCO` and `ENb_CP` settings |

---

## 🧾 Key Takeaways

- **Gate-level waveforms** should replicate RTL functionality with minor timing differences
- **Primary validation objective** ensures synthesized design maintains intended behavior
- Essential verifications:
  - ✅ Clock stability
  - ✅ Proper reset sequence
  - ✅ Data flow through `RV_TO_DAC` and `OUT`

---

## 🧰 Required Software Versions

| Software | Recommended Release |
|------|-------------------|
| Yosys | `0.32+` |
| Icarus Verilog | `11.0+` |
| GTKWave | `3.3.100+` |

---

## 📝 License

This project documentation is provided as-is for educational purposes.

---

## 🤝 Contributing

Contributions, issues, and feature requests are welcome! Feel free to check the issues page.

---

## 📧 Contact

For questions or support, please open an issue in the repository.

---

<div align="center">

Made with ❤️ for the VLSI community

</div>
