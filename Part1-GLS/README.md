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

<img width="1920" height="1080" alt="Screenshot from 2025-10-05 18-02-50" src="https://github.com/user-attachments/assets/caa15e55-803c-4016-ad3c-24d81876c163" />


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

<img width="1920" height="1080" alt="Screenshot from 2025-10-05 18-04-01" src="https://github.com/user-attachments/assets/94df4d47-8302-476a-98a3-ac4583cbf24c" />


---

### Step 3: High-Level Synthesis

This is where the actual transformation from RTL to logic begins.

####
```tcl
synth -top vsdbabysoc
```
- Executes the main synthesis flow. Converts the RTL code into a synthesized design.

<img width="1920" height="1080" alt="Screenshot from 2025-10-05 18-05-41" src="https://github.com/user-attachments/assets/6a35dbbf-a8fb-486b-b464-65b7a9a8730f" />

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3b10361a-27ee-46a2-be08-ca2554be0507" />

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/85db41d2-e54b-4e2e-9903-26722c6fd290" />


---

### Step 4: Technology Mapping

Now we map the generic logic to actual physical cells from the SKY130 library.

####
```tcl
dfflibmap -liberty ./lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
- Maps all generic D flip-flops (`$dff` cells) inferred during synthesis to actual flip-flop implementations from the SKY130 library.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/107953bc-f5b0-471c-a74f-288319a3a4e3" />


####
```tcl
opt
```
- Runs a series of optimization passes on the partially mapped design.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5034f875-111c-4496-9e82-fe4ab934e845" />


####
```tcl
abc -liberty ./lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
- This is the **core technology mapping step** where combinational logic is mapped to actual SKY130 gates.
- This single command determines the area, delay, and power of your entire synthesized design. The quality of ABC's mapping directly impacts chip performance.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/563bde01-7481-435e-a213-9dbb232088a5" />


<img width="1909" height="760" alt="image" src="https://github.com/user-attachments/assets/e7c9a7e6-8123-4c22-a49e-d0bd7840d221" />


---

### Step 5: Netlist Finalization

After mapping is complete, the netlist needs to be cleaned up and prepared for output.

####
```tcl
flatten
```
- Removes all hierarchical boundaries, creating a single flat module containing all standard cells.
- To view the hierarchical design we can use `show vsdbabysoc` before the `flatten` command.

The hierarchical synthesized output:

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/00b4df10-aa77-42e5-85d1-e4399b44b59c" />


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

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/eaeb7d8b-15da-43ea-b423-fae23f7da1d1" />


####
```tcl
stat
```
- Generates a detailed report of the synthesized design.

**Report of the VSDBabySoC:**
```
=== vsdbabysoc ===

        +----------Local Count, excluding submodules.
        | 
     4054 wires
     5528 wire bits
      104 public wires
     1578 public wire bits
        7 ports
        7 port bits
     5238 cells
        8   $scopeinfo
        1   avsddac
        1   avsdpll
        1   sky130_fd_sc_hd__a2111oi_0
        8   sky130_fd_sc_hd__a211o_1
      329   sky130_fd_sc_hd__a211oi_1
        8   sky130_fd_sc_hd__a21boi_0
        9   sky130_fd_sc_hd__a21o_1
      728   sky130_fd_sc_hd__a21oi_1
        1   sky130_fd_sc_hd__a221o_1
       42   sky130_fd_sc_hd__a221oi_1
      227   sky130_fd_sc_hd__a222oi_1
        2   sky130_fd_sc_hd__a22o_1
      112   sky130_fd_sc_hd__a22oi_1
        1   sky130_fd_sc_hd__a2bb2oi_1
       27   sky130_fd_sc_hd__a311oi_1
        3   sky130_fd_sc_hd__a31o_1
       15   sky130_fd_sc_hd__a31oi_1
        1   sky130_fd_sc_hd__a32oi_1
        1   sky130_fd_sc_hd__a41oi_1
       39   sky130_fd_sc_hd__and2_0
       33   sky130_fd_sc_hd__and3_1
        4   sky130_fd_sc_hd__and3b_1
        9   sky130_fd_sc_hd__and4_1
        1   sky130_fd_sc_hd__and4b_1
       41   sky130_fd_sc_hd__clkinv_1
     1144   sky130_fd_sc_hd__dfxtp_1
       16   sky130_fd_sc_hd__lpflow_inputiso1p_1
       62   sky130_fd_sc_hd__lpflow_isobufsrc_1
       18   sky130_fd_sc_hd__maj3_1
        5   sky130_fd_sc_hd__mux2_1
       64   sky130_fd_sc_hd__mux2i_1
        1   sky130_fd_sc_hd__mux4_2
     1251   sky130_fd_sc_hd__nand2_1
       28   sky130_fd_sc_hd__nand2b_1
       67   sky130_fd_sc_hd__nand3_1
       32   sky130_fd_sc_hd__nand4_1
      516   sky130_fd_sc_hd__nor2_1
        3   sky130_fd_sc_hd__nor2b_1
       19   sky130_fd_sc_hd__nor3_1
        7   sky130_fd_sc_hd__nor3b_1
        6   sky130_fd_sc_hd__nor4_1
        1   sky130_fd_sc_hd__o2111ai_1
       10   sky130_fd_sc_hd__o211ai_1
       16   sky130_fd_sc_hd__o21a_1
      146   sky130_fd_sc_hd__o21ai_0
        2   sky130_fd_sc_hd__o221ai_1
        2   sky130_fd_sc_hd__o22a_1
       13   sky130_fd_sc_hd__o22ai_1
        1   sky130_fd_sc_hd__o2bb2ai_1
        2   sky130_fd_sc_hd__o311ai_0
        7   sky130_fd_sc_hd__o31ai_1
        1   sky130_fd_sc_hd__o32a_1
        4   sky130_fd_sc_hd__o32ai_1
        2   sky130_fd_sc_hd__o41a_1
        4   sky130_fd_sc_hd__o41ai_1
        9   sky130_fd_sc_hd__or3_1
        3   sky130_fd_sc_hd__or4_1
        1   sky130_fd_sc_hd__or4b_1
       77   sky130_fd_sc_hd__xnor2_1
       46   sky130_fd_sc_hd__xor2_1

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

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fe9758a9-d638-4795-89d0-b48520dd8aef" />


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

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b222b1ac-bec4-4aed-b907-71b2ac286cf3" />


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

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0d90cea3-29a9-409d-bd24-f72f9c13e5ad" />


To solve the error we have to comment this:

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/09d995e6-8fcb-4190-bc62-feec53cc0f4b" />


After commenting:

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0543153f-8e50-43e3-972b-1500b5428b33" />


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

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/02015c45-4304-4496-a38d-e3852e9c8c8e" />


---

### Post-Synthesis Simulation Waveform

In the GTKWave viewer, right-click on the `OUT[9:0]` signal. From the context menu that appeared, navigate to `Data Format`, then to the `Analog` sub-menu, and finally select `Step`.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/96f90f9d-f33a-4b59-8472-ebed2f497f52" />


This is done to change the visual drawing style of the analog waveform for better analysis.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ed6d3d11-9b0a-46a2-a583-4c9d76804b13" />


*The above waveform shows the VSDBabySoC behavior after synthesis, with realistic gate delays included.*

---

### Waveform Analysis and Comparison

#### Functional Correctness Verification ✅

The primary objective of post-synthesis simulation is to confirm that the synthesized gate-level netlist maintains the same functional behavior as the original RTL design.

* **Analyzing the recieved output** -
  1. **`reset`**: The simulation begins with `reset` held **high** for a few microseconds. This correctly initializes the system. Once `reset` goes **low**, the processor starts executing its program.
  2. **`CLK`**: This is the main system clock, generated by the PLL. It appears to be a stable, periodic square wave.
  3. **`RV_TO_DAC[9:0]`**: This is the 10-bit digital output from the RISC-V core. A repeating pattern of changing bits. This pattern represents the sequence of numbers the processor is sending to the DAC.
  4. **`VREFH`**: This is the high reference voltage for the DAC. As expected, it is a **constant DC value**, represented by the flat line at the top.
  5. **`OUT`** (Analog Output): This signal is an analog waveform that directly mirrors the digital pattern from `RV_TO_DAC`. It is a periodic periodic signal. 

---

#### Comparison Summary Table

| **Aspect** | **Pre-Synthesis (RTL) Simulation** | **Post-Synthesis (GLS) Simulation** |
|------------|-------------------------------------|--------------------------------------|
| **Functional Behavior** | ✅ Correct waveform, incrementing and then decrementing digital values | ✅ Identical functional behavior maintained |
| **Signal Values** | Match expected sequence | Match expected sequence (same as RTL) |
| **Waveforms** |  <img width="1920" height="1080" alt="Screenshot from 2025-10-04 16-22-44" src="https://github.com/user-attachments/assets/6107a2f2-8ece-43ea-86ce-de3ace828784" /> | <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ab33e6b4-e022-4926-84e6-7afa954dd058" /> |

---

