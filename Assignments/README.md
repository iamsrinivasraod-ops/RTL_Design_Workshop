# VSDBabySoC — Comprehensive Static Timing Analysis Assignments
**OpenSTA Library Output Analysis + OpenROAD Multi-Corner STA | SKY130HD**  
**D. Srinivas Rao · 24EG104A06 · ECE-A**

> Two independent timing-analysis exercises on VSDBabySoC. Assignment 1 uses an **11 ns** clock and documents **15** reported corners. Assignment 2 uses a **12 ns** clock and documents **16** corners. Their results are not interchangeable, and both are pre-route/netlist-based analyses rather than verified extracted-parasitic signoff.

## Contents
- [Project background](#project-background)
- [STA fundamentals](#sta-fundamentals)
- [Assignment 1 — OpenSTA library analysis](#assignment-1--opensta-library-analysis)
- [Assignment 2 — OpenROAD 16-corner STA](#assignment-2--openroad-16-corner-sta)
- [Comparison and interpretation](#comparison-and-interpretation)
- [Limitations and future work](#limitations-and-future-work)
- [References](#references)

## Project background
VSDBabySoC integrates a PicoRV32-based CPU core with analog-oriented DAC and PLL macros. These assignments study the timing of the digital design through the SkyWater SKY130 high-density standard-cell timing models. The objective is to determine how cell propagation delays, setup constraints and timing margins change with process, voltage and temperature (PVT).

### Tools and inputs
| Item | Role |
|---|---|
| OpenSTA | Static timing engine for library/path analysis |
| OpenROAD | Integrated environment running multi-corner timing reports |
| SKY130HD Liberty (`.lib`) | Cell timing, slew, load, setup and hold characterization |
| Synthesized Verilog netlist | Structural connectivity of mapped cells |
| SDC constraints | Clock period and timing requirements |
| LEF files | Technology, placement geometry and macro descriptions (Assignment 2) |
| `avsddac.lib` and `avsdpll.lib` | Timing models for the two macros |
| Tcl | Automated loading and report generation |

## STA fundamentals
Static Timing Analysis checks timing paths without running stimulus-driven simulation. A **launch flip-flop** emits data following a clock edge; the data traverses a combinational network and must arrive at the **capture flip-flop** early enough to satisfy its setup requirement. For setup analysis, the reported path type is `max`.

```text
Launch FF (Q) -> combinational cells / nets -> Capture FF (D)
        clock-to-Q + logic/network delay        setup requirement
```

**Setup slack = data required time − data arrival time.** A positive slack means the *reported path* meets setup; a negative slack indicates a setup violation. A value of zero is exactly at the reported limit. Arrival time includes the path delays under the loaded timing models. Required time incorporates clock relationship and endpoint setup requirement. The raw clock period is therefore not necessarily identical to the data required time.

| Term | Meaning |
|---|---|
| Startpoint / Endpoint | Launching and capturing sequential elements |
| `Fanout` | Number of loads on a node/cell output |
| `Cap` | Electrical load capacitance in the report units |
| `Slew` | Transition time of the signal |
| `Delay` | Incremental delay contributed by a timing arc |
| `Time` | Accumulated path time |
| `^` / `v` | Rising / falling transition |
| `MET` / `VIOLATED` | Positive/nonnegative or negative path slack |
| WNS | Worst reported setup slack over the relevant analysis scope; some tools clamp the displayed worst-negative metric to zero when all paths pass |
| TNS | Sum of negative endpoint slacks over the relevant analysis scope |

### PVT nomenclature
`sky130_fd_sc_hd__ss_n40C_1v28.lib` means **SkyWater 130 nm**, **foundry-provided high-density standard cells**, **slow-slow process**, **−40 °C**, **1.28 V**. `tt` = typical-typical; `ss` = slow-slow; `ff` = fast-fast. Higher voltage commonly speeds up cells, but temperature trends can depend on the characterization and operating region. Neither a nominal process label nor a single fast path guarantees full-chip timing closure.

---
## Assignment 1 — OpenSTA Library Analysis
### Objective and setup
The first assessment used OpenSTA to load the synthesized VSDBabySoC netlist, its timing constraints and SKY130HD Liberty files, then inspect maximum-delay setup paths across **15 documented corners** under an **11 ns** clock period (approximately **90.91 MHz**). The original terminal output includes repeated-library warnings because the Tcl loop reloads libraries in the same session; the numerical results below reproduce the assessment, not an independently rerun STA.

### Complete corner results (original report order)
| # | Liberty corner | Arrival (ns) | Required (ns) | Slack (ns) | Status |
|---:|---|---:|---:|---:|---|
| 1 | `sky130_fd_sc_hd__tt_025C_1v80.lib` | 8.90 | 10.84 | +1.94 | MET |
| 2 | `sky130_fd_sc_hd__tt_100C_1v80.lib` | 8.81 | 10.90 | +2.09 | MET |
| 3 | `sky130_fd_sc_hd__ss_n40C_1v28.lib` | 71.70 | 9.14 | -62.56 | VIOLATED |
| 4 | `sky130_fd_sc_hd__ss_n40C_1v35.lib` | 49.47 | 9.77 | -39.70 | VIOLATED |
| 5 | `sky130_fd_sc_hd__ss_n40C_1v40.lib` | 39.74 | 10.04 | -29.69 | VIOLATED |
| 6 | `sky130_fd_sc_hd__ss_n40C_1v44.lib` | 34.10 | 10.20 | -23.91 | VIOLATED |
| 7 | `sky130_fd_sc_hd__ss_n40C_1v60.lib` | 21.15 | 10.54 | -10.61 | VIOLATED |
| 8 | `sky130_fd_sc_hd__ss_n40C_1v76.lib` | 15.07 | 10.69 | -4.38 | VIOLATED |
| 9 | `sky130_fd_sc_hd__ss_100C_1v40.lib` | 27.54 | 10.42 | -17.12 | VIOLATED |
| 10 | `sky130_fd_sc_hd__ss_100C_1v60.lib` | 18.51 | 10.64 | -7.87 | VIOLATED |
| 11 | `sky130_fd_sc_hd__ff_n40C_1v56.lib` | 8.64 | 10.92 | +2.28 | MET |
| 12 | `sky130_fd_sc_hd__ff_n40C_1v65.lib` | 7.52 | 10.93 | +3.41 | MET |
| 13 | `sky130_fd_sc_hd__ff_n40C_1v76.lib` | 6.52 | 10.95 | +4.43 | MET |
| 14 | `sky130_fd_sc_hd__ff_100C_1v65.lib` | 6.88 | 10.93 | +4.05 | MET |
| 15 | `sky130_fd_sc_hd__ff_100C_1v95.lib` | 5.39 | 10.96 | +5.57 | MET |

**Path identity:** the launch point for the listed analyses is `core/CPU_is_addi_a3_reg`. For the TT and SS results, the documented endpoint is `core/CPU_Xreg_value_a4_reg[27][30]`. FF endpoint choices vary: `[16][31]` at −40 °C/1.56–1.65 V, `[1][31]` at −40 °C/1.76 V, and `[16][30]` at 100 °C/1.65–1.95 V. The path traverses a deep chain including AOI/OAI gates, arithmetic and logic cells; path dominance can shift with PVT.

### Corner-by-corner interpretation
**1. TT, 25 °C, 1.80 V:** arrival **8.90 ns**, required **10.84 ns**, slack **+1.94 ns** (**MET**). The path meets its reported setup deadline with positive margin.

**2. TT, 100 °C, 1.80 V:** arrival **8.81 ns**, required **10.90 ns**, slack **+2.09 ns** (**MET**). The path meets its reported setup deadline with positive margin.

**3. SS, −40 °C, 1.28 V:** arrival **71.70 ns**, required **9.14 ns**, slack **-62.56 ns** (**VIOLATED**). The path misses the deadline by 62.56 ns.

**4. SS, −40 °C, 1.35 V:** arrival **49.47 ns**, required **9.77 ns**, slack **-39.70 ns** (**VIOLATED**). The path misses the deadline by 39.70 ns.

**5. SS, −40 °C, 1.40 V:** arrival **39.74 ns**, required **10.04 ns**, slack **-29.69 ns** (**VIOLATED**). The path misses the deadline by 29.69 ns.

**6. SS, −40 °C, 1.44 V:** arrival **34.10 ns**, required **10.20 ns**, slack **-23.91 ns** (**VIOLATED**). The path misses the deadline by 23.91 ns.

**7. SS, −40 °C, 1.60 V:** arrival **21.15 ns**, required **10.54 ns**, slack **-10.61 ns** (**VIOLATED**). The path misses the deadline by 10.61 ns.

**8. SS, −40 °C, 1.76 V:** arrival **15.07 ns**, required **10.69 ns**, slack **-4.38 ns** (**VIOLATED**). The path misses the deadline by 4.38 ns.

**9. SS, 100 °C, 1.40 V:** arrival **27.54 ns**, required **10.42 ns**, slack **-17.12 ns** (**VIOLATED**). The path misses the deadline by 17.12 ns.

**10. SS, 100 °C, 1.60 V:** arrival **18.51 ns**, required **10.64 ns**, slack **-7.87 ns** (**VIOLATED**). The path misses the deadline by 7.87 ns.

**11. FF, −40 °C, 1.56 V:** arrival **8.64 ns**, required **10.92 ns**, slack **+2.28 ns** (**MET**). The path meets its reported setup deadline with positive margin.

**12. FF, −40 °C, 1.65 V:** arrival **7.52 ns**, required **10.93 ns**, slack **+3.41 ns** (**MET**). The path meets its reported setup deadline with positive margin.

**13. FF, −40 °C, 1.76 V:** arrival **6.52 ns**, required **10.95 ns**, slack **+4.43 ns** (**MET**). The path meets its reported setup deadline with positive margin.

**14. FF, 100 °C, 1.65 V:** arrival **6.88 ns**, required **10.93 ns**, slack **+4.05 ns** (**MET**). The path meets its reported setup deadline with positive margin.

**15. FF, 100 °C, 1.95 V:** arrival **5.39 ns**, required **10.96 ns**, slack **+5.57 ns** (**MET**). The path meets its reported setup deadline with positive margin.

### Assignment 1 findings
- **7 of 15** reported paths meet setup: 2 TT and 5 FF; **8 of 15** fail, all at SS corners.
- The most severe documented path is **SS, −40 °C, 1.28 V: −62.56 ns**, with arrival **71.70 ns** against required **9.14 ns**.
- The highest positive path margin is **FF, 100 °C, 1.95 V: +5.57 ns**.
- At SS/−40 °C, increasing voltage from **1.28 V to 1.76 V** reduces arrival from **71.70 ns to 15.07 ns**, an improvement of **56.63 ns**; slack improves from **−62.56 to −4.38 ns**, yet remains violating.
- At TT/1.80 V, the 100 °C report shows slightly better slack than 25 °C (+2.09 vs +1.94 ns), illustrating that temperature dependence is not always monotonic.

### Original Tcl workflow — Assignment 1
The submitted script declares the 15 `.lib` filenames, then loops over them, reading each selected standard-cell library for `-min` and `-max`, reading `avsddac.lib` and `avsdpll.lib`, importing `vsdbabysoc.synth.v`, linking the `vsdbabysoc` top module, reading `vsdbabysoc_synthesis.sdc`, and calling `report_checks`. Its original absolute paths are under `/data/Desktop/OpenSTA/vsdbabysoc/`. This is a record of the submitted methodology, **not** a recommendation to repeatedly reload an already-linked design and all libraries in one process.

```tcl
# Representative commands from the submitted assessment
read_liberty -min /data/Desktop/OpenSTA/vsdbabysoc/lib/$lib
read_liberty -max /data/Desktop/OpenSTA/vsdbabysoc/lib/$lib
read_verilog /data/Desktop/OpenSTA/vsdbabysoc/vsdbabysoc.synth.v
link_design vsdbabysoc
read_sdc /data/Desktop/OpenSTA/vsdbabysoc/sdc/vsdbabysoc_synthesis.sdc
report_checks
```

### WNS and TNS explained
If endpoint slacks are `+4, −2, −5, −3, +1 ns`, then **WNS = −5 ns** and **TNS = −10 ns**. WNS identifies the single worst failing margin; TNS measures the aggregate negative margin across failing endpoints. Do **not** add the 15 path slacks in the table and call the result the design TNS: the table is one reported path per PVT condition, not a complete endpoint-slack census in one analysis corner.

---
## Assignment 2 — OpenROAD 16-Corner STA
### Objective and analysis setup
The second assignment expanded the experiment to **16** named SKY130HD PVT corners using OpenROAD/OpenSTA and a **12 ns** clock period (**83.33 MHz**). It additionally defines timing corners explicitly, loads technology/merged LEFs and DAC/PLL macro LEFs, and prints reports for each corner directly in the terminal. Although the PDF cover says “post-route timing,” its shown Tcl reads `1_synth.v` and **does not read a routed DEF, SPEF or extracted parasitics**; accordingly these are **post-synthesis, ideal-clock timing results**, not proven post-route STA.

### Full corner-by-corner results (original script order)
| # | Process | Temp (°C) | Voltage (V) | Arrival (ns) | Slack (ns) | Status |
|---:|---|---:|---:|---:|---:|---|
| 1 | FF | +100 | 1.65 | 2.64 | +9.26 | MET |
| 2 | FF | +100 | 1.95 | 2.05 | +9.88 | MET |
| 3 | FF | -40 | 1.56 | 3.46 | +8.40 | MET |
| 4 | FF | -40 | 1.65 | 2.98 | +8.90 | MET |
| 5 | FF | -40 | 1.76 | 2.56 | +9.34 | MET |
| 6 | FF | -40 | 1.95 | 2.08 | +9.84 | MET |
| 7 | SS | +100 | 1.40 | 9.93 | +1.57 | MET |
| 8 | SS | +100 | 1.60 | 6.76 | +4.95 | MET |
| 9 | SS | -40 | 1.28 | 27.48 | -17.16 | VIOLATED |
| 10 | SS | -40 | 1.35 | 18.81 | -7.89 | VIOLATED |
| 11 | SS | -40 | 1.40 | 15.11 | -3.95 | VIOLATED |
| 12 | SS | -40 | 1.44 | 12.95 | -1.65 | VIOLATED |
| 13 | SS | -40 | 1.60 | 8.08 | +3.53 | MET |
| 14 | SS | -40 | 1.76 | 5.78 | +5.97 | MET |
| 15 | TT | +25 | 1.80 | 3.41 | +8.46 | MET |
| 16 | TT | +100 | 1.80 | 3.32 | +8.56 | MET |

**Reported counts:** 12 MET and 4 VIOLATED. All four violating sampled paths occur in **SS, −40 °C**, at **1.28, 1.35, 1.40 and 1.44 V**. The worst sampled path has **27.48 ns** arrival, **10.32 ns** required time and **−17.16 ns** slack. The fastest sampled arrival is **2.05 ns** at FF/100 °C/1.95 V; that path has **+9.88 ns** slack.

### Interpretation of each measured corner
**1. FF, +100 °C, 1.65 V:** arrival **2.64 ns**; setup slack **+9.26 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**2. FF, +100 °C, 1.95 V:** arrival **2.05 ns**; setup slack **+9.88 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**3. FF, -40 °C, 1.56 V:** arrival **3.46 ns**; setup slack **+8.40 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**4. FF, -40 °C, 1.65 V:** arrival **2.98 ns**; setup slack **+8.90 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**5. FF, -40 °C, 1.76 V:** arrival **2.56 ns**; setup slack **+9.34 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**6. FF, -40 °C, 1.95 V:** arrival **2.08 ns**; setup slack **+9.84 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**7. SS, +100 °C, 1.40 V:** arrival **9.93 ns**; setup slack **+1.57 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**8. SS, +100 °C, 1.60 V:** arrival **6.76 ns**; setup slack **+4.95 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**9. SS, -40 °C, 1.28 V:** arrival **27.48 ns**; setup slack **-17.16 ns** (**VIOLATED**). The reported path violates setup by 17.16 ns.

**10. SS, -40 °C, 1.35 V:** arrival **18.81 ns**; setup slack **-7.89 ns** (**VIOLATED**). The reported path violates setup by 7.89 ns.

**11. SS, -40 °C, 1.40 V:** arrival **15.11 ns**; setup slack **-3.95 ns** (**VIOLATED**). The reported path violates setup by 3.95 ns.

**12. SS, -40 °C, 1.44 V:** arrival **12.95 ns**; setup slack **-1.65 ns** (**VIOLATED**). The reported path violates setup by 1.65 ns.

**13. SS, -40 °C, 1.60 V:** arrival **8.08 ns**; setup slack **+3.53 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**14. SS, -40 °C, 1.76 V:** arrival **5.78 ns**; setup slack **+5.97 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**15. TT, +25 °C, 1.80 V:** arrival **3.41 ns**; setup slack **+8.46 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

**16. TT, +100 °C, 1.80 V:** arrival **3.32 ns**; setup slack **+8.56 ns** (**MET**). The reported path satisfies the 12 ns timing constraint.

### Full `run.tcl` from the submitted assignment
```tcl
# ============================================================
# VSDBabySoC - Multi-Corner STA
# Terminal Output Only
# ============================================================
set base [pwd]
set lib_dir "$base/platforms/sky130hd/lib"
set lef_dir "$base/platforms/sky130hd/lef"
set macro_lib_dir "$base/designs/sky130hd/vsdbabysoc/lib"
set macro_lef_dir "$base/designs/sky130hd/vsdbabysoc/lef"
# ============================================================
# All 16 PVT Libraries
# ============================================================
set libs {
  sky130_fd_sc_hd__ff_100C_1v65.lib
  sky130_fd_sc_hd__ff_100C_1v95.lib
  sky130_fd_sc_hd__ff_n40C_1v56.lib
  sky130_fd_sc_hd__ff_n40C_1v65.lib
  sky130_fd_sc_hd__ff_n40C_1v76.lib
  sky130_fd_sc_hd__ff_n40C_1v95.lib
  sky130_fd_sc_hd__ss_100C_1v40.lib
  sky130_fd_sc_hd__ss_100C_1v60.lib
  sky130_fd_sc_hd__ss_n40C_1v28.lib
  sky130_fd_sc_hd__ss_n40C_1v35.lib
  sky130_fd_sc_hd__ss_n40C_1v40.lib
  sky130_fd_sc_hd__ss_n40C_1v44.lib
  sky130_fd_sc_hd__ss_n40C_1v60.lib
  sky130_fd_sc_hd__ss_n40C_1v76.lib
  sky130_fd_sc_hd__tt_025C_1v80.lib
  sky130_fd_sc_hd__tt_100C_1v80.lib
}
# ============================================================
# Define Timing Corners
# ============================================================
set corners {}
foreach lib $libs {
  lappend corners [file rootname $lib]
}
define_corners {*}$corners
# ============================================================
# Load LEF Files
# ============================================================
read_lef "$lef_dir/sky130_fd_sc_hd.tlef"
read_lef "$lef_dir/sky130_fd_sc_hd_merged.lef"
foreach macro {avsddac avsdpll} {
  set lef_file "$macro_lef_dir/${macro}.lef"
  if {[file exists $lef_file]} {
     read_lef $lef_file
  }
}


# ============================================================
# Load Liberty Libraries
# ============================================================
foreach lib $libs {
  set corner [file rootname $lib]
  puts "Loading corner: $corner"
  read_liberty -corner $corner "$lib_dir/$lib"
  foreach macro {avsddac avsdpll} {

     set macro_file "$macro_lib_dir/${macro}.lib"

     set f [open $macro_file r]
     set content [read $f]
     close $f

     regsub -all \
       {library[ \t]*\([ \t]*([a-zA-Z0-9_]+)[ \t]*\)} \
       $content \
       "library (${macro}_${corner})" \
       content

     set tmp_file "/tmp/${macro}_${corner}.lib"

     set f [open $tmp_file w]
     puts -nonewline $f $content
     close $f

     read_liberty -corner $corner $tmp_file

     file delete $tmp_file
 }
}
# ============================================================
# Load Synthesized Netlist
# ============================================================
read_verilog "$base/results/sky130hd/vsdbabysoc/base/1_synth.v"
link_design vsdbabysoc
# ============================================================
# Load Timing Constraints
# ============================================================
read_sdc "$base/designs/sky130hd/vsdbabysoc/vsdbabysoc_synthesis.sdc"
# ============================================================
# Display STA Reports in Terminal
# ============================================================
foreach lib $libs {
   set corner [file rootname $lib]
   puts "\n=========================================="
   puts "STA CORNER: $corner"
   puts "=========================================="



    puts "\n========== REPORT CHECKS =========="
    report_checks -corner $corner

    puts "\n========== WNS =========="
    report_worst_slack -max

    puts "\n========== TNS =========="
    report_tns -max
}

puts "\n=========================================="
puts "STA COMPLETED FOR ALL 16 PVT CORNERS"
puts "=========================================="
```

### Tcl execution sequence explained
1. **Resolve paths:** `set base [pwd]` makes the flow relative to the directory from which OpenROAD is launched. `lib_dir`, `lef_dir`, `macro_lib_dir` and `macro_lef_dir` locate the technology and macro data.
2. **Enumerate 16 corners:** `set libs {...}` lists FF, SS and TT characterized libraries; `file rootname` converts filenames into unique corner names.
3. **Declare corners:** `define_corners {*}$corners` creates the analysis-corner objects.
4. **Read physical abstracts:** technology and merged LEFs are loaded, followed by available `avsddac` and `avsdpll` macro LEFs.
5. **Read standard-cell timing:** `read_liberty -corner $corner` associates each SKY130HD timing library with its corresponding corner.
6. **Handle macro timing libraries:** the Tcl code rewrites the top-level `library(...)` identifier of each macro Liberty file to a corner-specific name in a temporary file, loads that file, then deletes it. This avoids library-name collisions; it does **not** recharacterize the macro for each PVT corner.
7. **Read netlist and link:** `read_verilog .../1_synth.v` loads the mapped netlist; `link_design vsdbabysoc` resolves the top-level hierarchy and cells.
8. **Read constraints:** `read_sdc .../vsdbabysoc_synthesis.sdc` applies the 12 ns clock and any other submitted constraints.
9. **Print reports:** the loop invokes `report_checks -corner $corner`, `report_worst_slack -max` and `report_tns -max` for every library.
10. **Exit after reports:** run from the flow root with `openroad -exit designs/sky130hd/vsdbabysoc/run.tcl`.

### WNS/TNS reporting caveat
The assignment displays **global WNS = −17.16 ns** and **global TNS = −5357.16 ns** repeatedly. The script applies `-corner $corner` only to `report_checks`, **not** to `report_worst_slack -max` or `report_tns -max`. Consequently those repeated WNS/TNS values must **not** be presented as independent per-corner totals. A selected path with positive slack does not prove that every endpoint in that corner passes. Per-corner WNS/TNS would require explicitly corner-scoped reporting and validation.

### Why low-voltage SS corners fail
The worst sampled SS/−40 °C path improves strongly as voltage increases:
| Supply (V) | Arrival (ns) | Slack (ns) | Status |
|---:|---:|---:|---|
| 1.28 | 27.48 | -17.16 | VIOLATED |
| 1.35 | 18.81 | -7.89 | VIOLATED |
| 1.40 | 15.11 | -3.95 | VIOLATED |
| 1.44 | 12.95 | -1.65 | VIOLATED |
| 1.60 | 8.08 | +3.53 | MET |
| 1.76 | 5.78 | +5.97 | MET |

Between **1.28 V and 1.76 V**, the arrival time falls by **21.70 ns** and slack improves by **23.13 ns**, crossing from violation to positive margin at the sampled **1.60 V** point. This is an observation about the reported paths, not a guaranteed voltage threshold for all endpoints or a recommendation to change the physical supply voltage outside technology limits.

---
## Comparison and interpretation
| Aspect | Assignment 1 | Assignment 2 |
|---|---|---|
| Engine/environment | OpenSTA terminal | OpenROAD/OpenSTA terminal |
| Clock period | 11 ns | 12 ns |
| Nominal frequency | 90.91 MHz | 83.33 MHz |
| Documented corners | 15 | 16 |
| Reported paths meeting timing | 7 | 12 |
| Reported paths violating timing | 8 | 4 |
| Most negative documented path slack | −62.56 ns | −17.16 ns |
| Largest documented positive path slack | +5.57 ns | +9.88 ns |
| Netlist | `vsdbabysoc.synth.v` | `1_synth.v` |
| Corner method | Sequential library reloads | Explicit `define_corners` and per-corner Liberty loading |
| WNS/TNS | Concepts discussed; no trustworthy per-corner totals documented | Global WNS/TNS printed repeatedly, not per-corner values |
| Physical parasitics | Not demonstrated | Not loaded by shown Tcl |

**Do not attribute the difference in results solely to the 1 ns clock-period change.** The assignments use different netlist paths, timing environments and corner-loading methods, and their reports identify different critical endpoints. To make a controlled comparison, run the same netlist, same cell/macro models, same corner, same constraints and same parasitic assumptions with only one variable changed.

## Limitations and future work
1. **Path-specific results:** the per-corner tables summarize the single `report_checks` path shown at each corner, not all constrained endpoints.
2. **No extracted interconnect parasitics demonstrated:** routed RC can materially alter delay, slew and slack.
3. **Ideal clock modeling:** real clock-tree insertion delay, skew and uncertainty require careful modeling.
4. **Setup is not hold:** `Path Type: max` tests setup; minimum-delay/hold checks must be reported separately.
5. **Macro characterization:** renaming a macro Liberty library does not generate true FF/SS/TT-specific DAC/PLL models.
6. **Repeated library loads in Assignment 1:** use separate STA sessions or properly isolated corners to prevent stale state and duplicate-name warnings.
7. **Constraint completeness:** review generated clocks, PLL output clocks, clock groups, I/O delays, false paths, multicycle paths and unconstrained endpoints.
8. **WNS/TNS scoping:** request per-corner values explicitly rather than repeating global metrics under every heading.
9. **Signoff:** run post-route STA with extracted parasitics, setup/hold analysis, and physical verification before claiming timing closure or tapeout readiness.

## What I learned
These assignments helped me move from simply reading a positive or negative slack number to understanding **why timing changes across PVT conditions**, **how setup margins are computed**, **how Liberty models and SDC constraints affect results**, and **why a passing critical-path sample is not equivalent to complete design closure**. Automating multi-corner analysis with Tcl also made it possible to compare many operating conditions systematically and identify the slow, low-voltage corners most deserving of optimization attention.

## References
- `AssessmentSTAA06Updated.pdf` — **OpenSTA Library Output Analysis**, 21 pages; source for Assignment 1 values, corner narratives, WNS/TNS explanation and original Tcl loop.
- `VSDBabySoC_Multi_Corner_STA_Report_A06.pdf` — **VSDBabySoC Multi-Corner STA Analysis**, 21 pages; source for Assignment 2 Tcl script, screenshots and 16-corner results.
- SkyWater SKY130HD Liberty library files named in the tables.
- OpenSTA/OpenROAD timing commands used in the submitted scripts.

> **Documentation note:** Numerical values are transcribed from the two submitted PDF reports, not recomputed by running the design. Any ambiguity in the original report is identified rather than silently resolved.