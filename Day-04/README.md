# Day 4 -- GLS and Synthesis Mismatch

## 1. Overview

Day 4 of the workshop focused on **Gate Level Simulation (GLS)**,
**synthesis mismatch**, and the importance of choosing the correct
Verilog assignment statements and coding style.

The main concepts covered were:

-   Ternary operator based Multiplexer
-   RTL simulation
-   Gate Level Simulation (GLS)
-   Difference between RTL and gate-level models
-   Blocking (`=`) and non-blocking (`<=`) assignments
-   `always @(*)` blocks
-   Synthesis mismatch and coding caveats
-   A blocking assignment caveat example

The objective was to understand how RTL code is converted into hardware
during synthesis and how GLS can be used to verify the synthesized
design.

------------------------------------------------------------------------

## 2. Ternary Operator MUX

A simple 2:1 Multiplexer can be written using the Verilog ternary
operator.

### RTL Code

``` verilog
module ternary_operator_mux(input i0 , input i1 , input sel , output y);

assign y = sel?i1:i0;

endmodule
```

### Working

The ternary operator follows the form:

``` text
condition ? value_if_true : value_if_false
```

For the MUX:

  `sel`   Output `y`
  ------- ------------
  0       `i0`
  1       `i1`

Therefore, when `sel = 0`, input `i0` is selected, and when `sel = 1`,
input `i1` is selected.

This is a simple example of combinational RTL that can be synthesized
into gate-level hardware.

------------------------------------------------------------------------

## 3. RTL Simulation

**RTL simulation** verifies the behavior of the design before synthesis.

The RTL design is simulated using a testbench, and the output waveforms
are observed to check whether the design behaves as expected.

### Basic RTL Simulation Flow

  Step   Description
  ------ -----------------------------------
  1      Write the RTL design
  2      Create a testbench
  3      Compile the RTL and testbench
  4      Run the simulation
  5      Observe signals and waveforms
  6      Verify the expected functionality

The important point is that RTL simulation verifies the **intended
functional behavior** of the HDL description.

------------------------------------------------------------------------

# 4. What is GLS?

**GLS (Gate Level Simulation)** means running a testbench with the
**synthesized gate-level netlist** as the Design Under Test instead of
the original RTL.

The synthesized netlist is logically equivalent to the RTL design when
synthesis is successful and the RTL is written correctly.

### GLS Flow

``` text
          RTL Design
              |
              v
          Synthesis
              |
              v
     Gate-Level Netlist
              |
              v
        GLS Simulation
              |
              v
       Waveform / VCD
              |
              v
          GTKWave
```

The same testbench can generally be used because the gate-level netlist
represents the same logical functionality as the RTL design.

### Why GLS?

  -----------------------------------------------------------------------
  Purpose                             Explanation
  ----------------------------------- -----------------------------------
  Functional verification             Checks whether the synthesized
                                      netlist still behaves correctly

  Detect synthesis issues             Helps identify differences between
                                      RTL intent and synthesized hardware

  Timing validation                   With appropriate delay annotation,
                                      GLS can be used for timing
                                      verification

  Post-synthesis confidence           Provides additional confidence that
                                      synthesis produced the intended
                                      hardware
  -----------------------------------------------------------------------

If the gate-level model is **delay annotated**, GLS can also be used for
timing validation.

------------------------------------------------------------------------

## 5. GLS Using IVerilog and GTKWave

The workshop demonstrated a basic GLS flow using **Icarus Verilog
(iverilog)** and **GTKWave**.

The main inputs are:

-   RTL/design description
-   Gate-level Verilog model/netlist
-   Testbench

These are supplied to `iverilog` for simulation.

The simulation produces a **VCD (Value Change Dump)** file containing
signal value changes. The VCD file can then be opened in GTKWave to
inspect the waveforms.

### Flow

  Component                  Role
  -------------------------- -----------------------------------------------------
  Design                     Original RTL design
  Gate-level Verilog model   Synthesized representation of the design
  Testbench                  Provides input stimulus and checks/observes outputs
  `iverilog`                 Compiles and runs the Verilog simulation
  VCD file                   Stores signal transitions
  GTKWave                    Displays the simulation waveforms

### Important Note

If the gate-level model contains delay information, GLS can be used for
**timing validation** in addition to functional verification.

------------------------------------------------------------------------

# 6. Blocking and Non-Blocking Assignments

One of the important topics covered was the difference between
**blocking** and **non-blocking** assignments.

## Blocking Assignment

The blocking assignment operator is:

``` verilog
=
```

It is normally used for combinational logic inside an `always` block.

A blocking statement executes in the order in which it is written.
Therefore, the next statement sees the updated value.

### Example

``` verilog
always @(*) begin
    a = b & c;
    e = a & f;
end
```

Here, `a` is assigned first. The following statement uses the updated
value of `a`.

------------------------------------------------------------------------

## Non-Blocking Assignment

The non-blocking assignment operator is:

``` verilog
<=
```

It schedules the update of the left-hand side and is normally used for
sequential logic, especially clocked `always` blocks.

### Example

``` verilog
always @(posedge clk) begin
    q <= d;
end
```

Non-blocking assignments are important for correctly modeling flip-flop
behavior.

------------------------------------------------------------------------

## Blocking vs Non-Blocking

  Feature             Blocking `=`           Non-Blocking `<=`
  ------------------- ---------------------- ---------------------------------------
  Typical use         Combinational logic    Sequential logic
  Execution           Immediate/sequential   Update scheduled
  Order matters       Yes                    Assignments can be evaluated together
  Common block        `always @(*)`          `always @(posedge clk)`
  Hardware modeling   Combinational logic    Flip-flops/registers

A good general rule is:

> **Use blocking assignments for combinational logic and non-blocking
> assignments for clocked sequential logic.**

------------------------------------------------------------------------

# 7. Importance of `always @(*)`

The statement:

``` verilog
always @(*)
```

is used to describe combinational logic.

The `*` automatically includes the signals read inside the block in the
sensitivity list.

### Example

``` verilog
module good_mux (input i0 , input i1 , input sel , output reg y);

always @(*) begin
    if(sel)
        y <= i1;
    else
        y <= i0;
end

endmodule
```

The `always @(*)` block is triggered whenever an input used by the
combinational logic changes.

### Why use `always @(*)`?

  -----------------------------------------------------------------------
  Advantage                           Explanation
  ----------------------------------- -----------------------------------
  Automatic sensitivity               Signals read inside the block are
                                      included automatically

  Less error-prone                    Avoids manually forgetting an input

  Suitable for combinational logic    Clearly indicates that the block
                                      models combinational behavior

  Better readability                  Makes the designer's intent easier
                                      to understand
  -----------------------------------------------------------------------

For combinational logic, `always @(*)` is preferred over manually
writing a sensitivity list.

------------------------------------------------------------------------

# 8. Synthesis Mismatch

A **synthesis mismatch** occurs when the behavior expected from RTL
simulation does not match the behavior of the synthesized hardware or
gate-level simulation.

This can happen because HDL is both a programming language and a
hardware description language. The way statements are written affects
the hardware inferred by synthesis.

### Common Causes

  -----------------------------------------------------------------------
  Cause                               Possible Problem
  ----------------------------------- -----------------------------------
  Incorrect blocking/non-blocking     Simulation behavior may not
  usage                               represent intended hardware

  Incomplete combinational            Can result in unintended latches
  assignments                         

  Incorrect sensitivity list          RTL simulation may not update when
                                      expected

  Race conditions                     Simulation ordering may produce
                                      unexpected results

  Ambiguous coding style              Synthesis may infer hardware
                                      differently from the designer's
                                      intention
  -----------------------------------------------------------------------

Therefore, correct RTL coding style is important for obtaining
predictable synthesized hardware.

------------------------------------------------------------------------

# 9. Blocking Assignment Caveat

The following example demonstrates an important caveat with blocking
assignments.

### Code

``` verilog
module blocking_caveat (input a , input b , input c , output reg d);
reg x;
always @(*)
begin
    d = x & c;
    x = a | b;
end
endmodule
```

### What happens?

The statements execute sequentially:

``` text
1. d = x & c;
2. x = a | b;
```

The value of `d` is calculated using the **old value of `x`**, because
`x` is assigned only after the first statement.

The intended combinational relationship may be:

``` text
x = a | b
d = x & c
```

which mathematically means:

``` text
d = (a | b) & c
```

However, because of the order of blocking assignments in the example,
`d` does not immediately use the newly calculated value of `x` within
that procedural execution.

### Why is this a caveat?

This example shows that **statement ordering matters when blocking
assignments are used**.

For combinational logic, the safest practice is to write assignments in
a clear dependency order.

For example:

``` verilog
always @(*) begin
    x = a | b;
    d = x & c;
end
```

Now `x` is calculated before it is used to calculate `d`.

### Key Point

> With blocking assignments, the order of statements can affect the
> simulation behavior.

This is one reason why careful RTL coding is essential for avoiding
simulation/synthesis mismatches and unexpected results.

------------------------------------------------------------------------

# 10. RTL Simulation vs GLS

  -----------------------------------------------------------------------
  Aspect                  RTL Simulation          Gate Level Simulation
  ----------------------- ----------------------- -----------------------
  Design used             RTL code                Synthesized gate-level
                                                  netlist

  Stage                   Before synthesis        After synthesis

  Main purpose            Verify RTL              Verify synthesized
                          functionality           implementation

  Testbench               RTL testbench           Usually the same
                                                  testbench

  Timing                  Usually abstract/ideal  Can include delays

  Output                  RTL waveforms           Gate-level waveforms

  Tool example            IVerilog                IVerilog

  Waveform viewer         GTKWave                 GTKWave
  -----------------------------------------------------------------------

GLS is therefore an important step between RTL verification and
implementation-level confidence.

------------------------------------------------------------------------

# 11. Workshop Demonstration Summary

The Day 4 demonstration followed a simple progression:

``` text
Ternary MUX RTL
      |
      v
RTL Simulation
      |
      v
Synthesis
      |
      v
Gate-Level Netlist
      |
      v
GLS
      |
      v
VCD
      |
      v
GTKWave
```

Along with this flow, the workshop examined how **blocking**,
**non-blocking**, and `always @(*)` statements influence RTL behavior
and synthesis.

The blocking caveat example demonstrated that procedural statement
ordering must be considered carefully when describing combinational
hardware.

------------------------------------------------------------------------

# 12. Key Takeaways

1.  **GLS stands for Gate Level Simulation** and is performed using the
    synthesized gate-level netlist.

2.  RTL simulation verifies the intended functionality before synthesis,
    while GLS verifies the synthesized implementation.

3.  A simple MUX can be described using the **ternary operator**:

    ``` verilog
    assign y = sel ? i1 : i0;
    ```

4.  **Blocking (`=`)** assignments execute sequentially within a
    procedural block, so statement order matters.

5.  **Non-blocking (`<=`)** assignments are generally used for
    sequential/clocked logic.

6.  `always @(*)` is commonly used for combinational logic because it
    automatically handles the sensitivity list.

7.  Incorrect coding style can lead to **simulation/synthesis
    mismatches** or unintended hardware.

8.  The blocking caveat example shows why dependent combinational
    assignments should be written in a clear and logical order.

9.  GLS can provide additional confidence that the synthesized netlist
    preserves the intended RTL functionality.

10. With delay annotation, GLS can also be used for **timing
    validation**.
