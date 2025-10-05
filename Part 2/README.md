<div align="center">
 
# Week 2 : Part 2
# BabySoC Core Concepts & Behavioral Testing

</div>

<div align="center">
 
[![RISC-V](https://img.shields.io/badge/RISC--V-SoC%20Tapeout-blue?style=for-the-badge&logo=riscv)](https://riscv.org/)
[![VSD](https://img.shields.io/badge/VSD-Program-orange?style=for-the-badge)](https://vsdiat.vlsisystemdesign.com/)
![Week](https://img.shields.io/badge/Week-2-pink?style=for-the-badge)

</div>

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

## 🚀 Initial Configuration

### 📥 Repository Download
```
cd VLSI
git clone https://github.com/manili/VSDBabySoC.git
cd VSDBabySoC
```

## 🔄 Converting TLV to Standard Verilog

The **RVMYTH** processor uses **TL-Verilog (.tlv)** format, requiring transformation to standard Verilog for simulation compatibility.

```
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


## 🎯 Running Simulations
## 🔹 Behavioral-Level Verification
Since Icarus Verilog & GTKWave are already installed, we can proceed directly with verification
```
mkdir -p output/pre_synth_sim

iverilog -o output/pre_synth_sim/pre_synth_sim.out \
  -DPRE_SYNTH_SIM \
  -I src/include -I src/module \
  src/module/testbench.v
cd output/pre_synth_sim
./pre_synth_sim.out
```
To examine the simulation results, launch the vcd file in GTKWave
```
gtkwave pre_synth_sim.vcd
```
<img width="1853" height="754" alt="Pre-Synth GTK Wave" src="https://github.com/user-attachments/assets/4359b636-a21f-4485-b8ef-13fa858ce77f" />
<img width="677" height="147" alt="Pre Synth Log " src="https://github.com/user-attachments/assets/302bae54-8f8d-45ad-a12d-3599da218ed2" />


🧠 Behavioral-Level Simulation Signal Inspection  


---

### 📘 Introduction
This documentation delivers comprehensive insights into signal patterns captured during **behavioral-level verification** of the `vsdbabysoc` architecture.  
The verification encompasses processor core, digital-to-analog interface, and frequency synthesis components integrated within the system.  
Signal patterns were examined through GTKWave visualization to validate proper system functionality.

---

### 🧩 Design Hierarchy
```
vsdbabysoc_tb
└── uut
    ├── core    → Processor core with memory interface  
    ├── dac     → Digital-to-analog interface  
    └── pll     → Frequency synthesis unit  
```

---

### ⚙️ Signal Pattern Analysis from Verification

### Detailed Signal Examination

#### 1. **CLK (Primary Clock)**
- **Type:** `reg`  
- **Purpose:** Main reference oscillator controlling synchronous components.  
- **Pattern:** Consistent, uniform square wave (~50 ns cycle).  

---

#### 2. **CPU_dmem_addr_a4[3:0]**
- **Type:** 4-bit bus  
- **Purpose:** Processor data memory address pathways.  
- **Activity:** Rapid switching during memory transaction periods, demonstrating active instruction processing.  

---

#### 3. **OUT[9:0]**
- **Type:** 10-bit bus  
- **Purpose:** Digital samples from DAC input buffer.  
- **Activity:**  
  - Indeterminate (`x`) during initialization.  
  - Cyclical ramp-like sequence after initialization — demonstrates signal synthesis.  

---

#### 4. **reset**
- **Type:** `wire`  
- **Purpose:** System-level asynchronous initialization (active HIGH).  
- **Activity:**  
  - Active during startup (~0–200 ns).  
  - Released subsequently to commence standard operations.  

---

#### 5. **Dext[10:0]**
- **Type:** 11-bit bus  
- **Purpose:** External or bridging data pathway supplying DAC.  
- **Activity:**  
  - Indeterminate at initialization.  
  - Stabilizes to legitimate sample data after initialization.  

---

#### 6. **EN (Activation Signal)**
- **Type:** `wire`  
- **Purpose:** Universal activation control for operational components.  
- **Activity:** Remains HIGH throughout active phases.  

---

#### 7. **NaN (Error Indicator)**
- **Type:** `wire`  
- **Purpose:** Flag for invalid or indeterminate analog calculations.  
- **Activity:** Persistent `nan`.  

---

#### 8. **VREFH / VREFL**
- **Type:** `wire`  
- **Purpose:** DAC reference thresholds (upper/lower).  
- **Activity:**  
  - `VREFH` = Fixed `1`  
  - `VREFL` = Fixed `0`  

---

#### 9. **OUT (Analog Signal)**
- **Type:** `real`  
- **Purpose:** DAC analog output pathway.  
- **Activity:** Continuous cyclical pattern, validating digital-to-analog transformation.  

---

#### 10. **PLL Component Signals**

| Signal | Type | Purpose | Activity |
|:-------|:------|:-------------|:-----------|
| **CLK** | `reg` | Internal PLL clock pathway | Elevated-frequency scaled version of input oscillator |
| **ENb_CP** | `wire` | Charge pump activation (active low) | Low → Operational; demonstrates PLL feedback operation |
| **ENb_VCO** | `wire` | VCO activation (active low) | Fixed low, maintaining continuous functionality |
| **REF** | `wire` | PLL reference oscillator | Cyclical slower pulse employed for phase correlation |
| **VCO_IN** | `wire` | Voltage Controlled Oscillator input | Rapid cyclical pattern managed by loop feedback |
| **lastedge** | `real` | Timestamp of previous clock transition | Advances by ~283 ns between transitions |
| **period** | `real` | Calculated signal cycle | Stabilizes near 35.4 ns following lock |
| **refpd** | `real` | Reference phase correlator output | Stabilizes at ~283.33 ns — demonstrates phase synchronization |

---

#### 🔁 Operational Stages

##### 1. **Initialization Stage (0 – ~200 ns)**
- All digital pathways = `x` (indeterminate)
- PLL and DAC inactive
- No legitimate pattern generation

##### 2. **Activation Stage (~200 ns – 40 µs)**
- Initialization released
- CPU initiates address sequences (`CPU_dmem_addr_a4`)
- PLL commences phase synchronization (`refpd` and `period` stabilizing)
- DAC initiates digital-to-analog transformations

##### 3. **Stable-State Stage (> 40 µs)**
- PLL attains lock (consistent `period` and `refpd`)
- DAC produces uniform analog pattern
- CPU, PLL, and DAC fully coordinated

---

#### 📈 Operational Flow Overview

```
CPU → Dext[10:0] → DAC → OUT[9:0] → OUT(real)
          ↑
          └──── PLL coordinates via REF & VCO_IN
```

**System Activity:**
- CPU produces digital sample sequences.
- DAC transforms them to analog pattern.
- PLL maintains consistent timing and frequency management.
- The resulting output is a consistent, cyclical analog signal.

---

#### ✅ Primary Findings

- **PLL Lock Confirmed:** `refpd` and `period` stabilize at stable-state.  
- **DAC Output Confirmed:** Digital samples effectively produce analog pattern.  
- **CPU Functionality Validated:** Operational memory addressing and data movement evident.  
- **System Coordination Achieved:** All components function harmoniously after initialization.

---

#### 📂 Resources
| File | Purpose |
|:------|:-------------|
| `pre_synth_sim.vcd` | Value Change Database for GTKWave |
| `Pre_Synth GTK Wave.png` | Waveform visualization screenshot |
| `vsdbabysoc_tb.v` | Primary testbench environment |
| `uut/core.v`, `uut/dac.v`, `uut/pll.v` | Components under verification |

---

#### 🛠️ Development Environment
- **Simulation Tool:** Icarus Verilog  
- **Pattern Viewer:** GTKWave  
- **Platform:** Ubuntu Linux  
- **Verification Date:** October 5, 2025  

---

#### 🧾 Conclusion
This behavioral-level pattern analysis validates proper operational coordination between the processor, DAC, and PLL components of the `vsdbabysoc` architecture.  
The architecture effectively transitions from initialization to standard operation, attains PLL lock, and generates a legitimate analog output pattern.


## 🔹 Gate-Level Verification

For gate-level verification, we require a vsdbabysoc.synth.v file generated through synthesis using Yosys

#### 1.
```
source sp_env/bin/activate   
```
##### 🧠 Purpose
This instruction enables a Python virtual workspace named sp_env.
A virtual workspace is a separated Python environment — containing its own Python interpreter, together with designated packages (like yosys, cocotb, matplotlib, etc.) that remain isolated from your system-level Python configuration.

#### 2. Launch Yosys
```
yosys
```
#### 3. Initially we load all verilog design sources into Yosys for processing and transformation
```
read_verilog -I ./src/include ./src/module/clk_gate.v
read_verilog -I ./src/include ./src/module/rvmyth.v
read_verilog -I ./src/include ./src/module/avsdpll_stub.v
read_verilog -I ./src/include ./src/module/avsddac_stub.v
read_verilog -I ./src/include ./src/module/vsdbabysoc.v
```

<img width="1855" height="1075" alt="Reading all Verilog Files" src="https://github.com/user-attachments/assets/cf9f704b-90b0-494b-adbd-5569f6847643" />


#### 4. Next we execute the transformation process in Yosys targeting the module vsdbabysoc as the primary architecture.
```
synth -top vsdbabysoc
```
<img width="658" height="854" alt="Synthezing vsdbabysoc" src="https://github.com/user-attachments/assets/94eb1cb7-fd57-455e-a6f5-795a90364b4c" />


#### 5. Exit Yosys and prepare directory structure
```
mkdir -p output/synthesized
```
#### 6. Restart Yosys and execute
```
yosys
write_verilog output/synthesized/vsdbabysoc.synth.v
```

<img width="742" height="284" alt="Writing the netlist" src="https://github.com/user-attachments/assets/fa10a5d1-af8f-49cf-911c-4fcc08f1ebbe" />


##### Yosys executes the following:
- i). Produces a Verilog netlist — composed of fundamental gates, registers, and basic elements.
- ii). Simplifies the hierarchy (conditional) — based on transformation settings, submodules might be consolidated.
- iii). Exports that netlist into a .v format.

##### Folder structure following execution
```
output/
 └── synthesized/
      └── vsdbabysoc.synth.v
```
#### 7. Close Yosys and proceed with gate-level verification
```
iverilog -o output/post_synth_sim/post_synth_sim.out -DPOST_SYNTH_SIM \
    -I src/include -I src/module -I src/gls_model \
    src/module/testbench.v \
    output/synthesized/vsdbabysoc.synth.v \
    src/module/avsddac_stub.v
```

##### Upon execution we encountered an error: 
```
src/gls_model/sky130_fd_sc_hd.v:67667: syntax error
src/gls_model/sky130_fd_sc_hd.v:67667: error: Invalid module item.
```

<img width="1811" height="334" alt="Error at line 67667 " src="https://github.com/user-attachments/assets/c3102b60-9419-4bf4-a9b9-7a8eb0d4fe62" />


To correct the error, initially identified the problematic line within the sky130_fd_sc_hd.v file through the instruction:
```
nl -ba src/gls_model/sky130_fd_sc_hd.v | sed -n '67650,67680p'
```
Subsequently accessed the file in the vi editor through the instruction
```
vi sky130_fd_sc_hd.v
```
###### Prepare directory
```
mkdir -p output/post_synth_sim
```

##### Following modifications, preserved the file and re-executed the instruction

```
iverilog -o output/post_synth_sim/post_synth_sim.out -DPOST_SYNTH_SIM \
  -I src/include -I src/module -I src/gls_model \
  src/module/testbench.v \
  output/synthesized/vsdbabysoc.synth.v \
  src/module/avsddac_stub.v
```

##### Next we generate the vcd file identifier through the instruction:
```
vvp output/post_synth_sim/post_synth_sim.out
```

<img width="798" height="99" alt="Generated VCD File" src="https://github.com/user-attachments/assets/14b8d82b-6d3a-4930-b3f9-f0866a85afa7" />

    
#### 8. Launch GTKWave for gate-level analysis through the instruction:
```
gtkwave post_synth_sim.vcd
```

<img width="1855" height="558" alt="Post_Synthesis GTK Wave" src="https://github.com/user-attachments/assets/45532b31-be57-4258-a9f2-ead49e5bbd94" />


### 🧩 Gate-Level Signal Pattern Analysis — VSDBabySoC


---

#### 🧠 Introduction

The waveform illustrates the coordination between the **RISC-V processor (rvmyth)**, **frequency synthesis unit**, and **digital-to-analog interface** following synthesis transformation.

**Hierarchy in GTKWave:**
```
vsdbabysoc_tb
└── uut
    ├── core
    ├── dac
    └── pll
```

The signals validate that:
- The frequency synthesis unit is locked and producing a consistent output oscillator.
- The processor core is operational, generating addresses and sample data.
- The digital-to-analog interface outputs digital sequences consistent with analog pattern generation.

---

#### 🧩 Signal Observations

##### **1. CLK (Primary Oscillator)**
**Type**: `reg`  
**Pattern**: Consistent cyclical square wave  
**Finding**:  
- Operates continuously with uniform cycle.
- Powers the complete architecture.

---

##### **2. reset**
**Type**: `wire`  
**State**: `0` (released)  
**Finding**:  
- System is executing normally (initialization released).

---

##### **3. CPU_dmem_addr_a4[3:0]**
**Type**: 4-bit bus  
**States displayed**: `8`, `F`, `B`, `A`, `6`, `2`, `E`, `3`, ...  
**Finding**:  
- Processor is dynamically producing memory addresses.
- Validates proper core operation and memory transactions.

---

##### **4. D[9:0]**
**Type**: 10-bit bus  
**States displayed**: `0AB`, `0BE`, `099`, `088`, `078`  
**Finding**:  
- Represents digital samples transmitted to the digital-to-analog interface.
- Progressive reduction sequence demonstrates pattern shaping activity.

---

##### **5. OUT**
**Type**: wire  
**State**: `z` (floating / indeterminate at this moment)  
**Finding**:  
- Potentially represents the analog interface output prior to stabilization.

---

##### **6. VREFH**
**Type**: wire  
**State**: `1`  
**Finding**:  
- Upper reference threshold for digital-to-analog interface.
- Demonstrates proper voltage biasing.

---

##### **7. Frequency Synthesis Unit Signal Patterns**

###### **CLK (Frequency Synthesis Output Oscillator)**
- Consistent, elevated-frequency square wave.
- Powers coordinated logic components.

###### **ENb_CP (Activate Charge Pump)**
- State: `x` (indeterminate or released)
- Potentially demonstrates released control pathway during stable state.

###### **ENb_VCO (Activate VCO)**
- State: `1`
- VCO operational, frequency synthesis unit functioning in locked state.

###### **REF**
- State: `1`
- Reference input to the frequency synthesis unit for phase correlation.

###### **VCO_IN**
- State: `1`
- Represents operational internal oscillation from the VCO.

###### **lastedge**
- State: `42032.84 ns`
- Records timing of the previous detected transition of the VCO.

###### **period**
- State: `35.41625 ns`
- Demonstrates frequency synthesis unit has stabilized with uniform clock cycle.

###### **refpd**
- State: `283.33 ns`
- Demonstrates a fixed phase variance, validating frequency synthesis unit lock.

---

#### 🧭 Primary Findings

| Signal | Purpose | State/Activity |
|---------|--------------|----------------|
| **CLK** | Primary oscillator | Consistent, cyclical |
| **reset** | Initialization input | Released (`0`) |
| **CPU_dmem_addr_a4[3:0]** | Processor address bus | Operational transitions |
| **D[9:0]** | Digital-to-analog interface digital input | Sequential digital sequence |
| **OUT** | Digital-to-analog interface output | Floating (`z`) |
| **VREFH** | Digital-to-analog interface ref upper | Fixed upper |
| **ENb_VCO** | VCO activation | Operational (`1`) |
| **period** | Frequency synthesis unit clock cycle | `35.416 ns` |
| **refpd** | Frequency synthesis unit phase correlator output | `283.33 ns` |
| **Frequency synthesis unit lock** | Condition | Attained (consistent period + refpd) |

---

#### 🧩 Operational Flow

```
CPU (rvmyth)
   ↓
CPU_dmem_addr_a4[3:0] → Data Storage
   ↓
D[9:0] → Digital-to-analog Interface Input
   ↓
OUT → Analog Output
```

Concurrently, the frequency synthesis unit produces a consistent, elevated-frequency system oscillator:

```
REF → Phase Correlator → Charge Pump → VCO → CLK_out
```

---

#### ✅ Overview

| Feature | Condition | Observations |
|----------|---------|-------|
| Initialization handling | ✅ | Initialization is released; architecture operational |
| Processor functionality | ✅ | Memory addresses and sample data operational |
| Frequency synthesis unit locking | ✅ | Consistent cycle = 35.416 ns |
| Digital-to-analog interface signals | ✅ | Appropriate digital pattern |
| VREF signals | ✅ | Voltage bias levels appropriate |
| Verification accuracy | ✅ | Aligns with anticipated gate-level activity |



#### ⚠️ Common Issues and Solutions

| Problem | Root Cause | Resolution |
|-------|----------------|-----|
| `Unknown module avsddac_stub` | Digital-to-analog interface stub not included | Add `src/module/avsddac_stub.v` to compilation |
| `No waveform transitions` | Oscillator failure or initialization issue | Verify frequency synthesis unit configuration, initialization activity |
| `X or Z states continue` | Interconnection problems or unconnected pathways | Examine module port interconnections |
| `CLK irregular` | Frequency synthesis unit misconfiguration | Validate `ENb_VCO` and `ENb_CP` configurations |

---

#### 🧾 Conclusion

- The **gate-level waveform** should replicate behavioral-level operation with slight propagation variances.
- The **primary validation objective** is that synthesized architecture maintains functionality following synthesis.
- Essential validations:
  - ✅ Oscillator consistency
  - ✅ Proper initialization sequence
  - ✅ Signal flow through `RV_TO_DAC` and `OUT`

---

## 🧰 Development Environment

| Software | Recommended Release |
|------|-------------------|
| Yosys | `0.32+` |
| Icarus Verilog | `11.0+` |
| GTKWave | `3.3.100+` |
