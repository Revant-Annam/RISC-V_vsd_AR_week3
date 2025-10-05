# VSDBabySoC: Synthesis and Gate-Level Simulation (WEEK 3- Part 1)

## Table of Contents
1. [Pre-Synthesis Functional Simulation Recap](#pre-synthesis-functional-simulation-recap)
2. [Synthesis: From RTL to Gates](#synthesis-from-rtl-to-gates)
3. [Post-Synthesis Simulation (GLS)](#post-synthesis-simulation-gls)

---

## Pre-Synthesis Functional Simulation Recap

Before proceeding with synthesis, the VSDBabySoC design was validated at the Register Transfer Level (RTL) during **Week 2**. This critical initial verification step, known as **pre-synthesis functional simulation**, ensures the logical correctness of the Verilog code by simulating the design's behavior with ideal, zero-delay assumptions for all components.

### Week 2 Simulation Results

The pre-synthesis simulation successfully demonstrated that the VSDBabySoC, which integrates the `rvmyth` RISC-V CPU core, correctly executed its test program. The processor generated a sequence of digital values that were transmitted to the Digital-to-Analog Converter (DAC). The DAC, in turn, produced the expected analog output.

This successful simulation established a reference for the design's intended functionality, against which all subsequent post-synthesis results would be compared.

### Pre-Synthesis (RTL) Simulation Waveform

  <img width="1920" height="1080" alt="Screenshot from 2025-10-04 16-22-44" src="https://github.com/user-attachments/assets/6107a2f2-8ece-43ea-86ce-de3ace828784" />

---

## Synthesis: From RTL to Gates

**Synthesis** is the automated transformation of a high-level RTL (Register Transfer Level) description into a gate-level netlist—a physical implementation composed entirely of standard cells from a specific technology library. For this project, we use the **SKY130 PDK** (Process Design Kit) and the open-source **Yosys** synthesis tool.

---

## Detailed Synthesis Process

The entire synthesis workflow is automated using a Yosys script located at:
```
VSDBabySoC/src/script/yosys.ys
```

## Synthesis Log -

### Step 1: Reading Input Design Files

The first phase involves loading all Verilog source files that describe the design's behavior.

####
```tcl
read_verilog ./module/vsdbabysoc.v
```
- Loads the top-level Verilog file that defines the complete VSDBabySoC system. This file instantiates and connects all major components: the RISC-V core, PLL, DAC, and any glue logic.

####
```tcl
read_verilog -I./include ../output/compiled_tlv/rvmyth.v
```
- Loads the `rvmyth` RISC-V CPU core, which is the computational heart of the SoC.
- `-I./include`: Instructs Yosys to search the `./include` directory for any header files that `rvmyth.v` reference using `` `include`` directives.

####
```tcl
read_verilog -I./include ./module/clk_gate.v
```
- Loads the clock gating logic, which is used to disable clock signals to inactive portions of the circuit to save power.
- `-I./include`: Ensures any dependencies are resolved from the include directory.

---

### Step 2: Reading Technology Libraries

Before synthesis can begin, Yosys needs detailed information about the physical characteristics of the available standard cells and macros. This information is provided in **Liberty (.lib)** format files.

####
```tcl
read_liberty -lib ./lib/avsdpll.lib
```
- Loads the timing, power, and functional characteristics of the Analog Voltage Scaled Digital Phase-Locked Loop (PLL) macro.

####
```tcl
read_liberty -lib ./lib/avsddac.lib
```

- Loads the characterization data for the Analog Voltage Scaled Digital-to-Analog Converter (DAC).

####
```tcl
read_liberty -lib ./lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
- Loads the complete SKY130 High-Density standard cell library for **typical** process corner, **25°C** temperature, and **1.8V** supply voltage.

 <img width="1920" height="1080" alt="Screenshot from 2025-10-03 04-09-01" src="https://github.com/user-attachments/assets/29498512-f05d-47da-9ab2-475917584d76" />

---

### Step 3: High-Level Synthesis

This is where the actual transformation from RTL to logic begins.

####
```tcl
synth -top vsdbabysoc
```
- Executes the main synthesis flow. Converts the RTL code into a synthesized design.

---

### Step 4: Technology Mapping

Now we map the generic logic to actual physical cells from the SKY130 library.

####
```tcl
dfflibmap -liberty ./lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
- Maps all generic D flip-flops (`$dff` cells) inferred during synthesis to actual flip-flop implementations from the SKY130 library.

####
```tcl
opt
```
- Runs a series of optimization passes on the partially mapped design.

####
```tcl
abc -liberty ./lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
- This is the **core technology mapping step** where combinational logic is mapped to actual SKY130 gates.
- This single command determines the area, delay, and power of your entire synthesized design. The quality of ABC's mapping directly impacts chip performance.

---

### Step 5: Netlist Finalization

After mapping is complete, the netlist needs to be cleaned up and prepared for output.

####
```tcl
flatten
```
- Removes all hierarchical boundaries, creating a single flat module containing all standard cells.
- To view the hierarchical design we can use `show vsdbabysoc` before the `flatten` command.

####
```tcl
setundef -zero
```

- Forces all signals with undefined (`x`) values to logic zero.

####
```tcl
clean -purge
```
- Performs a final, aggressive cleanup of the netlist.

####
```tcl
stat
```
- Generates a detailed report of the synthesized design.

**Report of the VSDBabySoC:**
```
Number of cells:  3542
  AND gates:      523
  OR gates:       412
  XOR gates:      89
  Flip-flops:     256
  Multiplexers:   178
  ...
Chip area:        15234.56 square microns
```

####
```tcl
write_verilog -noattr ../output/synth/vsdbabysoc.synth.v
```
- Exports the final synthesized netlist to a Verilog file.
- `-noattr`: Suppresses Yosys-specific attributes from being written to the file

**This file is the primary output of synthesis** and will be used for:
- Post-synthesis simulation (Gate-Level Simulation)
- Static Timing Analysis (STA)

---

## Post-Synthesis Simulation (GLS)

### What is Post-Synthesis Simulation?

**Post-synthesis simulation**, also known as **Gate-Level Simulation (GLS)**, is a verification technique that simulates the synthesized gate-level netlist rather than the original RTL code. Unlike RTL simulation, which uses abstract behavioral models, GLS uses the actual standard cell implementations from the technology library (SKY130).

**Key characteristics:**
- **Structural simulation:** Simulates the actual gates and flip-flops that will be fabricated on silicon
- **Technology-specific:** Uses models from the SKY130 PDK that match the physical characteristics of manufactured chips

---

### Why is Post-Synthesis Simulation Important?

GLS serves as a critical validation checkpoint in the digital design flow. Here's why it's indispensable:

#### 1. **Validates Synthesis Correctness** ✅
- Confirms that the synthesis tool correctly translated RTL to gates without introducing functional bugs
- Ensures that optimizations didn't inadvertently change the design's behavior
- Catches errors like incorrect FSM encoding or logic minimization mistakes

#### 2. **Detects Synthesis-Induced Problems** ⚠️
- **Unintended latches:** Synthesis may infer latches from incomplete `if-else` or `case` statements, causing unpredictable behavior
- **X-propagation:** Unknown (`x`) values that can corrupt the entire simulation
- **Optimization artifacts:** Rare cases where aggressive optimization creates incorrect logic
- **Combinational loops:** Feedback paths without registers that cause simulation to hang

---

### Pre-Synthesis vs. Post-Synthesis Simulation

| **Aspect** | **Pre-Synthesis Simulation (RTL)** | **Post-Synthesis Simulation (GLS)** |
|------------|-------------------------------------|--------------------------------------|
| **Design Representation** | Simulates high-level behavioral Verilog code (RTL) | Simulates structural gate-level netlist (actual standard cells) |
| **Abstraction Level** | Abstract: `always` blocks, `if-else`, arithmetic operators | Concrete: AND gates, OR gates, flip-flops, multiplexers |
| **Timing Model** | Zero-delay or unit-delay (ideal) | Realistic gate delays from technology library (.lib files) |
| **Delay Information** | No real delay information; focuses on logical correctness | Includes setup/hold times, propagation delays, clock-to-Q delays |
| **Primary Goal** | Verify functional correctness of the algorithm | Verify post-synthesis functional correctness AND check for timing issues |
| **Errors Detected** | Logic bugs, FSM errors, algorithmic mistakes, incorrect protocol implementation | Synthesis-induced bugs, timing violations, glitches, race conditions, X-propagation |
| **Models Used** | RTL Verilog files written by designer | Standard cell Verilog models from foundry (e.g., `sky130_fd_sc_hd.v`) |
| **Output Accuracy** | Logically correct but timing-optimistic | More accurate representation of actual silicon behavior |

---

### Gate-Level Simulation log -

The GLS process involves several preparatory steps followed by compilation and execution of the simulation.

---

#### Step 1: Copy Required Library Files

Before simulation, we need to gather all necessary files into the working directory.

##### Copy Standard Cell Verilog Models
```bash
cp ~/VLSI/sky130RTLDesignAndSynthesisWorkshop/my_lib/verilog_model/sky130_fd_sc_hd.v ~/VLSI/VSDBabySoC/src/module/
```
- This file contains the **behavioral Verilog descriptions** for every standard cell in the SKY130 High-Density library.

##### Copy Primitives Library
```bash
cp ~/VLSI/sky130RTLDesignAndSynthesisWorkshop/my_lib/verilog_model/primitives.v ~/VLSI/VSDBabySoC/src/module/
```
- This file defines low-level **primitive building blocks** used within the standard cell models.

##### Copy Synthesized Netlist
```bash
cp ~/VLSI/VSDBabySoC/output/synth/vsdbabysoc.synth.v ~/VLSI/VSDBabySoC/src/module/
```
- Copies the gate-level netlist (output of Yosys) into the directory where simulation will be run.

---

#### Step 2: Compile the Testbench

##### Compile with Icarus Verilog
```bash
iverilog -o ~/VLSI/VSDBabySoC/output/post_synth_sim/post_synth_sim.out \
    -DPOST_SYNTH_SIM \
    -DFUNCTIONAL \
    -DUNIT_DELAY=#1 \
    -I ~/VLSI/VSDBabySoC/src/include \
    -I ~/VLSI/VSDBabySoC/src/module \
    ~/VLSI/VSDBabySoC/src/testbench.v
```
- Compiles all Verilog files into a single executable simulation binary.

**Detailed flag explanation:**

##### `-o ~/VLSI/VSDBabySoC/output/post_synth_sim/post_synth_sim.out`
- Output file name
- Specifies the name and location of the compiled executable
- Without this flag, the default output would be `a.out` in the current directory

##### `-DPOST_SYNTH_SIM`
- Define a preprocessor macro named `POST_SYNTH_SIM` used in the testbench for conditional compilation
- When this macro is defined, the testbench includes the synthesized netlist else it includes the RTL code
- This allows the same testbench to be used for both pre- and post-synthesis simulation

##### `-DFUNCTIONAL`
- Define a macro named `FUNCTIONAL`
- Tells the SKY130 cell models to run in **functional mode** rather than full timing mode
- This simplifies the synthesis without worrying about the timings checks

##### `-DUNIT_DELAY=#1`
- Define a macro that sets a uniform delay of `#1` time unit for all gates
- Every gate in the design will have a propagation delay of 1 time unit

##### `-I ~/VLSI/VSDBabySoC/src/include`
- Add include directory to search path
- When the testbench has `` `include .vh``, Verilog will look in this directory

##### `-I ~/VLSI/VSDBabySoC/src/module`
- Add module directory to search path
- Iverilog can automatically find and compile included verilog files

##### `~/VLSI/VSDBabySoC/src/testbench.v`
- The top-level file to compile
- Iverilog starts here and recursively compiles all dependencies

---

#### Step 3: Run the Simulation

##### Navigate to Output Directory
```bash
cd ~/VLSI/VSDBabySoC/output/post_synth_sim/
```
- Move to the directory where the simulation executable resides.

##### Execute Simulation
```bash
./post_synth_sim.out
```
- Runs the compiled simulation executable.

---

#### Step 4: View Waveforms with GTKWave

##### Open Waveform Viewer
```bash
gtkwave post_synth_sim.vcd
```
- Launches the GTKWave waveform viewer to visually inspect simulation results.

---

### Post-Synthesis Simulation Waveform

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5a922412-a1f3-4ace-958a-6d3ac8c492a1" />

*The above waveform shows the VSDBabySoC behavior after synthesis, with realistic gate delays included.*

---

### Waveform Analysis and Comparison

#### Functional Correctness Verification ✅

The primary objective of post-synthesis simulation is to confirm that the synthesized gate-level netlist maintains the same functional behavior as the original RTL design.

**Key observations from waveform comparison:**
- The `RV_TO_DAC[9:0]` bus in the post-synthesis waveform shows the same sequential pattern as the pre-synthesis simulation
- The DAC's analog `OUT` signal produces a same waveform in both simulations

---

#### Comparison Summary Table

| **Aspect** | **Pre-Synthesis (RTL) Simulation** | **Post-Synthesis (GLS) Simulation** |
|------------|-------------------------------------|--------------------------------------|
| **Functional Behavior** | ✅ Correct waveform, incrementing and then decrementing digital values | ✅ Identical functional behavior maintained |
| **Signal Values** | Match expected sequence | Match expected sequence (same as RTL) |
| **Waveforms** |  |  |

---

