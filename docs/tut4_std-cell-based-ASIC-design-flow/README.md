# Tutorial IV: Standard Cell Based ASIC Design Flow

## Overview:
This tutorial introduces you to the standard cell based ASIC design flow using tools and libraries from various vendors. We will use the Nangate 45nm standard-cell library with the FreePDK45 to implement an 8-bit accumulator design. We will first synthesize the design using the Synopsys Design Compiler and then perform place and route using the Cadence Innovus. The final layout will be painted in the Cadence Virtuoso platform and the final design will be verified by the Synopsys Formality Equivalence Checker.

Note: Since network failure may interrupt your operation, please save your data often.

## Table of Contents:
1. [RTL Simulation](#1.-RTL-Simulation)
2. Logic Synthesis using Synopsys Design Compiler
3. Place and Route using Cadence Innovus
4. Layout Printing using Cadence Virtuoso

Create and initialize a directory for lab 9
```
source /import/scripts/ece429.cshrc
source /import/scripts/hspice.cshrc
source /import/scripts/synopsys2012.cshrc
mkdir accu
cd accu
ece429-init-dir
```
---

## **1. RTL Simulation**

#### **Step 1: RTL Behavioral Description and Verification by Description**
In this step, you enter a code in Verilog on the Register-Transfer-Level (RTL) based on the design requirments, where you model your design using clocked registers, datapath elements and control elements. For the verification, you will use Cadence Verilog-XL to simulate your design via the testbench code.
In this tutorial there are 2 files as follows:

1. **accu.v:** Verilog RTL code for an 8-bit accumulator

```
module dff(d, q, clk);
        output  q;
        input   d;
        input   clk;
        reg     q;

always @(posedge clk)
        q <= d;

endmodule

module accu(in, accu, clk, rst);
        output  [7:0]   accu;
        input   [7:0]   in;
        input           clk;
        input           rst;

        wire    [7:0]   accu;
        wire    [7:0]   dff_in;

dff r0(dff_in[0], accu[0], clk);
dff r1(dff_in[1], accu[1], clk);
dff r2(dff_in[2], accu[2], clk);
dff r3(dff_in[3], accu[3], clk);
dff r4(dff_in[4], accu[4], clk);
dff r5(dff_in[5], accu[5], clk);
dff r6(dff_in[6], accu[6], clk);
dff r7(dff_in[7], accu[7], clk);

assign dff_in = rst? 8'b0: accu+in;

endmodule
```

2. **tb_accu.v:** Verilog testbench for accu.v

```
module tb_accu;

reg     [7:0]   in;
reg             clk;
reg             rst;
wire    [7:0]   accu;

accu dut(.in(in),
         .accu(accu),
         .clk(clk),
         .rst(rst)
);

initial
 begin
        $shm_open("shm.db", 1);
        $shm_probe("AS");
        #50 $finish;
        #100 $shm_close();
 end

initial
 begin
  clk = 1'b0;
  forever
   begin
   #5 clk = ~clk;
   $display("At time: %d The Accumulator Output=%d",$time, accu);
   end
 end

initial
 begin
        #0 rst <= 1;
           in  <= 1;
        #5 rst <= 0;
 end

endmodule
```

#### **Step 2:**
In order to simulate Verilog code via Cadence Verilog-XL, use this command:
```
xrun tb_accu.v accu.v +access+r
```
This testbench provides results directly on the screen and also in a waveform database. From the screen we can see that the design behaves as expected as follows.

![1](./fig/rtl_sim.png)

That is, every 10ns we add 1 to the accumulator. This is expected since in the testbench a clock of 10ns is specified and the input 'in' is connected to a constant 1.

#### **Step 3:**
We use the program Cadence SimVision to look at the waveform database that was created by Verilog-XL. Type the following command:
```
simvision
```
![2](./fig/simvision_1.png)

Now we need to open the Waveform database. 
1. Click on "File" > "Open Database..." or "ctl+o".
2. Choose the directory "shm.db", which is where the file is located, and double-click on the file "shm.trn" to open it.

![3](./fig/simvision_2.png)

3. To see the contents of the waveform database, from the "Design Browser" menu on the left, click on tb_accu, and the four waveforms (accu[7:0], clk, in[7:0], rst) appears on the show contents, click on each one of them to add the waveforms.

![4](./fig/simvision_3.png)

---

### **2. Logic Synthesis using Synopsys Design Compiler**
Once you have verified that your Verilog RTL code is working correctly you can synthesize it into standard cells. The result will be a gate-level netlist that only contains interconnected standard cells.

**Step 1: Copy Template Files**
There are template files for all the following steps already prepared for you. We will now copy those templates into our project.
```
cp /import/scripts/Nangate45nm/* .
```
(Do not forget to add a star-space-dot(* .) after the file path.)

We will use the Synopsys Design Compiler for logic synthesis. Since a hardware design requires not only the Verilog descriptions but also the specifications, we will use a script file to automate the synthesis task. The template file is provided as 'compile_dc.tcl'. Note that dc stands for Design Compiler (DC).

**Step 2: Script File**
Please open 'compile_dc.tcl' in a text editor. Although you don't need to modify this file for this tutorial, you will need to modify it for the final project so please read the description of the file below carefully. To make it easier to modify the file, all key values are defined in the beginning of the file.

```
#/**************************************************/
#/* Compile Script for Synopsys Design Compiler    */
#/*                                                */
#/* dc_shell -f compile_dc.tcl                     */
#/*                                                */
#/* Standard Cell Library: Nangate45nm             */
#/**************************************************/

#/* All verilog files, separated by spaces         */
set my_verilog_files [list accu.v]

#/* Top-level Module                               */
set my_toplevel accu

#/* Target frequency in MHz for optimization       */
set my_clk_freq_MHz 1000

#/* The name of the clock pin. If no clock-pin     */
#/* exists, pick anything                          */
set my_clock_pin clk

#/* Delay of input signals (Clk-to-Q, Package etc.)  */
set my_input_delay_ns 0.1

#/* Reserved time for output signals (Holdtime etc.)   */
set my_output_delay_ns 0.1

#/**************************************************/
#/* No modifications needed below                  */
#/**************************************************/

define_design_lib WORK -path ./WORK
set_app_var target_library "stdcells.db"
set_app_var link_library "* stdcells.db"

analyze -format verilog $my_verilog_files
elaborate $my_toplevel

set my_period [expr 1000 / $my_clk_freq_MHz]
set find_clock [ find port [list $my_clock_pin] ]
if {  $find_clock != [list] } {
   set clk_name $my_clock_pin
   create_clock -period $my_period $clk_name
} else {
   set clk_name vclk
   create_clock -period $my_period -name $clk_name
}
set_input_delay $my_input_delay_ns -clock $clk_name [all_inputs]
set_output_delay $my_output_delay_ns -clock $clk_name [all_outputs]

check_design
compile 

set filename [format "%s%s" $my_toplevel "_post_synth.v"]
write -format verilog -output $filename
set filename [format "%s%s" $my_toplevel "_post_synth.sdc"]
write_sdc $filename

redirect timing.rep { report_timing }
redirect cell.rep { report_cell }
redirect power.rep { report_power }

quit
```
#### Commands Desciption of the compile_dc.tcl file: 
1. The **target_library** variable specifies the standard cells that Synopsys DC should use when synthesizing the RTL.
2. The **link_library** variable should search the standard cells, but can also search other cells (e.g., SRAMs) when trying to resolve references in our design. These other cells are not meant to be available for Synopsys DC to use during synthesis, but should be used when resolving references. Including * in the link_library variable indicates that Synopsys DC should also search all cells inside the design itself when resolving references.
3. We are now ready to read in the Verilog file which contains the top-level design and all referenced modules. We do this with two commands.
   - The **analyze** command reads the Verilog RTL into an intermediate internal representation.
   - The **elaborate** command recursively resolves all of the module references starting from the top-level module, and also infers various registers and/or advanced data-path components.
4. We need to create a clock constraint to tell Synopsys DC what our target cycle time is. Synopsys DC will not synthesize a design to run “as fast as possible”. Instead, the designer gives Synopsys DC a target cycle time and the tool will try to meet this constraint while minimizing area and power.
   - The create_clock command takes the name of the clock signal in the Verilog (which in this course will always be **clk**), the label to give this clock (i.e., ideal_clock1), and the target clock period in nanoseconds. So in this example, we are asking Synopsys DC to see if it can synthesize the design to run at 1.0GHz (i.e., a cycle time of 1000ps).
   - In an ideal world, all inputs and outputs would change immediately with the clock edge. In reality, this is not the case. We need to include reasonable delays for inputs and outputs, so Synopsys DC can factor this into its timing analysis so we would still meet timing if we were to tape our design out in real silicon. Here, we choose 10% of the clock period for our input and output delays.
5. The **check_design** command to make sure there are no obvious errors in our Verilog RTL.
6. The **compile** command will do the synthesis.
    - During synthesis, Synopsys DC will display information about its optimization process. It will report on its attempts to map the RTL into standard-cells, optimize the resulting gate-level netlist to improve the delay, and then optimize the final design to save area.
    - The **compile** command does not perform many optimizations. Synopsys DC also includes **compile_ultra** which does many more optimizations and will likely produce higher quality of results. Keep in mind that the compile command will not flatten your design by default, while the compile_ultra command will flattened your design by default. You can turn off flattening by using the **-no_autoungroup** option with the compile_ultra command. **compile_ultra** also has the option -gate_clock which automatically performs clock gating on your design, which can save quite a bit of power. Once you finish this tutorial, feel free to go back and experiment with the compile_ultra command.
7. Now that we have synthesized the design, we output the resulting gate-level netlist in the Verilog format. We also output an .sdc file which contains the constraint information we gave Synopsys DC. We will pass this same constraint information to Cadence Innovus during the place and route portion of the flow.
8. We can use various commands to generate reports about area, energy, and timing.
    - The **report_timing** command will show the critical path through the design. Part of the report is displayed below.
        ```
        ...
          Point                                    Incr       Path
          -----------------------------------------------------------
          clock clk (rise edge)                    0.00       0.00
          clock network delay (ideal)              0.00       0.00
          input external delay                     0.10       0.10 f
          in[0] (in)                               0.00       0.10 f
          add_30/B[0] (accu_DW01_add_0)            0.00       0.10 f
          add_30/U1/ZN (AND2_X1)                   0.04       0.14 f
          add_30/U1_1/CO (FA_X1)                   0.09       0.22 f
          add_30/U1_2/CO (FA_X1)                   0.09       0.31 f
          add_30/U1_3/CO (FA_X1)                   0.09       0.40 f
          add_30/U1_4/CO (FA_X1)                   0.09       0.50 f
          add_30/U1_5/CO (FA_X1)                   0.09       0.59 f
          add_30/U1_6/CO (FA_X1)                   0.09       0.68 f
          add_30/U1_7/S (FA_X1)                    0.13       0.81 r
          add_30/SUM[7] (accu_DW01_add_0)          0.00       0.81 r
          U21/ZN (AND2_X1)                         0.04       0.84 r
          r7/d (dff_1)                             0.00       0.84 r
          r7/q_reg/D (DFF_X1)                      0.01       0.85 r
          data arrival time                                   0.85
        
          clock clk (rise edge)                    1.00       1.00
          clock network delay (ideal)              0.00       1.00
          r7/q_reg/CK (DFF_X1)                     0.00       1.00 r
          library setup time                      -0.03       0.97
          data required time                                  0.97
          -----------------------------------------------------------
          data required time                                  0.97
          data arrival time                                  -0.85
          -----------------------------------------------------------
          slack (MET)                                         0.12
        ```
      - This timing report uses static timing analysis to find the critical path. Static timing analysis checks the timing across all paths in the design (regardless of whether these paths can actually be used in practice) and finds the longest path. For more information about static timing analysis, consult Chapter 1 of the Synopsys Timing Constraints and Optimization User Guide.
      - The difference between the required arrival time and the actual arrival time is called the slack. Positive slack means the path arrived before it needed to while negative slack means the path arrived after it needed to. If you end up with negative slack, then you need to rerun the tools with a longer target clock period until you can meet timing with no negative slack. The process of tuning a design to ensure it meets timing is called “timing closure”. In this course, we are primarily interested in design-space exploration as opposed to meeting some externally defined target timing specification. So you will need to sweep a range of target clock periods. Your goal is to choose the shortest possible clock period which still meets timing without any negative slack! This will result in a well-optimized design and help identify the “fundamental” performance of the design. Alternatively, if you are comparing multiple designs, sometimes the best situation is to tune the baseline so it meets timing and then ensure the alternative designs have similar cycle times. This will enable a fair comparison since all designs will be running at the same cycle time.
    - The **report_cell** command will show the number of cells in the design.
    - The **report_power** command can show how much area each module uses and can enable detailed area breakdown analysis.


**Step 3: Synthesis**
Once you have the script file ready, you can go ahead to synthesize the circuit:
```
dc_shell -f compile_dc.tcl
```
Design Compiler will run for a short time and create substantial amounts of output. When it is finished it will return to the command line. If there is an error it will specify the exact source of the error and the line number in the script that was responsible for the error. We can look at the output of DC. As we said above, it is a gate-level Verilog netlist that only contains interconnected standard cells. The netlist is called **accu_post_synth.v** and you can use any text editor to check its content. 
```
module accu_DW01_add_0 ( A, B, CI, SUM, CO );
  input [7:0] A;
  input [7:0] B;
  output [7:0] SUM;
  input CI;
  output CO;
  wire   n1;
  wire   [7:1] carry;

  FA_X1 U1_7 ( .A(A[7]), .B(B[7]), .CI(carry[7]), .S(SUM[7]) );
  FA_X1 U1_6 ( .A(A[6]), .B(B[6]), .CI(carry[6]), .CO(carry[7]), .S(SUM[6]) );
  FA_X1 U1_5 ( .A(A[5]), .B(B[5]), .CI(carry[5]), .CO(carry[6]), .S(SUM[5]) );
  FA_X1 U1_4 ( .A(A[4]), .B(B[4]), .CI(carry[4]), .CO(carry[5]), .S(SUM[4]) );
  FA_X1 U1_3 ( .A(A[3]), .B(B[3]), .CI(carry[3]), .CO(carry[4]), .S(SUM[3]) );
  FA_X1 U1_2 ( .A(A[2]), .B(B[2]), .CI(carry[2]), .CO(carry[3]), .S(SUM[2]) );
  FA_X1 U1_1 ( .A(A[1]), .B(B[1]), .CI(n1), .CO(carry[2]), .S(SUM[1]) );
  AND2_X1 U1 ( .A1(B[0]), .A2(A[0]), .ZN(n1) );
  XOR2_X1 U2 ( .A(B[0]), .B(A[0]), .Z(SUM[0]) );
endmodule


module accu ( in, accu, clk, rst );
  input [7:0] in;
  output [7:0] accu;
  input clk, rst;
  wire   N3, N4, N5, N6, N7, N8, N9, N10, n5;
  wire   [7:0] dff_in;

  dff_0 r0 ( .d(dff_in[0]), .q(accu[0]), .clk(clk) );
  dff_7 r1 ( .d(dff_in[1]), .q(accu[1]), .clk(clk) );
  dff_6 r2 ( .d(dff_in[2]), .q(accu[2]), .clk(clk) );
  dff_5 r3 ( .d(dff_in[3]), .q(accu[3]), .clk(clk) );
  dff_4 r4 ( .d(dff_in[4]), .q(accu[4]), .clk(clk) );
  dff_3 r5 ( .d(dff_in[5]), .q(accu[5]), .clk(clk) );
  dff_2 r6 ( .d(dff_in[6]), .q(accu[6]), .clk(clk) );
  dff_1 r7 ( .d(dff_in[7]), .q(accu[7]), .clk(clk) );
  accu_DW01_add_0 add_30 ( .A(accu), .B(in), .CI(1'b0), .SUM({N10, N9, N8, N7,
        N6, N5, N4, N3}) );
  AND2_X1 U13 ( .A1(N3), .A2(n5), .ZN(dff_in[0]) );
  AND2_X1 U14 ( .A1(N9), .A2(n5), .ZN(dff_in[6]) );
  AND2_X1 U15 ( .A1(N4), .A2(n5), .ZN(dff_in[1]) );
  AND2_X1 U16 ( .A1(N5), .A2(n5), .ZN(dff_in[2]) );
  AND2_X1 U17 ( .A1(N6), .A2(n5), .ZN(dff_in[3]) );
  AND2_X1 U18 ( .A1(N7), .A2(n5), .ZN(dff_in[4]) );
  AND2_X1 U19 ( .A1(N8), .A2(n5), .ZN(dff_in[5]) );
  INV_X1 U20 ( .A(rst), .ZN(n5) );
  AND2_X1 U21 ( .A1(N10), .A2(n5), .ZN(dff_in[7]) );
endmodule
```
Note that the top-level module still has the name 'accu' and the names of the inputs and outputs have not changed. From the outside it is exactly the same circuit as you coded on the RTL level. But on the inside all functionality is now expressed only in terms of standard cells.

#### **Step 4: Post Synthesis Simulation**
A post-synthesis simulation can be performed by including Verilog models of the standard cells available from 'gscl45nm.v'. The command line is:
```
xrun gscl45nm.v tb_accu.v accu_post_synth.v +access+r
```
Note how we re-used the original testbench from the RTL level simulation. That is an excellent way to ensure that the gate-level representation matches the RTL level. The simulation results should look similar to before.

![5](./fig/post_syn_sim.png)

---

