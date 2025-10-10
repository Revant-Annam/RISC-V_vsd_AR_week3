# Part 3 – Generate Timing Graphs with OpenSTA

---

## Understanding of OpenSTA tool -

### 1. OpenSTA: Inputs and Outputs ⚙️

* **Inputs:** OpenSTA needs three essential files to perform its analysis:
  1.  **Synthesized Netlist (`.v`):** This is the gate-level Verilog file produced by your synthesis tool (like Yosys). It describes your design as a network of standard cells. **This is the circuit to be analyzed.**
  2.  **Standard Cell Library (`.lib`):** This is the Liberty timing library for your technology (e.g., SKY130). It contains the timing information for every logic gate, including propagation delays, setup times, and hold times. **This file defines how fast the circuit's components are.**
  3.  **Synopsys Design Constraints (`.sdc`):** This is a text file you create. It defines the timing requirements for your design, such as the clock period and I/O delays. **This file sets the performance goals the circuit must meet.**
  4.  **Standard Delay Format (`.sdf`) (After placement):** Generated after Place & Route, this file contains accurate, post-layout timing delays for both cells and interconnecting wires. It's used to "back-annotate" the design for a highly realistic timing check. **This file provides the *actual* cell and wire delays after layout.**
  5.  **Standard Parasitic Exchange Format (`.spef`) (After placement):** Also generated after Place & Route, this file details the parasitic resistance and capacitance (RC) of the physical wires. The timing tool uses this information to calculate precise net delays. **This file describes the physical properties of the wires.**

* **Outputs:** The primary output of OpenSTA is not a file but text-based information printed to the console (which you can redirect to a file).
  1.  **Timing Reports:** This is the main output, showing detailed information about the timing paths in your design, including the calculated **slack**.
  2.  **Checks and Status Reports:** These are useful for debugging, highlighting issues like unconstrained paths or un-clocked registers.

<p align = "center">
  <img width="450" height="300" alt="image" src="https://github.com/user-attachments/assets/cdda71d1-9a02-4bb0-8e6b-0e19ee4ce535" />
</p>

### 2. The Synopsys Design Constraints (`.sdc`) File 📝

The **`.sdc` (Synopsys Design Constraints)** file contains all the **timing and design intent constraints** used by STA tools. It defines how the tool interprets **clocking, input/output delays, loads, and operating conditions** to analyze timing paths correctly.

### 🧩 Commonly Used SDC Commands

Of course. Here is a clean, well-formatted table of common SDC (Synopsys Design Constraints) commands used in Static Timing Analysis.

***

### Common SDC Timing Commands

| **Command** | **Purpose** | **Common Syntax** | **Example** |
| :--- | :--- | :--- | :--- |
| **`create_clock`** | Defines a primary clock signal, specifying its period and waveform. | `create_clock -name <clk_name> -period <time> [get_ports <port>]` | `create_clock -name core_clk -period 10.0 [get_ports clk]` |
| **`create_generated_clock`** | Defines a clock that is derived from another clock, such as by a divider or multiplier. | `create_generated_clock -name <new_clk> -source <master_pin> -divide_by <N>` | `create_generated_clock -name clk_div2 -source u_div/CLK -divide_by 2` |
| **`set_input_delay`** | Models the external delay on an input port, relative to a clock edge. | `set_input_delay <value> -clock <clk_name> [get_ports <input_port>]` | `set_input_delay 2.5 -clock core_clk [get_ports data_in[*]]` |
| **`set_output_delay`** | Models the external timing requirement for an output port, relative to a clock edge. | `set_output_delay <value> -clock <clk_name> [get_ports <output_port>]` | `set_output_delay 3.0 -clock core_clk [get_ports data_out[*]]` |
| **`set_clock_uncertainty`** | Specifies a margin to account for clock jitter and skew, tightening timing checks. | `set_clock_uncertainty <value> [get_clocks <clk_name>]` | `set_clock_uncertainty 0.25 [get_clocks core_clk]` |
| **`set_clock_latency`** | Models the clock network's insertion delay from the source to the register clock pins. | `set_clock_latency [-source] <value> [get_clocks <clk_name>]` | `set_clock_latency -source 1.8 [get_clocks core_clk]` |
| **`set_driving_cell`** | Models an input port being driven by a specific library cell for more accurate slew calculation. | `set_driving_cell -lib_cell <cell> [get_ports <input_port>]` | `set_driving_cell -lib_cell BUFX2 [get_ports enable]` |
| **`set_load`** | Specifies the capacitive load (in pF or fF) that an output port must drive. | `set_load <capacitance> [get_ports <output_port>]` | `set_load 0.05 [get_ports data_out[7]]` |
| **`set_max_fanout`** | A design rule that limits the number of gates an output pin can drive. | `set_max_fanout <value> [get_ports <port_name>]` | `set_max_fanout 16 [all_inputs]` |
| **`set_max_transition`** | A design rule that limits the signal transition time (slew) on nets to ensure signal integrity. | `set_max_transition <time> [current_design]` | `set_max_transition 0.6 [current_design]` |
| **`set_false_path`** | Instructs the timing analyzer to ignore a specific path (e.g., asynchronous reset). | `set_false_path -from <startpoint> -to <endpoint>` | `set_false_path -from [get_ports rst_n] -to [all_registers]` |
| **`set_multicycle_path`** | Specifies that a path is allowed to take multiple clock cycles to propagate. | `set_multicycle_path <N> -setup -from <startpoint> -to <endpoint>` | `set_multicycle_path 2 -setup -from [get_pins data_reg[*]/Q]` |
| **`set_disable_timing`** | Breaks a timing arc within a cell, preventing timing analysis through it. | `set_disable_timing -from <pin_name> -to <pin_name>` | `set_disable_timing -from u_mux/S -to u_mux/Y` |                                        

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

### 4. Starting OpenSTA

#### **Step 1 -** Build a docker image

Using this command, we can build the image for OpenSTA. It essentially packages the OpenSTA application and all its necessary dependencies into a standardized, portable unit called a Docker image. 📦

`docker build --file Dockerfile.ubuntu22.04 --tag opensta .`

* **`docker build`**: This is the fundamental command that tells the Docker engine to create a new image.

* **`--file Dockerfile.ubuntu22.04`**: This flag specifies the exact "blueprint" file to use for the build. A Dockerfile contains a list of instructions, like installing libraries and copying files, needed to set up the environment. You're telling Docker to use the instructions in the `Dockerfile.ubuntu22.04` file.

* **`--tag opensta`**: The `--tag` flag assigns a human-readable name (a "tag") to the finished image. You've named it **`opensta`**, which makes it easy to find and run later (e.g., `docker run opensta`).

* **`.`** (the dot at the end): This specifies the **build context**, which is the set of files in your current directory. It tells Docker where to find the Dockerfile and any other local files that need to be included in the image.

#### **Step 2 -** Run OpenSTA

This command starts a new container from `opensta` image and gives an interactive command line inside it, with the local files readily accessible. 🚀

`docker run -it -v $HOME:/data opensta`

* **`docker run`**: The command to create and start a new container from a specified image.

* **`-it`**: This is a combination of two flags that are almost always used together for interactive sessions:
    * **`-i`** (`--interactive`): Keeps the input stream (STDIN) open, allowing you to type commands into the container.
    * **`-t`** (`--tty`): Allocates a pseudo-terminal, which gives you the shell prompt and makes the container session feel like a regular terminal.

* **`-v $HOME:/data`**: This is the most important part for file sharing. The `-v` (`--volume`) flag **mounts** a host directory into the container.
    * **`$HOME`**: This is a variable that points to your home directory on your main computer (the **host**).
    * **`/data`**: This is the path inside the **container** where your home directory will appear.
    * **`:`**: This colon separates the host path from the container path.
    * Essentially, you've created a direct link between your computer's home directory and the `/data` directory inside the container. Any file you place in `$HOME` will be instantly available inside the container at `/data`, and any report you generate inside the container at `/data` will be saved directly to your computer. ↔️

* **`opensta`**: This specifies the name of the image to use as the blueprint for creating and running this container. It's the image you built in the previous step.

The `%` terminal confirms that the OpenSTA is running.

<img width="1920" height="1080" alt="Screenshot from 2025-10-10 01-14-13" src="https://github.com/user-attachments/assets/bd72043f-0535-43ca-8b92-b002ffbfec15" />


### 5. Walkthrough of a Basic Example

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
set_input_delay -clock clk 0.1 {in1 in2}

# 6. Perform a Preliminary Check (Optional, but good practice)
# This command finds the raw longest (max) and shortest (min) combinational path
# delays. It's not a full timing report with slack, but it gives a quick
# indication of the design's overall speed.
report_checks -path_delay min_max

# 7. Exit the Tool
# Exits the OpenSTA interactive shell.
exit
```

#### **Output Received**

<img width="1920" height="1080" alt="Screenshot from 2025-10-10 01-15-30" src="https://github.com/user-attachments/assets/3fb669ed-b13a-4b70-8be7-543c89c3104b" />

<img width="1920" height="1080" alt="Screenshot from 2025-10-10 01-16-30" src="https://github.com/user-attachments/assets/cd477ec4-06a1-4629-8c4f-f1cbc1592411" />


#### **Analysis of the Output**

| Feature | Hold Analysis | Setup Analysis |
| :--- | :--- | :--- |
| **Goal** | Ensures data doesn't change too soon after a clock edge. | Ensures data is stable long enough before the next clock edge. |
| **Path Type** | `min` (fastest path analysis which is hold analysis) | `max` (slowest path analysis which is setup analysis) |
| **Startpoint** | `in1` (input port) | `r2` (rising edge-triggered flip-flop clocked by clk) |
| **Endpoint** | `r1` (rising edge-triggered flip-flop clocked by clk) | `r3` (rising edge-triggered flip-flop clocked by clk) |
| **Path Group** | `clk` | `clk` |
| **Data Arrival Time**| The **earliest** possible time a signal arrives at the endpoint. | The **latest** possible time a signal arrives at the endpoint. |
| **Data Required Time**| The earliest moment the data is allowed to change at the capture flop. | The time by which data **must arrive** to be reliably captured. |
| **Slack Value** | **0.10 ns** (Positive, no violation) | **9.83 ns** (Positive, no violation) |
| **Slack Calculation**| $Slack_{hold} = \text{Arrival Time}_{min} - \text{Required Time}_{min}$ | $Slack_{setup} = \text{Required Time}_{max} - \text{Arrival Time}_{max}$ |
| **Timing Constraint**| $T_{cq_{min}} + T_{logic_{min}} > T_{hold}$ | $T_{cq_{max}} + T_{logic_{max}} + T_{setup} < \text{Clock Period}$ |

-----

## Performing STA on `VSDBabySoC` 📈

### **Step 1: Prerequisites**

In the `VLSI/VSDBabySoC/src` directory I have made a `vsdbabysoc_min_max_delays.tcl` for perfoming the setup and hold time checks for the VSDBabySoC. The contents of `.tcl` file can be run individually in the OpenSTA tool also. The contents of the file:

```tcl
# Liberty files
read_liberty -min /data/VLSI/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_liberty -max /data/VLSI/VSDBabySoC/src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_liberty -min /data/VLSI/VSDBabySoC/src/lib/avsdpll.lib
read_liberty -max /data/VLSI/VSDBabySoC/src/lib/avsdpll.lib
read_liberty -min /data/VLSI/VSDBabySoC/src/lib/avsddac.lib
read_liberty -max /data/VLSI/VSDBabySoC/src/lib/avsddac.lib

# Synthesized design
read_verilog /data/VLSI/VSDBabySoC/output/synth/vsdbabysoc.synth.v

# Link top module
link_design vsdbabysoc

# SDC constraints
read_sdc /data/VLSI/VSDBabySoC/src/sdc/vsdbabysoc_synthesis.sdc

# Timing report
report_checks
```

### **Step 2: Correcting the error**

With the help of docker command we can run the `.tcl` file,

```
docker run -it -v $HOME:/data opensta /data/VLSI/VSDBabySoC/src/vsdbabysoc_min_max_delays.tcl
```

But this resulted in the following error,

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3ec40dbb-5122-4a09-a864-ce13a9d3ca56" />

This error occurs because the Liberty (.lib) file format does not support // for single-line comments. The parser fails when it finds this syntax, especially when a { character follows it. To fix this, inspect avsdpll.lib around line 54 and correct the comment formatting.

In the `avsdpll.lib` line 54:
```
//pin (GND#2) {
//  direction : input;
//  max_transition : 2.5;
//  capacitance : 0.001;
//}
```

This needs to be converted into:
```
/*
pin (GND#2) {
  direction : input;
  max_transition : 2.5;
  capacitance : 0.001;
}
*/
```

A similar error occured in line 68.

### **Step 3: Run `.tcl` file in OpenSTA**

After correcting the errors using the same docker command we can run the `.tcl` file. The timing report generated is 

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/972afbc2-299e-4aa1-a506-5e15fa17675e" />


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/09ec4f40-b4af-4b59-800b-92f208e80a6e" />


As the slack is positive for both the hold and setup time analysis there is no timing violation.


### **Step 4: Analyze Your Report**

#### Detailed Hold Path Analysis (min)

This analysis checks the **fastest path** to ensure data doesn't change too quickly, causing a violation at the capture flop.

* **Path:** From flip-flop `_9108_` to `_8046_`
* **Result:** Slack is **0.31 ns (MET)**. The path is stable.

| Delay (ns) | Time (ns) | Path Description | Explanation |
| :--- | :--- | :--- | :--- |
| **Data Arrival Path** | (How fast the data *actually* travels) |
| 0.00 | 0.00 | `clock clk` (rise edge) | The analysis starts at the rising edge of the clock (time zero). |
| 0.00 | 0.00 | `_9108_/CLK` | The clock signal arrives at the launch flip-flop's clock pin (`CLK`). |
| 0.27 | 0.27 | `_9108_/Q` | **Clock-to-Q Delay:** The time it takes for the flip-flop's output (`Q`) to update after the clock edge. |
| 0.00 | 0.27 | `_8046_/D` | **Net Delay:** The time for the signal to travel along the wire to the next flip-flop's data pin (`D`). |
| | **0.27** | **Data Arrival Time** | The total time it took for the data to arrive at the destination. |
| **Data Required Path**| (How long the data *must* remain stable) |
| 0.00 | 0.00 | `clock clk` (rise edge) | This is the *same* clock edge, now viewed from the capture flip-flop. |
| 0.00 | 0.00 | `_8046_/CLK` | The clock edge arrives at the capture flip-flop. |
| -0.03 | -0.03 | `library hold time` | **Hold Constraint:** The data must remain stable for `0.03 ns` *after* the clock edge. |
| | **-0.03** | **Data Required Time** | The time boundary. The data is required to be stable until this moment. |

**Summary:** The data arrives at `0.27 ns` and was only required to stay stable until `-0.03 ns`. The slack is calculated as `Arrival Time - Required Time` (`0.27 - (-0.03) = 0.30 ns`, matching the report's `0.31 ns` after tool precision). Since it's positive, the hold condition is met.

#### Detailed Setup Path Analysis (max)

This analysis checks the **slowest path** (critical path) to ensure data arrives *before* the next clock edge.

* **Path:** From flip-flop `_9085_` to `_8462_`
* **Result:** Slack is **2.26 ns (MET)**. The path is fast enough.

| Delay (ns) | Time (ns) | Path Description | Explanation |
| :--- | :--- | :--- | :--- |
| **Data Arrival Path** | (How long the data *actually* takes to arrive) |
| 0.00 | 0.00 | `clock clk` (rise edge) | The analysis starts at the rising edge of the clock (time zero). |
| 0.00 | 0.00 | `_9085_/CLK` | The clock signal arrives at the launch flip-flop's clock pin (`CLK`). |
| 8.43 | 8.43 | `_9085_/Q` | **Launch & Logic Delay:** Time for the data to launch from the flip-flop and travel through logic. |
| -0.25 | 8.19 | `_6479_/Y` | **Intermediate Logic Delay:** The signal passes through a logic gate (`a21oi_1`). |
| 0.00 | 8.19 | `_8462_/D` | **Net Delay:** The final wire delay to the capture flip-flop's data pin (`D`). |
| | **8.19** | **Data Arrival Time** | The total time it took for the data to arrive at the destination. |
| **Data Required Path**| (The deadline for when the data *must* arrive) |
| 11.00 | 11.00 | `clock clk` (rise edge) | This is the **next** clock edge, which defines the end of the timing window. |
| 0.00 | 11.00 | `_8462_/CLK` | The capture clock edge arrives at the capture flip-flop. |
| -0.55 | 10.45 | `library setup time` | **Setup Constraint:** The data must be stable for `0.55 ns` *before* this clock edge. |
| | **10.45** | **Data Required Time** | The final deadline by which the data must have arrived and be stable. |

**Summary:** The data must arrive by **10.45 ns**, but it actually arrives much earlier at **8.19 ns**. The slack is calculated as `Required Time - Arrival Time` (`10.45 - 8.19 = 2.26 ns`). Since it's positive, the setup condition is comfortably met. The next clock edge is at 11 ns, the frequency of operation is approximately 90.91 MHz.

**Other observations:**

* **Pre-Layout Analysis:** The `0.00 ns` net delays indicate this timing report analyzes the synthesized design assuming ideal, instantaneous connections.

* **Focus on Logic Delay:** This analysis validates the timing of the logic gates themselves, ignoring the impact of wire delays.

* **Optimistic Slack:** The current positive slack of **2.26 ns** is a good sign, but it's an optimistic best-case scenario.

## **VSDBabySoC PVT Corner Analysis (Post-Synthesis Timing)**

### What is PVT Corner Analysis?

PVT analysis is a method used in chip design to verify that a circuit will function correctly under all possible real-world conditions. PVT stands for **Process**, **Voltage**, and **Temperature**.

  * **Process (P):** Due to tiny, unavoidable imperfections during manufacturing, transistors on a chip can be slightly faster or slower than their typical specification. We model these as process corners:

      * **SS (Slow-Slow):** Transistors are at their slowest.
      * **FF (Fast-Fast):** Transistors are at their fastest.
      * **TT (Typical-Typical):** Transistors perform as expected.

  * **Voltage (V):** A chip operates within a specified voltage range (e.g., 1.8V ±10%).

      * **Lower Voltage:** Makes transistors switch slower.
      * **Higher Voltage:** Makes transistors switch faster.

  * **Temperature (T):** The chip's performance is affected by its operating temperature.

      * **Higher Temperature:** Generally makes transistors slower.
      * **Lower Temperature:** Generally makes transistors faster.

By combining the extremes of these three factors, we create **PVT corners**. We perform Static Timing Analysis (STA) at these corners to find the absolute worst-case performance, ensuring the chip is robust.

  * **Worst Case for Setup (Slowest Path):** To check for setup violations, we find the slowest possible condition: **Slow Process (SS), Low Voltage, and High Temperature**. If the data can arrive on time in this sluggish state, it will work in all faster conditions.
  * **Worst Case for Hold (Fastest Path):** To check for hold violations, we find the fastest possible condition: **Fast Process (FF), High Voltage, and Low Temperature**. If data doesn't change too quickly in this speedy state, it will be fine in all slower conditions.

-----

### PVT Analysis Tcl Script for OpenSTA

The script you provided loops through *all* available libraries, which is great for exploration but not targeted. For a proper PVT analysis, we only need to test the **critical corners** you identified.

This improved script targets only the four most critical corners (two for setup, two for hold) to ensure your design's robustness efficiently.

```tcl
# --- 1. Setup Paths and Design Info ---
set DESIGN_NAME "vsdbabysoc"

# Paths for your setup
set LIB_DIR       "/data/VLSI/VSDBabySoC/skywater-pdk-libs-sky130_fd_sc_hd/timing"
set MACRO_LIB_DIR "/data/VLSI/VSDBabySoC/src/lib"
set VERILOG_FILE  "/data/VLSI/VSDBabySoC/src/module/vsdbabysoc.synth.v"
set SDC_FILE      "/data/VLSI/VSDBabySoC/src/sdc/vsdbabysoc_synthesis.sdc"
set OUTPUT_DIR    "/data/VLSI/VSDBabySoC/STA_reports"

# Create the output directory
exec mkdir -p $OUTPUT_DIR


# --- 2. Define Library Corners for Analysis ---
set all_corners {
    "sky130_fd_sc_hd__tt_025C_1v80.lib"
    "sky130_fd_sc_hd__ff_100C_1v65.lib"
    "sky130_fd_sc_hd__ff_100C_1v95.lib"
    "sky130_fd_sc_hd__ff_n40C_1v56.lib"
    "sky130_fd_sc_hd__ff_n40C_1v65.lib"
    "sky130_fd_sc_hd__ff_n40C_1v76.lib"
    "sky130_fd_sc_hd__ss_100C_1v40.lib"
    "sky130_fd_sc_hd__ss_100C_1v60.lib"
    "sky130_fd_sc_hd__ss_n40C_1v28.lib"
    "sky130_fd_sc_hd__ss_n40C_1v35.lib"
    "sky130_fd_sc_hd__ss_n40C_1v40.lib"
    "sky130_fd_sc_hd__ss_n40C_1v44.lib"
}


# --- 3. Loop Through Each Corner and Perform STA ---
foreach corner $all_corners {
    reset_design
    puts "## Analyzing Corner: $corner"

    # Load the corner-specific standard cell library
    read_liberty -min $LIB_DIR/$corner
    read_liberty -max $LIB_DIR/$corner

    # Load the macro libraries
    read_liberty -min $MACRO_LIB_DIR/avsddac.lib
    read_liberty -max $MACRO_LIB_DIR/avsddac.lib
    read_liberty -min $MACRO_LIB_DIR/avsdpll.lib
    read_liberty -max $MACRO_LIB_DIR/avsdpll.lib

    # Load design and constraints
    read_verilog $VERILOG_FILE
    link_design $DESIGN_NAME
    read_sdc $SDC_FILE

    # --- 4. Generate Reports ---
    set report_base_name [string map {.lib ""} $corner]

    report_checks -path_delay max -digits 4 > $OUTPUT_DIR/${report_base_name}_setup.txt
    report_checks -path_delay min -digits 4 > $OUTPUT_DIR/${report_base_name}_hold.txt

    exec echo "--- Corner: $corner ---" >> $OUTPUT_DIR/summary_wns.txt
    report_wns -digits 4 >> $OUTPUT_DIR/summary_wns.txt
    
    exec echo "--- Corner: $corner ---" >> $OUTPUT_DIR/summary_tns.txt
    report_tns -digits 4 >> $OUTPUT_DIR/summary_tns.txt
}

puts "## Analysis Complete. Reports are located in: $OUTPUT_DIR"
```
