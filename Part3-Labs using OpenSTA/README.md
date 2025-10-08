# Part 3 – Generate Timing Graphs with OpenSTA

---

## Understanding of OpenSTA tool -

### 1. OpenSTA: Inputs and Outputs ⚙️

* **Inputs:** OpenSTA needs three essential files to perform its analysis:
  1.  **Synthesized Netlist (`.v`):** This is the gate-level Verilog file produced by your synthesis tool (Yosys). It describes your design as a network of standard cells. **This is the circuit to be analyzed.**
  2.  **Standard Cell Library (`.lib`):** This is the Liberty timing library for your technology (e.g., SKY130). It contains the timing information for every logic gate, including propagation delays, setup times, and hold times. **This file defines how fast the circuit's components are.**
  3.  **Synopsys Design Constraints (`.sdc`):** This is a text file you create. It defines the timing requirements for your design, such as the clock period and I/O delays. **This file sets the performance goals the circuit must meet.**

* **Outputs:** The primary output of OpenSTA is not a file but text-based information printed to the console (which you can redirect to a file).
  1.  **Timing Reports:** This is the main output, showing detailed information about the timing paths in your design, including the calculated **slack**.
  2.  **Checks and Status Reports:** These are useful for debugging, highlighting issues like unconstrained paths or un-clocked registers.

### 2. The Synopsys Design Constraints (`.sdc`) File 📝

The **`.sdc` (Synopsys Design Constraints)** file contains all the **timing and design intent constraints** used by STA tools. It defines how the tool interprets **clocking, input/output delays, loads, and operating conditions** to analyze timing paths correctly.

### 🧩 Commonly Used SDC Commands

| **Command**                           | **Purpose**                                                                             | **Key Options / Syntax**                                                                        | **Example**                                                                 |                                             
| ------------------------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------- |
| **`create_clock`**                    | Defines a clock signal used for timing analysis.                                        | `create_clock -name <clk_name> -period <time> -waveform {<rise> <fall>} [get_ports <clk_port>]` | `create_clock -name core_clk -period 20.0 -waveform {0 10} [get_ports clk]` |                                             
| **`set_input_delay`**                 | Models delay between clock edge and when input data arrives at the chip.                | `set_input_delay <value> -clock <clk_name> [get_ports <input_port>]`                            | `set_input_delay 2.0 -clock core_clk [get_ports data_in]`                   |                                             
| **`set_output_delay`**                | Models delay between clock edge and when output data must be valid for the next device. | `set_output_delay <value> -clock <clk_name> [get_ports <output_port>]`                          | `set_output_delay 2.5 -clock core_clk [get_ports data_out]`                 |                                             
| **`set_clock_uncertainty`**           | Accounts for clock jitter, skew, and other uncertainties.                               | `set_clock_uncertainty <value> [get_clocks <clk_name>]`                                         | `set_clock_uncertainty 0.2 [get_clocks core_clk]`                           |                                             
| **`set_input_transition`**            | Defines input signal transition time (slew).                                            | `set_input_transition <value> [get_ports <input_port>]`                                         | `set_input_transition 0.1 [get_ports data_in]`                              |                                             
| **`set_load`**                        | Specifies the capacitive load seen by an output port.                                   | `set_load <cap_value> [get_ports <output_port>]`                                                | `set_load 0.05 [get_ports data_out]`                                        |                                             
| **`set_driving_cell`**                | Models a specific driving cell for inputs (to emulate realistic driver strength).       | `set_driving_cell -lib_cell <cell_name> [get_ports <input_port>]`                               | `set_driving_cell -lib_cell INVX4 [get_ports data_in]`                      |                                             
| **`set_false_path`**                  | Excludes specific paths from timing analysis (e.g., asynchronous or test paths).        | `set_false_path -from [get_ports <src>] -to [get_ports <dst>]`                                  | `set_false_path -from [get_ports rst_n] -to [get_ports data_out]`           |                                             
| **`set_multicycle_path`**             | Defines timing paths that take more than one clock cycle to complete.                   | `set_multicycle_path <cycles> -from [get_ports <src>] -to [get_ports <dst>]`                    | `set_multicycle_path 2 -from [get_ports data_in] -to [get_ports data_out]`  |                                             
| **`set_max_delay` / `set_min_delay`** | Overrides default max/min delay values for a path.                                      | `set_max_delay <value> -from <start> -to <end>`                                                 | `set_max_delay 10 -from [get_ports data_in] -to [get_ports data_out]`       |                                             
| **`set_disable_timing`**              | Disables timing checks on specific arcs (useful for non-timing control signals).        | `set_disable_timing -from <pin> -to <pin>`                                                      | `set_disable_timing -from u1/A -to u1/Y`                                    |                                             

### 3. Basic OpenSTA Commands ⚙️

You typically run **OpenSTA** (Static Timing Analyzer) either in an **interactive shell** or by providing a **TCL script**. The commands must be entered in a logical sequence to load the libraries, design netlist, timing constraints, and then perform timing analysis.

### Common OpenSTA Tcl Commands

| **Command** | **Purpose** | **Common Syntax** | **Example** |
| :--- | :--- | :--- | :--- |
| **`read_liberty`** | Loads a standard cell timing library (`.lib`). Use `-max` for setup checks (slow corner) and `-min` for hold checks (fast corner). | `read_liberty [-max \| -min] <path_to_lib.lib>` | `read_liberty -max slow.lib`<br>`read_liberty -min fast.lib` |
| **`read_verilog`** | Reads the synthesized gate-level Verilog netlist into the tool. | `read_verilog <path_to_netlist.v>` | `read_verilog my_design.synth.v` |
| **`link_design`** | Links the netlist instances to the corresponding cells in the loaded libraries. The argument must be the top-level module name. | `link_design <top_module_name>` | `link_design vsdbabysoc` |
| **`create_clock`** | Defines a clock source, specifying its period and the port where it's applied. This is the most fundamental timing constraint. | `create_clock -name <clk_name> -period <val> [get_ports <port>]` | `create_clock -name sys_clk -period 10.0 [get_ports clk]` |
| **`read_sdc`** | Reads a Synopsys Design Constraints (`.sdc`) file, which contains all timing constraints (`create_clock`, I/O delays, false paths, etc.). | `read_sdc <path_to_sdc_file>` | `read_sdc my_design.sdc` |
| **`report_checks`** | Reports **detailed timing path information**, including delay, required time, and slack for setup (max) and hold (min) paths. | `report_checks [-path_delay min_max] [-digits <N>]` | `report_checks -path_delay min_max -digits 4` |



### 4. Walkthrough of a Basic Example

#### **Netlist**

```verilog
module top (in1, in2, clk1, clk2, clk3, out);
  input in1, in2, clk1, clk2, clk3;
  output out;
  wire r1q, r2q, u1z, u2z;

  DFF_X1 r1 (.D(in1), .CK(clk1), .Q(r1q));
  DFF_X1 r2 (.D(in2), .CK(clk2), .Q(r2q));
  BUF_X1 u1 (.A(r2q), .Z(u1z));
  AND2_X1 u2 (.A1(r1q), .A2(u1z), .ZN(u2z));
  DFF_X1 r3 (.D(u2z), .CK(clk3), .Q(out));
endmodule 
```

#### **Objective**

Find the worst-case hold and setup slack in the netlist (example1.v).

#### **Commands to Enter**

```tcl
# --- OpenSTA Timing Analysis Script for example1 ---

# 1. Load the Standard Cell Library
# This command reads the Nangate 45nm library. The .lib file is crucial as it
# contains all the timing information (propagation delays, setup/hold times) for
# the logic gates used in the design.
read_liberty /OpenSTA/examples/nangate45_typ.lib.gz

# 2. Read the Synthesized Netlist
# This reads the gate-level Verilog file that describes the circuit's structure
# and the interconnection of all the logic gates.
read_verilog /OpenSTA/examples/example1.v

# 3. Link the Design to the Library
# This essential step connects the gates in the Verilog netlist to their
# corresponding timing models in the loaded library, making the design "timing-aware."
link_design top

# 4. Define the Clock Constraint
# This is the most important constraint. It sets the performance target by defining
# a clock named 'clk' with a 10ns period (100 MHz frequency) and applies this
# constraint to the design's three clock input ports.
create_clock -name clk -period 10 {clk1 clk2 clk3}

# 5. Define Input Delays
# This command models the external world. It tells the tool that for any timing
# path starting at inputs 'in1' or 'in2', the data signal arrives 1ns *after*
# the rising edge of the 'clk'.
set_input_delay -clock clk 1 {in1 in2}

# 6. Perform a Preliminary Check (Optional, but good practice)
# This command finds the raw longest (max) and shortest (min) combinational path
# delays. It's not a full timing report with slack, but it gives a quick
# indication of the design's overall speed.
report_checks -path_delay min_max

# 7. Generate the Full Setup Timing Report
# This is the main analysis command. It performs a full setup analysis and generates
# a detailed report for the 10 worst-offending paths. The report will show the
# complete delay breakdown, the required time, and the final slack value.
# The output of this command is what you analyze for timing violations.
report_timing -nworst 10

# 8. Exit the Tool
# Exits the OpenSTA interactive shell.
exit
```

#### **Output Received**

<img width="1920" height="1080" alt="Screenshot from 2025-10-08 21-32-35" src="https://github.com/user-attachments/assets/2888c77e-521c-4a00-804b-bb239584b469" />


<img width="1920" height="1080" alt="Screenshot from 2025-10-08 21-32-52" src="https://github.com/user-attachments/assets/1d8afc2f-08d3-4aac-860a-5876a314457a" />


#### **Analysis of the Output**

* **Hold Analysis:** 

1.  **Startpoint**: in1 (input port clocked by clk)
2.  **Endpoint:** r1 (rising edge-triggered flip-flop clocked by clk)
3.  **Path Group:** clk (the timing path is being analyzed)
4.  **Path Type:** min (the analysis is a hold time check)
5.  **Data Arrival Time**: The earliest possible time a signal can travel from its startpoint to its endpoint which is the sum of min delays in the timing path.
6.  **Data Required Time**: The earliest moment the data is allowed to change at the input of the capture flip-flop. 
7.  **Slack**: This is the difference between the **Arrival Time** and the **Required Time**. If it's positive, the path has no violation. If it's negative, it's a timing violation. Here it is 0.10ns which is positive and hence there is no timing violation. 

Slack calculation:

$$Slack_{hold} = \text{Arrival Time}_{min} - \text{Required Time}_{min}$$

Hold time constraint:

$$T_{cq_{min}} + T_{logic_{min}} > T_{hold}$$

* **Setup Analysis:** 

1.  **Startpoint**: r2 (rising edge-triggered flip-flop clocked by clk)
2.  **Endpoint:** r3 (rising edge-triggered flip-flop clocked by clk)
3.  **Path Group:** clk (the timing path is being analyzed)
4.  **Path Type:** max (the analysis is a setup time check)
5.  **Data Arrival Time**: The latest possible time a signal can travel from its startpoint to its endpoint which is the sum of max delays in the timing path.
6.  **Data Required Time**: The time by which the data must arrive at the capture flip-flop's input to be reliably captured by the next clock edge.
7.  **Slack**: This is the difference between the **Required Time** and the **Arrival Time**. If it's positive, the path has no violation. If it's negative, it's a timing violation. Here it is 9.83ns which is positive and hence there is no timing violation. 

Slack calculation:

$$Slack_{setup} = \text{Required Time}_{max} - \text{Arrival Time}_{max}$$

Setup time constraint:

$$T_{cq_{max}} + T_{logic_{max}} + T_{setup} < \text{Clock Period}$$

-----

### \#\# Performing STA on `VSDBabySoC` 📈

Now, let's apply this to your own project.

#### **Step 1: Prerequisites**

Make sure you have the following files ready:

  * Your synthesized netlist: `vsdbabysoc.synth.v`
  * The SKY130 Liberty file: `sky130_fd_sc_hd__tt_025C_1v80.lib`

#### **Step 2: Create the `vsdbabysoc.sdc` File**

Create a new file named `vsdbabysoc.sdc`. Let's target a **50 MHz clock (20 ns period)**.

```tcl
# vsdbabysoc.sdc
# Target clock frequency: 50 MHz

# Define the clock on the 'clk' port with a 20ns period
create_clock -period 20.0 [get_ports {clk}]

# Add a small amount of uncertainty to account for jitter
set_clock_uncertainty 0.2 [get_clocks *]
```

*(Note: Ensure `clk` is the correct name of your top-level clock port.)*

#### **Step 3: Run OpenSTA**

Launch OpenSTA (`sta`) and run the following commands, replacing the paths with your own.

```tcl
# In the OpenSTA shell
read_liberty /path/to/your/libs/sky130_fd_sc_hd__tt_025C_1v80.lib

read_verilog /path/to/your/output/vsdbabysoc.synth.v

link_design vsdbabysoc

read_sdc /path/to/your/src/vsdbabysoc.sdc

report_timing -nworst 10 > vsdbabysoc_timing_report.txt
```

#### **Step 4: Analyze Your Report**

Open the generated `vsdbabysoc_timing_report.txt` file.

  * **What is the worst slack value?** Is it positive or negative?
  * If it's positive, your design meets the 50 MHz timing goal. ✅
  * If it's negative, your design fails to meet timing. Look at the path details: which cells or nets in the `VSDBabySoC` design are contributing the most delay? This is your **critical path**.
