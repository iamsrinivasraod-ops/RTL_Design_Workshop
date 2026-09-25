# Day 8 – Design Cell Library Using Magic Layout and ngspice Characterization

## 1. 16-Mask CMOS Fabrication Process

A CMOS fabrication process creates NMOS and PMOS transistors on a silicon wafer through repeated masking, doping, deposition, etching and annealing steps. A simplified 16-mask flow is:

1. **Selecting the substrate** – A silicon wafer is selected as the starting material. The substrate type and doping are chosen according to the CMOS process.
2. **Well doping / well formation** – An n-well is formed for PMOS devices (in a p-type substrate) using masking and ion implantation/diffusion.
3. **Creating active regions** – Isolation structures are formed and active areas are defined where the source, drain and transistor channels will be fabricated.
4. **Gate oxide formation** – A thin, high-quality oxide is formed over the active silicon.
5. **Polysilicon gate formation** – Polysilicon is deposited and patterned to form the transistor gates.
6. **LDD formation** – Lightly Doped Drain regions are implanted beside the gate. LDD reduces the high electric field near the drain.
7. **Sidewall spacers** – An insulating layer is deposited and etched anisotropically, leaving spacers on the sides of the gate.
8. **Source and drain formation** – Heavily doped source/drain regions are formed. **Arsenic** is commonly used as an n-type dopant for NMOS source/drain implantation.
9. **Annealing** – Heat treatment activates the dopants and repairs implantation damage.
10. **Silicide / titanium processing** – Titanium can be deposited and reacted with silicon to form low-resistance silicide on selected regions.
11. **Contact formation** – Contact holes are created through the dielectric to connect the transistor regions to metal.
12. **Metal deposition** – Metal is deposited and patterned to create electrical interconnections.
13. **Higher metal layers** – Additional dielectric, vias and metal layers are formed for routing.
14. **Passivation and final processing** – A protective layer is added and openings are made for the required external connections.

### Important fabrication concepts

- **LDD:** Lightly Doped Drain reduces the electric field near the drain and helps reduce hot-carrier damage.
- **Hot-electron effect:** In a short-channel MOSFET, carriers near the drain can gain high energy because of the strong electric field. These energetic carriers can enter the oxide or create interface damage, changing transistor characteristics.
- **Short-channel effect:** As channel length becomes small, the gate has less control over the channel. Effects such as threshold-voltage reduction and drain-induced barrier lowering become important.
- **Sidewall spacers:** These are formed beside the gate after LDD implantation. They allow the final heavily doped source/drain implant to be placed farther from the channel, creating the LDD structure.
- **Arsenic:** A heavily used n-type dopant for NMOS source/drain formation because it provides high concentration with relatively low diffusion during processing.
- **Titanium:** Used in semiconductor processing for contacts/silicide and related low-resistance connections.
- **Sputtering:** A physical vapour deposition method in which energetic ions strike a target and eject atoms that deposit on the wafer.
- **Masking:** Each mask defines where a particular process such as implantation, etching or deposition is allowed to occur.

## 2. LEF Technology

**LEF = Library Exchange Format.**

LEF is an abstract physical description used by place-and-route tools. It provides information such as:

- Standard-cell dimensions
- Pin names and locations
- Metal and routing layers
- Via information
- Routing directions
- Obstructions and physical constraints

LEF does not contain the complete detailed transistor layout. The detailed geometry is normally represented by formats such as GDS.

---

# 3. Changing Pin Spacing in OpenLane for PicoRV32A

For the PicoRV32A design, pin placement can be changed through the OpenLane configuration.

The command used in this setup was:

```tcl
set ::env(FP_IO_MODE) 1
```

To change the pin placement mode:

```tcl
set ::env(FP_IO_MODE) 2
```

`FP_IO_MODE` controls the I/O pin placement mode used during floorplanning. Changing the mode changes how the pins are distributed around the core, which can change the effective distance/spacing between pins.

Other floorplanning parameters such as die/core dimensions and utilization also affect the physical distances available for routing.

The basic idea is:

```text
Larger available perimeter/core area
        ↓
More space for I/O distribution
        ↓
Pin spacing and routing distances can change
```

---

# 4. W/L Ratio of PMOS and NMOS

The transistor strength is strongly related to its **W/L ratio**:

\[
\frac{W}{L}
\]

where:

- **W** = transistor channel width
- **L** = transistor channel length

A larger W/L generally means a stronger transistor because it can carry more current.

For the extracted inverter used here:

```text
NMOS: W = 35, L = 23
PMOS: W = 37, L = 23
```

Therefore:

\[
(W/L)_n = 35/23 \approx 1.52
\]

\[
(W/L)_p = 37/23 \approx 1.61
\]

The PMOS is slightly wider because hole mobility is lower than electron mobility. Making the PMOS wider helps compensate for its lower mobility and makes the pull-up and pull-down strengths more comparable.

### Effect on switching

The switching point of a CMOS inverter depends on the relative strengths of PMOS and NMOS.

- **Increase NMOS W/L:** NMOS becomes stronger → switching point tends to move lower.
- **Increase PMOS W/L:** PMOS becomes stronger → switching point tends to move higher.
- **Balanced strengths:** Gives a more symmetric inverter transition and more similar rise/fall behavior.

The basic current-strength relationship is:

\[
\beta = \mu C_{ox}\frac{W}{L}
\]

Since electron mobility is normally higher than hole mobility, PMOS is often designed with a larger W/L than NMOS.

---

# 5. CMOS Inverter Layers in Magic

A CMOS inverter contains one PMOS and one NMOS.

### Important layers to identify

- **n-well** – contains the PMOS.
- **p-diffusion** – PMOS source/drain regions.
- **n-diffusion** – NMOS source/drain regions.
- **Polysilicon** – forms the gates.
- **Metal layers** – provide interconnections.
- **Contacts / vias** – connect diffusion or polysilicon to metal and connect different metal layers.

The polysilicon crossing a diffusion region forms the transistor channel/gate.

Basic inverter connection:

```text
             VDD
              |
            PMOS
              |
Input ────────|──── Output
              |
            NMOS
              |
             GND
```

When input = 0, PMOS is ON and NMOS is OFF, so output is HIGH.

When input = 1, PMOS is OFF and NMOS is ON, so output is LOW.

---

# 6. Opening the PicoRV32A Layout in Magic

Magic was opened using the SKY130 technology file:

```bash
magic -T /home/vsduser/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech
```

The LEF and DEF were then read:

```tcl
lef read ../../tmp/merged.lef
def read picorv32a.floorplan.def
```

The LEF provides the abstract cell/library information, while the DEF contains the physical design information such as placement and routing.

---

# 7. Opening Metal 3 in Magic

In SKY130, the third routing metal layer is:

```text
met3
```

Metal layers are used mainly for routing/interconnection between different cells and circuit regions. In Magic, the layer can be selected using the layer-selection commands or the appropriate layer information shown by the Magic technology file.

---

# 8. CMOS Inverter Layout and SPICE Extraction

For the standard-cell design work, the inverter layout was opened using:

```bash
magic -T sky130.tech sky130_inv.mag &
```

Inside the Magic window, extraction was performed using:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

This creates the extracted SPICE representation of the layout.

The generated SPICE file was then edited for ngspice simulation.

---

# 9. SPICE File Used for Characterization

The extracted inverter was represented approximately as:

```spice
* SPICE3 file created from sky130_inv.ext - technology: sky130A

.option scale=0.01u
.include ./libs/nshort.lib
.include ./libs/pshort.lib

M0 Y A VGND VGND nshort_model.0 ad=1.44n pd=0.152m as=1.37n ps=0.148m w=35 l=23
M1 Y A VPWR VPWR pshort_model.0 ad=1.44n pd=0.152m as=1.52n ps=0.156m w=37 l=23

VDD VPWR 0 3.3V
VSS VGND 0 0V

Va A VGND PULSE(0V 3.3V 0 0.1ns 0.1ns 2ns 4ns)

C0 VPWR A 0.0774f
C1 Y A 0.0754f
C2 Y VPWR 0.117f
C3 Y VGND 0.279f
C4 A VGND 0.45f
C5 VPWR VGND 0.781f

.tran 1n 20n

.control
run
.endc

.end
```

The inverter was simulated with:

```bash
ngspice sky130_inv.spice
```

---

# 10. CMOS Inverter Output Waveform

The input is a pulse waveform from 0 V to 3.3 V.

Because the circuit is an inverter:

```text
Input:    LOW ─── HIGH ─── LOW

Output:   HIGH ─── LOW ─── HIGH
```

The output does not switch instantaneously because the transistor resistance and parasitic/load capacitances produce a finite charging and discharging time.

---

# 11. Rise Time

Rise time is the time required for the output to rise from a lower percentage to a higher percentage of its final voltage.

For this characterization, **20% to 80%** was used.

For a 3.3 V supply:

\[
20\% = 0.66V
\]

\[
80\% = 2.64V
\]

From the ngspice cursor measurements:

```text
At approximately 20%:
x0 = 2.16845e-09 s
y0 = 0.649918 V

At approximately 80%:
x0 = 2.20466e-09 s
y0 = 2.645 V
```

Therefore:

\[
t_r = 2.20466ns - 2.16845ns
\]

\[
\boxed{t_r \approx 0.03621ns = 36.21ps}
\]

---

# 12. Propagation Delay

Propagation delay is measured between the 50% voltage points of the input and output.

For a 3.3 V supply:

\[
50\% = 1.65V
\]

The measured points were:

```text
Point 1:
x0 = 2.1866e-09 s
y0 = 1.65 V

Point 2:
x0 = 2.14997e-09 s
y0 = 1.65 V
```

The propagation delay is the absolute difference between the two times:

\[
t_p = |2.1866ns - 2.14997ns|
\]

\[
\boxed{t_p \approx 0.03663ns = 36.63ps}
\]

The exact delay can vary slightly depending on which input/output transition is selected and the cursor precision.

---

# 13. Important Viva Points

- **Magic** is used for layout editing, visualization and extraction.
- **LEF** contains abstract physical library information.
- **DEF** contains physical design information such as placement and routing.
- **SPICE** represents the circuit for electrical simulation.
- **ngspice** is used to simulate the extracted/edited SPICE circuit.
- **W/L** determines transistor strength and affects switching behavior.
- PMOS is usually made wider because hole mobility is lower than electron mobility.
- **LDD** reduces the drain electric field and helps control hot-carrier effects.
- **Sidewall spacers** separate the LDD region from the heavily doped source/drain region.
- **Rise time** describes how quickly the output rises between specified voltage levels.
- **Propagation delay** measures the time shift between corresponding input and output threshold crossings.
- In this experiment, the rise-time measurement used **20%–80%**, while propagation delay used the **50% voltage point**.
- **Metal 3 (`met3`)** is a routing layer in the SKY130 technology.
