# DAY 10 — Physical Design: Routing and Final Implementation

## Overview

Day 10 focuses on **routing**, the stage where the placed and clocked design is converted into an actual physical interconnect structure.

The main topics covered are:

- Routing concepts and routing resources
- Global and detailed routing
- RC delay in routed wires
- Maze routing and Lee's algorithm
- FastRoute in OpenLane
- TritonRoute for detailed routing
- Final physical implementation of PicoRV32a

---

## Routing Basics

Routing connects the pins of standard cells, clock structures, and other design elements using metal layers and vias.

Routing must consider:

- Available routing tracks
- Metal layers and their preferred directions
- Wire length
- Resistance and capacitance
- Congestion
- Design-rule restrictions
- Connections between different metal layers

Routing is generally divided into **global routing** and **detailed routing**.

### Types of Routing

| Routing Type | Purpose |
|---|---|
| **Global Routing** | Finds approximate paths through coarse routing regions called GCells. |
| **Detailed Routing** | Converts those paths into exact wires and vias on real tracks and metal layers while obeying design rules. |
| **Pattern Routing** | Uses predefined simple patterns such as L-shaped or Z-shaped paths for easy nets. |
| **Monotonic Routing** | Routes in a consistent direction without backtracking, making it fast for less-congested regions. |
| **Maze Routing** | Searches through a grid to find a valid path around obstacles. |
| **RSMT-based Routing** | Builds a tree connecting multiple pins while attempting to keep total wirelength small. |

Global routing considers the overall routing problem at a coarse level, while detailed routing performs the exact physical implementation.

---

## TritonRoute in OpenLane

**TritonRoute** is the detailed router used in the OpenLane flow.

It receives the routing information generated during global routing and converts it into actual physical geometry:

- Metal wire segments
- Routing tracks
- Vias
- Layer changes
- DRC-compliant connections

The basic idea is:

```text
Placement
   ↓
Global Routing
   ↓
TritonRoute
   ↓
Detailed Routing
   ↓
Final Routed Design
```

TritonRoute does not solve the entire routing problem blindly. It uses the global routing solution to narrow down the possible routing locations and then performs detailed routing and repair.

### TritonRoute Stages

1. **Track Assignment**  
   Nets are assigned to specific routing tracks within the regions selected during global routing.

2. **Initial Detailed Routing**  
   Actual wire segments and vias are created on the assigned tracks.

3. **Search and Repair**  
   Routing problems and DRC violations are identified and the affected portions are repaired.

This staged approach makes routing practical for large physical designs.

> **Image Placeholder — TritonRoute / Detailed Routing**

`[Insert TritonRoute or detailed-routing image here]`

---

## RC Delay in Routing

A routed wire is not an ideal connection. It has electrical resistance and capacitance.

The wire can therefore be represented using an **RC network**.

The simplified **Elmore delay model** estimates propagation delay using the resistance and capacitance accumulated along the path.

Important points:

- Longer wires generally have greater resistance and capacitance.
- Larger loads increase capacitance.
- Higher RC results in greater delay.
- Actual routed geometry is required for accurate post-route RC extraction.

Therefore, routing affects timing directly.

> **Image Placeholder — RC Delay Model**

`[Insert RC delay image here]`

---

## Global Routing

Global routing determines approximate paths for the nets before exact wires are created.

The routing area is divided into coarse regions called **GCells**.

Global routing determines:

- Which regions a net should pass through
- Approximate routing paths
- Routing demand
- Congestion across the design

It does not yet assign every wire to an exact physical track.

### FastRoute

**FastRoute** performs global routing in the OpenLane flow.

It aims to create routing paths while considering:

- Wirelength
- Congestion
- Available routing resources
- Multi-pin connections

FastRoute combines several routing techniques:

- **Pattern routing** for simple nets
- **Monotonic routing** for suitable paths
- **Selective maze routing** for difficult or congested nets
- **RSMT construction** for multi-pin nets

The result is a global routing solution that guides the detailed-routing stage.

> **Image Placeholder — Global Routing / FastRoute**

`[Insert FastRoute or global-routing image here]`

---

## Maze Routing — Lee's Algorithm

Maze routing treats the routing area as a grid and searches for a path between a source and destination while avoiding blocked locations.

### Basic Steps

1. Start from the source.
2. Expand through reachable grid locations.
3. Continue until the destination is reached.
4. Trace backward from the destination to obtain the route.

Lee's algorithm can find a shortest path when a valid path exists, but performing a full maze search for every net can be computationally expensive.

Therefore, practical routers use maze routing selectively along with faster routing techniques.

> **Image Placeholder — Maze Routing**

`[Insert maze-routing image here]`

---

## Global Routing vs Detailed Routing

| Feature | Global Routing | Detailed Routing |
|---|---|---|
| Routing level | Coarse | Exact |
| Main unit | GCells | Tracks, wires and vias |
| Main objective | Find feasible paths | Create legal physical geometry |
| DRC awareness | Limited/coarse | Detailed |
| Output | Routing guides | Final routed wires |

The two stages work together: global routing reduces the search space, while detailed routing performs the final physical implementation.

---

## Final Physical Implementation

After floorplanning, power planning, placement, CTS, and routing, the design reaches a complete physical implementation.

At this stage, the design contains:

- Placed standard cells
- Power distribution
- Clock tree
- Routed signal nets
- Metal interconnect
- Vias
- Physical routing geometry

The completed design can then proceed toward:

- RC extraction
- Post-route STA
- DRC
- LVS
- Final GDSII generation

### Final Design Views

> **Image Placeholder — Final Synthesized Netlist**

`[Insert final synthesis image here]`

> **Image Placeholder — Final Floorplan**

`[Insert final floorplan image here]`

> **Image Placeholder — Final Placement**

`[Insert final placement image here]`

> **Image Placeholder — Final Routed Design**

`[Insert final routed-design image here]`

---

## Day 10 Key Learnings

- Understood the purpose of physical routing.
- Studied global and detailed routing.
- Understood routing tracks, wires and vias.
- Studied RC delay in routed wires.
- Understood Elmore delay modeling.
- Studied pattern and monotonic routing.
- Studied maze routing and Lee's algorithm.
- Understood FastRoute as the global router.
- Understood TritonRoute as the detailed router.
- Studied track assignment and search-and-repair.
- Understood the relationship between routing and timing.
- Reviewed the final physical implementation of PicoRV32a.

---

## Conclusion

Day 10 completes the routing stage of the physical-design flow.

The main idea is that **global routing decides where nets should go, while detailed routing converts those decisions into exact physical wires and vias**.

Routing is also closely connected to timing because wire resistance, capacitance, length, and loading affect RC delay. The final routed design therefore provides the physical information needed for post-route timing analysis and physical verification.
