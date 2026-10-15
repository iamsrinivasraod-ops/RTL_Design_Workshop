# VSDBabySoC — RTL-to-Routing Physical Design Using OpenROAD

> **A hands-on ASIC implementation study of VSDBabySoC using the SKY130 HD technology platform, from synthesis and floorplanning through placement, clock-tree synthesis, routing, and timing/power analysis.**

## Overview

This project documents my practical experience taking **VSDBabySoC** through the major stages of an ASIC physical-design flow using **OpenROAD**. Rather than studying each implementation step in isolation, I examined how the synthesized logic becomes a physically organized design, how clock and power networks are built, how cells are connected by interconnect, and how timing and power reports help evaluate the result.

The objective was to understand the *dependencies between stages*: synthesis determines the available standard cells and connectivity; floorplanning allocates physical space and power infrastructure; placement determines cell locations; CTS builds a usable clock distribution network; and routing establishes the signal interconnections that make the layout physically meaningful.

The screenshots in this README were captured from my OpenROAD environment and terminal reports. They document the implementation stages reached and the observations made. **This is an RTL-to-routing implementation record, not a claim of completed signoff, LVS/DRC-clean tapeout, or fabricated silicon.**

## Tools and Design Environment

| Component | Role |
|---|---|
| **VSDBabySoC** | Top-level SoC design under implementation |
| **OpenROAD** | Physical design, GUI inspection, placement, CTS, routing, and analysis |
| **SKY130 HD (`sky130hd`)** | Open-source 130 nm standard-cell technology and associated design data |
| **Liberty (`.lib`)** | Cell timing, capacitance and power models |
| **LEF / technology LEF** | Cell dimensions, pins, routing layers and technology geometry |
| **Verilog netlist** | Synthesized connectivity between instances |
| **SDC constraints** | Clock and timing requirements |
| **Tcl / flow configuration** | Controls flow stages and analysis commands |
| **DEF** | Placement and physical interconnect representation between stages |
| **OpenSTA (within OpenROAD)** | Static timing analysis and critical-path reporting |

The OpenROAD GUI screenshots show the **`sky130hd/vsdbabysoc`** platform/design context and stage-specific databases such as `3_place`, `4_cts`, and `5_route`. The screenshots also show a `pll` instance, a macro-level block within the design.

## Implementation Flow

```text
VSDBabySoC RTL + libraries + constraints
                   |
                   v
         Logic synthesis
      (technology-mapped netlist)
                   |
                   v
       Floorplan + I/O planning
        + power distribution
                   |
                   v
       Standard-cell placement
       + placement optimization
                   |
                   v
        Clock Tree Synthesis
       + timing optimization
                   |
                   v
         Global / detailed routing
                   |
                   v
        Timing / power inspection
                   |
                   v
       Further physical verification
         (required for signoff)
```

Each stage uses the output of the previous one. A change to cell sizing, placement, clock buffers, or routing can affect both the timing and power of the final implementation.

---

## 1. Logic Synthesis — Turning RTL Into Standard Cells

The starting point is a hardware description of VSDBabySoC. During synthesis, RTL constructs are elaborated and transformed into a gate-level network. Technology mapping then selects cells from the SKY130 HD standard-cell library, producing a netlist that can be implemented physically.

### What synthesis establishes

- **Combinational logic:** arithmetic, control, decoding, multiplexing and other functions are expressed using available logic cells.
- **Sequential logic:** state is implemented using flip-flops and other supported sequential elements.
- **Connectivity:** a netlist describes which pins are connected, independent of their eventual physical positions.
- **Library dependence:** timing and area depend on the chosen cells, their drive strengths, and the target technology.
- **Optimization trade-offs:** synthesis can favor delay, area, or other constraints, but the final result must still be assessed after physical implementation.

The synthesized netlist is **not yet a physical layout**. It does not tell us whether nets are routable, how long interconnect will be, or whether timing remains acceptable once cells are placed and wires are introduced.

### Synthesis checks to understand

Before physical implementation, the critical checks include unresolved modules, unintended latches, unmapped cells, clock definitions, the number of sequential and combinational cells, and consistency between the netlist and technology libraries. For designs containing macros, the logical and physical macro views must also agree.

**Key learning:** A valid synthesized netlist is the starting point of physical design, not its final proof of correctness.

---

## 2. Floorplanning — Establishing the Physical Architecture

Floorplanning converts a logical design into a physical planning problem. The die boundary and core region must provide space for cells, macros, power distribution and routing resources. Cell utilization, aspect ratio, macro placement and available channels all influence how difficult the later stages become.

### What I examined

- **Die vs. core:** the die is the overall chip boundary, while the core is the main standard-cell placement region.
- **Utilization:** the fraction of available placement area occupied by cells; high utilization can worsen congestion and limit optimization flexibility.
- **Macro/block positioning:** larger fixed or semi-fixed structures require careful planning because they obstruct standard-cell placement and routing.
- **Rows and tracks:** standard cells align to placement rows, while routing resources are constrained by the process layer geometry.
- **Power distribution network (PDN):** power straps and connections distribute supply and ground to the cells and macros.

### Floorplan timing report

![Floorplan timing report showing worst slack and critical-path information](images/floorplanreport.png)

The floorplan report shows:

| Reported metric | Value | Interpretation |
|---|---:|---|
| WNS (`wns max`) | **0.00 ns** | No negative setup slack is accumulated in this report |
| TNS (`tns max`) | **0.00 ns** | No total negative setup slack is reported |
| Worst setup slack | **+1.33 ns** | The displayed worst path meets its timing requirement with margin |
| Reported minimum clock period | **10.67 ns** | Timing-based estimate under the report's stage-specific assumptions |
| Reported maximum frequency | **93.68 MHz** | Reciprocal of the minimum period, approximately |

The timing-path excerpt also shows a **12.00 ns clock period**, a **10.24 ns data arrival time**, and an **11.57 ns data required time**, giving **1.33 ns positive slack**. These values are *floorplan-stage estimates*, not post-route signoff results. The report displays an ideal clock network at this stage.

**Why this matters:** A positive floorplan slack is encouraging, but wire parasitics, clock-tree insertion, and subsequent optimization can change the timing picture.

### Floorplan power report

![Floorplan power report showing sequential and combinational power](images/floorplanpower.png)

The reported floorplan power is approximately **5.67 mW**, divided as follows:

| Group | Total power | Share |
|---|---:|---:|
| Sequential | **4.34 mW** | **76.5%** |
| Combinational | **1.33 mW** | **23.5%** |
| Clock | **0.00 mW** | **0.0%** |
| **Total** | **5.67 mW** | **100%** |

The report's total internal power is approximately **4.74 mW**, switching power approximately **0.925 mW**, and leakage approximately **1.80 × 10⁻⁸ W**. At this stage the clock group is reported as zero; this should not be interpreted as proof that the eventual clock network consumes no power.

**Key learning:** Floorplanning is where timing, area, routing access and power infrastructure first become tightly coupled physical-design decisions.

---

## 3. Placement — Giving Every Cell a Physical Location

Placement assigns coordinates to standard-cell instances inside the legal placement area. The tool aims to minimize wirelength and congestion while meeting timing and placement constraints. Global placement finds a good overall arrangement, and legalization ensures cells occupy valid, non-overlapping positions in placement rows.

The screenshots show the design at the **`3_place`** stage in OpenROAD, including the `pll` instance and the surrounding power-grid structures.

### Inspecting the PLL instance

![PLL instance viewed within the placement-stage physical design](images/pll%20instance.png)

This close-up is useful because it makes the distinction between a named logical instance and its **actual physical footprint** visible. The instance is shown within the larger design context, surrounded by metal shapes and physical-design infrastructure.

A PLL is typically involved in clock generation or conditioning, but the screenshot alone does not establish the exact PLL circuitry or its internal operating parameters. Its importance here is its role as a recognizable physical instance that must coexist with standard-cell placement and routing resources.

### Placement questions I learned to ask

1. Are the standard cells legal and inside the intended placement area?
2. Are macros and fixed instances positioned without obstructing critical access?
3. Is there sufficient routing capacity around dense cell regions?
4. Are long timing-critical nets being reduced where possible?
5. Can clock buffers and later timing-repair cells be inserted without severe congestion?

**Key learning:** Placement quality strongly influences the difficulty of CTS and routing. A physically legal placement is not automatically a timing-optimal or congestion-free placement.

---

## 4. Clock Tree Synthesis — Building a Real Clock Network

Clock Tree Synthesis (CTS) constructs the distribution network that delivers clock edges to sequential elements. Instead of assuming an ideal, zero-delay clock, the physical implementation introduces clock buffers, wires and measurable insertion delay.

The central objective is not simply to connect every clock pin: the clock network must manage **skew**, **insertion delay**, transition times, capacitance and timing constraints.

### Key CTS concepts

- **Clock insertion delay:** time taken for the clock to propagate from the defined clock source to a sink.
- **Clock skew:** difference in clock arrival time between clock sinks; it can affect setup and hold timing.
- **Clock buffers:** inserted to drive clock capacitance, control slew and distribute the clock efficiently.
- **Setup timing:** data must arrive sufficiently before the capturing clock edge.
- **Hold timing:** data must remain stable for the required interval after the capturing edge.
- **Post-CTS optimization:** timing can require buffer insertion, resizing, or other changes after the clock network is built.

### Clock-tree layout inspection

![Clock tree synthesis close-up showing physical interconnect and cells](images/ctscloser.png)

The `4_cts` OpenROAD view shows a dense physical region containing standard cells and multiple routing layers. The visible `clkbuf` label is particularly useful when discussing clock-buffer insertion and the transition from idealized clock assumptions to a physical clock distribution network.

### CTS timing and power results

![CTS critical-path delay, slack and power report](images/ctsreport.png)

The terminal report provides the following stage-specific values:

| CTS metric | Reported value |
|---|---:|
| Critical-path delay | **5.1979 ns** |
| Critical-path slack | **+7.5613 ns** |
| Total estimated power | **10.8 mW** |
| Sequential power | **4.38 mW (40.4%)** |
| Combinational power | **2.52 mW (23.3%)** |
| Clock power | **3.93 mW (36.3%)** |

The total power consists of approximately **6.92 mW internal**, **3.91 mW switching**, and **2.11 × 10⁻⁸ W leakage**. Small discrepancies between component sums and displayed totals are due to rounding.

#### Comparing floorplan and CTS power

| Power component | Floorplan | After CTS |
|---|---:|---:|
| Sequential | 4.34 mW | 4.38 mW |
| Combinational | 1.33 mW | 2.52 mW |
| Clock | 0.00 mW | 3.93 mW |
| **Total** | **5.67 mW** | **10.8 mW** |

The estimated total increases by approximately **5.13 mW**, or **90.5%**, between these two displayed reports. The newly reported clock network accounts for a substantial share of the CTS-stage estimate. These are *stage-specific modeled power estimates*, not silicon measurements; differences in activity assumptions and analysis conditions can also influence a comparison.

The CTS report's **+7.5613 ns slack** is positive for the displayed critical-path summary. It should **not** be directly treated as an improvement over the floorplan's +1.33 ns worst slack without confirming that both reports use identical timing constraints, path sets and analysis settings.

**Key learning:** The clock network has a real physical and power cost. CTS makes that cost visible and creates a new timing situation that must be analyzed after clock insertion.

---

## 5. Routing — Converting Connectivity Into Metal Interconnect

Routing establishes the physical connections between cell and macro pins. The router must select metal layers, create vias, avoid obstacles, and obey technology design rules while balancing wirelength, congestion and timing.

Two complementary concepts are important:

- **Global routing** estimates routes and allocates routing resources across the design.
- **Detailed routing** creates geometrically valid wire segments and vias within the available tracks, resolving local conflicts and rule constraints.

The OpenROAD screenshots show the **`5_route`** stage with signal-routing shapes displayed alongside the power network.

### Routed design overview

![Routed VSDBabySoC physical layout in OpenROAD](images/routing.png)

The design view contains multiple visible metal layers and a dense collection of signal interconnections. The horizontal and vertical wiring patterns illustrate how interconnect must be distributed across available layers rather than placed on a single plane.

### Routed interconnect close-up

![Detailed view of routed metal interconnect and vias](images/routingcloser.png)

The closer view highlights dense interconnect in the central implementation region. This is the point at which a logical netlist becomes a much more realistic physical connectivity problem: wires occupy space, require vias, and contribute resistance and capacitance.

### Routing challenges and what they mean

| Challenge | Why it matters |
|---|---|
| Congestion | Too many nets competing for limited routing tracks can cause detours or failed connections |
| Wirelength | Longer wires increase resistance and capacitance, affecting delay and power |
| Via count | Layer transitions add parasitics and must satisfy manufacturing rules |
| Macro obstructions | Fixed blocks can restrict routing channels and pin access |
| DRC constraints | Metal width, spacing and via geometries must satisfy the technology rules |
| Post-route timing | Extracted or estimated interconnect parasitics can change setup and hold slack |

**Key learning:** Routing is not merely drawing connections between gates. It is a constrained optimization problem where electrical behavior and manufacturability depend on physical geometry.

---

## 6. Visual Analysis — Heat Maps and Congestion Awareness

OpenROAD provides visualization tools to help interpret the implementation beyond raw netlists and terminal reports. Heat maps are especially useful because they turn spatially distributed physical-design information into a visual pattern.

### Heat-map overview

![Heat-map view of the VSDBabySoC design](images/heatmap.png)

The overview shows a localized concentration of colored regions inside the larger physical boundary. It makes it easier to see that design activity and resource demand are not necessarily uniform across the floorplan.

### Heat-map close-up

![Zoomed-in heat-map view of the implemented design](images/heatmapcloser.png)

The close-up reveals the local distribution more clearly. Such views can help guide investigation into high-density regions, possible congestion hot spots, or other spatially varying implementation metrics.

**Interpretation note:** The screenshots do not show a readable heat-map legend or a clearly selected heat-map metric. Therefore, the colors should **not** be assigned specific numerical congestion, temperature, utilization or timing values from these images alone. The exact meaning depends on the heat-map mode and scale selected in OpenROAD.

### What visual inspection contributes

Timing and power reports answer quantitative questions, while GUI views help identify *where* physical structures and potential problems are located. The strongest analysis combines both: locate an interesting region visually, inspect its instances and nets, then correlate the observation with a report.

---

## 7. Understanding Timing Reports Across the Flow

A static timing report checks whether data can travel through a path within the available timing budget. The basic setup relationship is:

```text
Setup slack = Data required time − Data arrival time
```

- **Positive slack** indicates the reported path meets its setup requirement.
- **Negative slack** indicates a setup timing violation for that path.
- **WNS** is the worst negative slack summary; depending on reporting convention it may be zero when all paths meet timing.
- **TNS** accumulates negative slack across violating endpoints.
- **Critical path** is the path that most constrains the relevant timing objective.

### Observed results

| Stage | Metric | Observation |
|---|---|---|
| Floorplan | Worst setup slack | **+1.33 ns** |
| Floorplan | WNS / TNS | **0.00 / 0.00 ns** |
| Floorplan | Estimated minimum period | **10.67 ns** |
| CTS | Critical-path delay | **5.1979 ns** |
| CTS | Critical-path slack | **+7.5613 ns** |

These are observations from **different stage reports**. A rigorous stage-to-stage timing comparison requires matching constraints, corners, path definitions, clock assumptions and parasitic models. The supplied images do not show a final routed timing report, so no post-route WNS or signoff frequency is asserted here.

## 8. Power Analysis — What the Numbers Reveal

Power in a digital circuit can be separated conceptually into:

- **Internal power:** energy dissipated within standard cells during operation.
- **Switching power:** charging and discharging load capacitances on nets.
- **Leakage power:** static current when transistors are nominally off.

The report comparison illustrates how the modeled power profile evolves as the physical implementation becomes more detailed:

| Reported power | Floorplan | CTS |
|---|---:|---:|
| Internal | 4.74 mW | 6.92 mW |
| Switching | 0.925 mW | 3.91 mW |
| Leakage | 0.000018 mW | 0.0000211 mW |
| **Total** | **5.67 mW** | **10.8 mW** |

The **clock category reaches 36.3% of the CTS total**, demonstrating why clock distribution is a major consideration in low-power physical design. These power estimates depend on the timing libraries, activity assumptions, operating conditions and analysis setup; they are not measured hardware consumption.

---

## 9. Major Technical Lessons

### RTL correctness is necessary, but physical feasibility is a separate challenge

A logically correct circuit can still have timing, congestion or routability problems. The physical design flow tests whether the mapped logic can be implemented under geometric and electrical constraints.

### Early decisions propagate through later stages

Cell mapping, macro placement, utilization and PDN geometry affect placement freedom and routing resources. CTS adds new cells and wires, which in turn change the power and timing landscape.

### Timing must be interpreted in context

A single positive slack value does not prove overall signoff. The timing corner, path type, clock assumptions, extraction stage and constraint set must be considered before drawing conclusions.

### Clock distribution is a major design subsystem

CTS is more than connecting the clock. The clock tree affects skew, insertion delay, transition quality, area and power. The measured stage estimates show how significant the clock-power contribution can become.

### Visualization and reports complement each other

OpenROAD's physical views reveal the organization of cells, macros and metal layers. Terminal reports quantify delay, slack and power. Understanding the design requires moving between both perspectives.

---

## 10. Verification and Signoff: What Comes Next

The supplied screenshots demonstrate synthesis-to-routing workflow progress and stage-level analysis, but do **not** provide evidence of all final signoff checks. A production-ready ASIC flow would additionally require, as applicable:

1. **Post-route STA** with appropriate extracted parasitics and all required timing corners.
2. **Setup and hold closure**, including clock uncertainty and all intended operating modes.
3. **DRC** to check the final physical geometries against process rules.
4. **LVS** to confirm that the extracted layout connectivity matches the intended schematic/netlist.
5. **Antenna and connectivity checks**, and other technology-specific physical verification.
6. **Power integrity analysis**, including IR drop and electromigration where supported and required.
7. **Final GDSII generation and inspection** after the required verification steps have passed.

This distinction matters: **reaching the routing stage is a substantial milestone, but it is not equivalent to signoff.**

---

## Screenshot Gallery and File Structure

All screenshots referenced above are stored in the `images/` directory relative to this README:

```text
VSDBabySoC/
├── README.md
└── images/
    ├── floorplanreport.png
    ├── floorplanpower.png
    ├── pll instance.png
    ├── ctscloser.png
    ├── ctsreport.png
    ├── routing.png
    ├── routingcloser.png
    ├── heatmap.png
    └── heatmapcloser.png
```

The screenshots are presented at the relevant stages rather than only in a gallery, so each visual is directly tied to its technical explanation.

## Conclusion

Taking VSDBabySoC through synthesis, floorplanning, placement, clock tree synthesis and routing gave me a practical understanding of how a digital design is transformed into a physical ASIC implementation. I learned to inspect real OpenROAD layouts, follow the physical-design sequence, interpret timing and power reports, and recognize the trade-offs between logic, clock distribution and interconnect.

The most valuable outcome was learning to **think beyond the RTL**: every standard cell needs a legal physical location, every signal requires a route, every clock sink must receive a usable clock, and every reported timing or power number must be understood in the context of its implementation stage.

This project strengthened my foundation in **ASIC physical design, OpenROAD, SKY130 technology, static timing analysis and PPA interpretation**, and provides a clear starting point for deeper post-route optimization and physical verification work.

---

*Documented as part of my hands-on VLSI design learning journey.*
