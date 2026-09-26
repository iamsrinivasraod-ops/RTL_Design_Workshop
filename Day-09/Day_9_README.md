# DAY 9 — SKY130 Module 4: Timing Analysis, CTS and Post-CTS STA

## Day 9 Introduction

Day 9 focuses on the timing-aware stage of the SKY130 ASIC physical design flow. The main objective is to understand how timing information, clock distribution, physical implementation, and Static Timing Analysis (STA) are connected.

Before moving into the detailed timing and CTS concepts, the following points are important for understanding how a clock network is physically built and analyzed.

### Tracks in Physical Design

Routing tracks are predefined routing locations in the physical layout where wires can be placed. They are associated with the routing layers and their preferred directions.

- Tracks provide a regular structure for physical routing.
- Standard-cell placement and routing must work within the available routing resources.
- The number and availability of tracks affect how easily signals and clock nets can be routed.
- Clock networks require sufficient routing resources because they connect many sequential elements and can have significant fanout.
- Physical routing, wire length, resistance, and capacitance directly affect delay.

### AND Gate and OR Gate as Buffers

An AND or OR gate can be configured to behave like a buffer when one of its inputs is held at a constant logic value.

For an AND gate:

```text
Y = A · 1 = A
```

For an OR gate:

```text
Y = A + 0 = A
```

Therefore, the gate can pass the input signal without changing its logic value.

In physical design, dedicated buffer cells are normally preferred because they are characterized specifically for buffering and provide suitable drive strength and timing characteristics. The AND/OR configuration demonstrates the basic idea that a logic gate can be used to reproduce a signal while providing a driving stage.

### Why Each Node Should Drive a Similar Load

A clock tree is designed so that branches are balanced as much as possible. If one node drives a much larger load than another node, its transition and propagation delay can become different.

The important reasons for keeping the load of comparable branches similar are:

- Similar load produces more similar delay.
- Similar delay helps balance clock arrival times.
- Balanced arrival times reduce clock skew.
- Large differences in capacitance can cause different slew rates and delay.
- Unequal loading can make timing closure more difficult.

Thus, load balancing is an important part of building a predictable clock distribution network.

### Why Identical Buffers Are Used at the Same Level

Using identical buffer cells at the same level of a balanced clock tree helps make the branches electrically similar.

If the same buffer type and drive strength are used:

- The intrinsic cell delay is similar.
- The drive characteristics are similar.
- The output transition behavior is more predictable.
- Differences between branches are mainly determined by their physical load and interconnect.

This makes the clock tree easier to balance and helps control skew.

Using a different buffer size or type may be necessary when the loads are different, but a balanced tree generally tries to keep corresponding branches electrically and physically similar.

### Power-Aware CTS

Clock networks are one of the major sources of dynamic power because the clock switches continuously and drives many sequential elements.

Power-aware Clock Tree Synthesis considers both timing and power while constructing the clock network.

Important considerations include:

- Clock buffer count.
- Buffer drive strength.
- Clock wire length.
- Capacitive load.
- Clock transition time.
- Clock latency and skew.
- Switching activity of the clock network.

Larger buffers can drive larger loads and may improve transition time, but they can also increase power and area. Therefore, CTS attempts to satisfy timing requirements without unnecessarily increasing the clock network's power consumption.

### Delay Tables

Standard-cell libraries contain characterized timing information that relates input slew and output load to cell delay and output transition.

A simplified view is:

```text
Input Slew + Output Load
          ↓
     Delay Table
          ↓
      Cell Delay
```

The STA engine uses these characterized values to estimate how long a signal takes to propagate through a cell under the actual electrical conditions of the path.

### Clock Skew

Clock skew is the difference in clock arrival time between two relevant sequential elements.

```text
Skew = |Δ1 - Δ2|
```

If the clock reaches two flip-flops at different times, the available timing window between them changes. Therefore, CTS attempts to control skew by balancing clock paths.

Skew is affected by:

- Clock-buffer delay.
- Interconnect delay.
- Wire resistance and capacitance.
- Different path lengths.
- Different loads.
- Clock-tree structure.

### Why Clock-Tree Branches Are Balanced

A balanced clock tree attempts to make corresponding clock paths have similar electrical and physical characteristics.

This means considering:

- Similar path lengths.
- Similar loads.
- Similar buffer stages.
- Similar buffer drive strengths.
- Similar routing conditions.

The goal is not necessarily to make every physical path perfectly identical, but to make clock arrival times sufficiently close to satisfy timing requirements while keeping power and area under control.

---

## Overview

Day 9 focuses on the timing-aware stage of the SKY130 ASIC physical design flow. The work connects **standard-cell timing models**, **setup/hold timing**, **clock tree synthesis (CTS)**, **placement**, **clock skew**, and **static timing analysis (STA)**.

The objective is to understand how a synthesized digital design is analyzed and optimized for timing after physical implementation, and how clock distribution affects the timing of sequential logic.

---

# 1. Timing Modeling and Delay Tables

Timing analysis begins with accurate cell timing information. Standard-cell libraries contain characterized delay and transition information for different combinations of:

- Input slew
- Output load capacitance
- Cell type
- Input/output pin transitions

Delay tables allow the STA engine to estimate the propagation delay of a standard cell under a particular electrical condition.

### Important concepts

- **Input slew** — transition time of the input signal.
- **Output load** — capacitive load driven by the cell.
- **Cell delay** — propagation delay through the standard cell.
- **Buffer delay** — delay introduced by clock/data buffering.
- **Slew degradation** — change in transition quality as a signal propagates through logic.

### Delay Table — Buffering Level 1

[Delay Table Buffering Level 1](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Delay_Table_Buffering_1.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Delay_Table_Buffering_1.png))

### Delay Table — Buffering Level 2

[Delay Table Buffering Level 2](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Delay_Table_Buffering_2.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Delay_Table_Buffering_2.png))

### Delay Components

[Delta Delay](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Delta_delay.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Delta_delay.png))

The delay observed on a physical path is not determined only by the logic-cell delay. Interconnect resistance, capacitance, buffering, and physical wire length also contribute to the total delay.

---

# 2. Setup and Hold Timing Analysis

Sequential timing analysis verifies whether data reaches a flip-flop within the required timing window relative to the clock edge.

## 2.1 Setup Timing

Setup analysis checks whether data arrives sufficiently **before the active clock edge**.

Conceptually:

```
Launch FF ─── Combinational Logic ─── Capture FF
    │                                  │
 Launch Clock                      Capture Clock

```

**svg**

If data arrives too late, a **setup violation** occurs.

### Setup Analysis with Ideal Clock

[Setup Analysis — Ideal Clock](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Setup_analysis_ideal_clk.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Setup_analysis_ideal_clk.png))

### Setup Analysis with Real Clock

[Setup Analysis — Real Clock](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Setup_Analysis_real_clk.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Setup_Analysis_real_clk.png))

### Setup Timing Concept

[Setup Time](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Setup_time.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Setup_time.png))

---

## 2.2 Hold Timing

Hold analysis checks whether the data remains stable for the required interval **after the active clock edge**.

If new data reaches the capture flip-flop too early, a **hold violation** can occur.

### Hold Analysis with Ideal Clock

[Hold Analysis — Ideal Clock](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/hold_analysis_ideal_clk.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/hold_analysis_ideal_clk.png))

### Hold Analysis with Real Clock

[Hold Analysis — Real Clock](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Hold_Analysis_real_clk.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Hold_Analysis_real_clk.png))

### Hold Timing Concept

[Hold Time](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Hold_time.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Hold_time.png))

---

## 2.3 Ideal Clock vs Real Clock

An ideal clock assumes an idealized clock arrival relationship without physical clock-network effects.

After CTS, the clock is distributed through an actual network containing:

- Clock buffers
- Interconnect
- Wire resistance
- Wire capacitance
- Different path lengths

Therefore, real-clock analysis includes clock insertion delay and clock skew.

---

# 3. Clock Tree Synthesis (CTS)

Clock Tree Synthesis creates a physical clock distribution network from the clock source to the sequential elements in the design.

The primary objectives of CTS include:

- Delivering the clock to all required sequential elements.
- Controlling clock latency.
- Reducing clock skew.
- Maintaining acceptable transition characteristics.
- Building a physically realizable clock network.

---

## 3.1 H-Tree Clock Distribution

An H-tree is a symmetric clock-distribution structure intended to provide similar path lengths to different branches.

[CTS H-Tree](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/CTS_H-Tree.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/CTS_H-Tree.png))

---

## 3.2 Clock Buffers

Clock buffers are inserted to drive the capacitive load of the clock network and maintain acceptable signal transition characteristics.

[CTS Buffer](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/CTS_Buffer.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/CTS_Buffer.png))

---

## 3.3 Clock Net Shielding

Clock nets are sensitive to coupling and noise. Shielding can be used to reduce unwanted capacitive coupling from neighboring signal wires.

[CTS Net Shielding](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/CTS_Net_shielding.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/CTS_Net_shielding.png))

---

## 3.4 CTS Terminal / Clock Network

The resulting clock network connects the clock source to the required sequential elements through the synthesized clock distribution structure.

[CTS Terminal](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/CTS_Terminal.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/CTS_Terminal.png))

---

# 4. Physical Implementation and Placement

Timing is strongly affected by physical implementation because interconnect delay depends on the actual geometry of the design.

The physical design stage therefore considers:

- Floorplan
- Placement
- Cell locations
- Routing resources
- Interconnect length
- Parasitic effects

### Floorplan / Terminal View

[Floorplan Terminal](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/floorplan_terminal.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/floorplan_terminal.png))

### Layout Grid

[Layout Grid](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Layout_grid.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Layout_grid.png))

### Placement

[Placement](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/placement.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/placement.png))

### Placement — Additional View

[Placement 1](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/placement_1.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/placement_1.png))

### Expanded Placement View

[Placement Expanded](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/placement_expand.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/placement_expand.png))

### Placement Terminal View

[Placement Terminal](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/placement_terminal.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/placement_terminal.png))

---

# 5. Post-CTS Timing Effects

After CTS, timing analysis becomes more realistic because the clock network is physically implemented.

Important effects include:

- Clock insertion delay
- Clock skew
- Clock uncertainty
- Clock-buffer delay
- Interconnect RC delay
- Crosstalk / coupling effects
- Clock waveform degradation

---

## 5.1 Clock Skew

Clock skew is the difference in clock arrival time between two relevant sequential elements.

For two clock paths:

```
Skew = |Δ1 - Δ2|

```

**svg**

where the clock arrival times are affected by the physical clock network and its associated delays.

[Clock Skew](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Skew.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Skew.png))

---

## 5.2 Glitch Analysis

Clock and signal integrity must also be considered during physical implementation. Unwanted transitions or glitches can affect timing and functional behavior depending on where they occur.

[Glitch Analysis](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/Glitch.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/Glitch.png))

---

# 6. Static Timing Analysis (STA)

Static Timing Analysis verifies timing behavior without requiring exhaustive functional simulation of every possible input sequence.

STA analyzes timing paths and determines whether the design satisfies its timing constraints.

Important quantities include:

### Data Arrival Time

The time at which data reaches the capture point.

### Data Required Time

The latest time by which data must arrive to satisfy the timing constraint.

### Slack

Slack represents the timing margin.

For setup analysis:

```
Slack = Data Required Time - Data Arrival Time

```

**svg**

A negative setup slack indicates that the analyzed path does not meet the corresponding setup requirement.

---

## STA Output

[STA Report](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/STA.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/STA.png))

[STA Report — Additional](https://github.com/Moursagna/VSDIAT-Chip-Design/blob/main/Day_9/images/STA_1.png) ([image](https://github.com/Moursagna/VSDIAT-Chip-Design/raw/main/Day_9/images/STA_1.png))

The terminal reports contain timing-path information such as cell delays, net delays, clock information, data arrival time, data required time, and slack.

---

# 7. WNS and TNS

Two important summary metrics used during timing analysis are:

## Worst Negative Slack (WNS)

WNS represents the most negative slack among the analyzed violating paths.

```
WNS = minimum slack

```

**svg**

For a timing-clean group of paths, the corresponding worst slack is non-negative.

## Total Negative Slack (TNS)

TNS represents the accumulated negative slack across violating paths.

```
TNS = sum of negative slacks

```

**svg**

WNS identifies the worst individual timing margin, while TNS indicates the aggregate magnitude of timing violations.

---

# 9. Practical VLSI Significance

Day 9 connects timing theory with actual ASIC implementation.

A design may be logically correct at RTL and still fail timing after synthesis or physical implementation because:

- Logic paths have finite propagation delay.
- Standard cells have different delay characteristics.
- Interconnect contributes resistance and capacitance.
- Clock paths have insertion delay.
- Different clock paths can have different arrival times.
- Setup and hold constraints must both be satisfied.

Therefore, timing closure is an iterative process involving RTL, synthesis, cell selection, buffering, placement, CTS, routing, parasitic extraction, and STA.

---

# 10. Day 9 Key Learnings

By completing this module, the following concepts were studied:

- Standard-cell timing characterization
- Input slew and output load
- Delay tables
- Cell and buffer delay
- Setup timing
- Hold timing
- Ideal-clock timing analysis
- Real-clock timing analysis
- Clock Tree Synthesis
- H-tree clock distribution
- Clock buffering
- Clock net shielding
- Placement and physical implementation
- Clock insertion delay
- Clock skew
- Glitch analysis
- Static Timing Analysis
- Data arrival time
- Data required time
- Slack
- WNS
- TNS
- Timing violations and timing closure

---

# 11. Day 9 Completion Checklist

-  Studied timing-model concepts
-  Studied delay tables and buffering
-  Studied setup timing
-  Studied hold timing
-  Compared ideal and real clock behavior
-  Studied Clock Tree Synthesis
-  Studied H-tree clock distribution
-  Studied clock buffering
-  Studied clock net shielding
-  Reviewed physical placement
-  Studied clock skew
-  Studied glitch behavior
-  Reviewed STA reports
-  Studied WNS and TNS
-  Connected timing analysis with physical implementation

---

## Conclusion

Day 9 establishes the connection between **timing models, clock distribution, physical implementation, and STA** in the SKY130 ASIC flow.

The central idea is:

> **Physical implementation changes timing, and STA is used to measure whether the implemented design satisfies its timing constraints.**

This forms the foundation for understanding timing closure in a complete ASIC design flow.
