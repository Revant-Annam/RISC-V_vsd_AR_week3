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

The `.sdc` file communicates your design intent and performance goals. The file contains a series of commands. Some of the important commands are:

  * **`create_clock`**: Defines a clock signal.
      * **`-period`**: Specifies the clock's period (e.g., `20.0` for 20 ns, which is 50 MHz).
      * **`-waveform`**: Describes the rising and falling edges (e.g., `{0 10}` for a 20 ns clock that rises at 0 ns and falls at 10 ns).
      * **`[get_ports {clk}]`**: Associates this clock definition with a specific port on your design.
  * **`set_input_delay` / `set_output_delay`**: These commands model the timing of external circuits connected to your chip's input and output pins.
  * **`set_clock_uncertainty`**: This command models real-world clock imperfections like **jitter**, effectively reducing the available time in a clock cycle to ensure a more robust analysis.

### 3. Basic OpenSTA Commands

You typically run OpenSTA in an interactive shell. The commands are entered in a logical sequence to load the design, apply constraints, and run the analysis.

| Command | Purpose |
| :--- | :--- |
| **`read_liberty <file.lib>`** | Loads the standard cell timing library. |
| **`read_verilog <file.v>`** | Reads the synthesized gate-level netlist. |
| **`link_design <top_module_name>`** | Links the netlist instances to their definitions in the library. |
| **`read_sdc <file.sdc>`** | Reads the design constraints and applies them to the design. |
| **`report_checks`** | Reports potential issues, like paths that aren't constrained. |
| **`report_timing`** | **This is the main command.** It performs the analysis and generates the timing report for the most critical paths. |

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
