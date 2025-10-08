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

#### **Objective**

Find the worst-case hold and setup slack in the netlist (example1.v).

#### **Commands to Enter**

```tcl
# In the OpenSTA shell
# 1. Load the library
read_liberty path/to/sample_library.lib

# 2. Load the design
read_verilog path/to/sample_design.v

# 3. Link the design
link_design top_module_of_sample

# 4. Apply constraints
read_sdc path/to/sample_constraints.sdc

# 5. Run the analysis and generate the report
report_timing -nworst 5 > timing_report.txt

# 6. Exit
exit
```

#### **Output Received**

*(Here you would paste the content of `timing_report.txt`)*

#### **Analysis of the Output**

When you look at the report, you'll analyze it by identifying these key parts for the worst path:

1.  **Startpoint**: The flip-flop where the path begins.
2.  **Endpoint**: The flip-flop where the path ends.
3.  **Data Arrival Time**: The total time it took for the signal to travel from the startpoint to the endpoint. This is the sum of all cell and net delays listed in the first part of the report.
4.  **Data Required Time**: The "deadline" by which the signal must arrive. This is calculated from the next clock edge, minus the destination flip-flop's setup time and any clock uncertainty.
5.  **Slack**: This is the most important number. It's the difference between the **Required Time** and the **Arrival Time**. If it's positive, the path passes. If it's negative, it's a timing violation. ❌

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
