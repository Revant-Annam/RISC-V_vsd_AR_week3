# **Fundamentals of Static Timing Analysis (STA) – Summary (Part - 2)**

## **1. Basic Terminology**

* **Starting Points:** Input ports or clock pins where signals originate and timing begins.
* **End Points:** Output ports or D pins of flip-flops where timing ends and data is captured.
* **Timing Path:** The route followed by a signal from a startpoint to an endpoint through combinational logic.

<img width="1920" height="1080" alt="Screenshot (344)" src="https://github.com/user-attachments/assets/d982c81a-6753-4305-8fea-05e88591f349" />
<p align = "center"> Timing paths </p>

* **Arrival Time (AT):** The actual time taken by a signal to propagate from the starting point to the endpoint.
* **Required Time (RT):** The latest or earliest time by which a signal must arrive for the circuit to function correctly.
* **Slack Equation:**

   <p align = "center"> $$Slack = \text{Required Time} - \text{Arrival Time}$$ </p>

  * **Positive Slack:** Timing is met (safe margin).
  * **Negative Slack:** Timing violation (requires optimization).
  * **Setup Slack:** Difference between the required maximum and actual arrival time; ensures data arrives **before** the capturing clock edge.
  * **Hold Slack:** Difference between actual arrival and required minimum time; ensures data remains **stable after** the clock edge.

<img width="1920" height="1080" alt="Screenshot (348)" src="https://github.com/user-attachments/assets/7f4f9ce8-2e81-4374-ac3a-0b323f4bfcf3" />
<p align = "center"> Timing graph </p>

---

## **2. Types of Setup & Hold Analysis**

* **Reg-to-Reg:** Path between launch and capture flip-flops (most critical for setup/hold checks).
* **In-to-Reg:** Path from input port to D pin of a register.
* **Reg-to-Out:** Path from register output to an output port.
* **In-to-Out:** Path from input port to output port (purely combinational).
* **Clock-Gating Paths:** Checks from gating enable signals to clock pins.
* **Recovery/Removal Paths:** Ensure asynchronous resets or enables occur with safe timing relative to the clock.

---

## **3. Timing Checks and Clock Analysis**

* **Setup & Hold Analysis:**

  * *Data-to-Data Check:* Ensures synchronization between two related data signals.
  * *Latch Timing:* Level-sensitive latches allow **time borrowing**, helping fix minor setup/hold violations.
* **Slew/Transition Analysis:** Evaluates how fast a signal rises or falls; both data and clock transitions have defined min/max limits.
* **Load Analysis:** Checks fanout and capacitance loading on cells to maintain proper switching.
* **Clock Analysis:**

  * **Clock Skew:** Difference in clock arrival times at various flip-flops; excessive skew can cause setup/hold violations.
  * **Pulse Width:** Ensures the clock high and low phases are wide enough to trigger proper data capture.
  * **Clock Definition:** Clock period, duty cycle, uncertainty, latency, and skew are all part of STA clock constraints.

---

## **4. Reg-to-Reg Setup & Hold Analysis**

* Combinational block between launch and capture FF includes **cell**, **wire**, and **input transition** delays.
* **Actual Arrival Time (AAT):** Calculated by summing all delays forward from launch to capture.
* **Required Arrival Time (RAT):** Calculated backward from the capture FF toward the launch FF by subtracting delays.
* **Slack = RAT – AAT** indicates timing margin.
* **Graph-Based Analysis (GBA):** Considers worst-case conditions using delay graphs.
* **Path-Based Analysis (PBA):** More realistic — analyzes only the actual paths used by data, improving accuracy.
* **ECO (Engineering Change Order):** Used to fix negative slack by resizing or buffering critical cells.

<img width="1920" height="1080" alt="Screenshot (349)" src="https://github.com/user-attachments/assets/17a62a6c-4ddb-4744-9a83-546059e4ac50" />
<p align = "center"> Setup analysis for reg2reg type </p>

---

## **5. Flip-Flop Timing and Delays**

* A flip-flop internally has **two latches** (positive and negative).
* **Clk-to-Q Delay:** Time from clock edge to valid output (caused by inverters and pass transistors).
* **Setup Equation:**
  
  <p align = "center"> 
     $$\theta + \delta_1 < T_{clk} + \delta_2 - T_{setup}$$   
  </p>
  
  where θ = logic delay, δ₁ and δ₂ are clock arrival times at launch and capture FFs.
* Skew (δ₂ − δ₁) plays a crucial role in determining setup or hold margins.

---

## **6. Jitter and Eye Diagram**

* **Jitter:** Temporal variation of clock edges — introduces timing uncertainty.
* **Noise Margin:** Voltage fluctuation affecting signal stability.
* In STA, jitter is modeled as **clock uncertainty**, subtracted from the clock period to analyze worst-case timing.

---

## **7. Timing Report**

* Lists **cell delays**, **net delays**, **uncertainties**, and **setup/hold slack values** for all critical paths.
* Helps identify timing violations and guides optimization during synthesis and post-layout analysis.

<img width="1920" height="1080" alt="Screenshot (350)" src="https://github.com/user-attachments/assets/7617b793-f118-494a-8ae8-0c6664f167c9" />
<p align = "center"> Timing report </p>

---

## **8. On-Chip Variation (OCV)**

* Caused by process variations in **W/L** ratios and **oxide thickness (tox)** during fabrication.
* Variations affect current, delay, and thus timing paths.
* For **Setup Analysis:** RT reduced (clock pull-in), AT increased (clock push-out) to simulate worst case.
* For **Hold Analysis:** Reverse scenario applied.
* **Pessimism Removal:** Shared clock paths excluded from both AT and RT to avoid overestimation of delay impact.

---

## **9. Summary**

Static Timing Analysis (STA) is a **mathematical approach** to verify timing correctness of a digital design **without simulation**.
It ensures that every path in the circuit meets setup and hold requirements under variations in **process, voltage, temperature, and clock conditions**.
Key metrics like **arrival time, required time, slack, skew, and jitter** determine the design’s ability to operate reliably at the target clock frequency.
By identifying violations early and performing ECOs, STA enables **timing closure** — a critical step before chip fabrication.

---

