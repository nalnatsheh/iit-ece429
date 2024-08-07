# Tutorial IV: Standard Cell Based ASIC Design Flow

## Overview:
This tutorial introduces you to the standard cell based ASIC design flow using tools and libraries from various vendors. We will use the Nangate 45nm standard-cell library with the FreePDK45 to implement an 8-bit accumulator design. We will first synthesize the design using the Synopsys Design Compiler and then perform place and route using the Cadence Innovus. The final layout will be painted in the Cadence Virtuoso platform and the final design will be verified by the Synopsys Formality Equivalence Checker.

Note: Since network failure may interrupt your operation, please save your data often.

## Table of Contents:
1. RTL Simulation
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
### 1. RTL Simulation

#### Step 1:
Typically you enter code in Verilog on the Register-Transfer Level (RTL), where you model your design using clocked registers, datapath elements and control elements. You will use Cadence Verilog-XL to simulate your design. You will also need to create a Verilog testbench for your circuit. 
In this tutorial there are 2 files as follows:

1. accu.v: Verilog RTL code for an 8-bit accumulator

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

2. tb_accu.v: Verilog testbench for accu.v

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
           in <= 1;
        #5 rst<=0;
 end

endmodule
```

#### Step 2:
In order to simulate Verilog code via Cadence Verilog-XL, use this command:
```
xrun tb_accu.v accu.v +access+r
```
This testbench provides results directly on the screen and also in a waveform database. From the screen we can see that the design behaves as expected as follows.

![1](./fig/rtl_sim.png)

That is, every 10ns we add 1 to the accumulator. This is expected since in the testbench a clock of 10ns is specified and the input 'in' is connected to a constant 1.

#### Step 3:
We use the program Cadence SimVision to look at the waveform database that was created by Verilog-XL. Type the following command:
```
simvision
```
![1](./fig/simvision_1.png)

Now we need to open the Waveform database. 
1. Click on "File" > "Open Database..." or "ctl+o".
2. Choose the directory "shm.db", which is where the file is located, and double-click on the file "shm.trn" to open it.

![1](./fig/simvision_2.png)

3. To see the contents of the waveform database, from the "Design Browser" menu on the left, click on tb_accu, and the four waveforms (accu[7:0], clk, in[7:0], rst) appears on the show contents, click on each one of them to add the waveforms.

![1](./fig/simvision_3.png)

---

### 1. Logic Synthesis using Synopsys Design Compiler
Once you have verified that your Verilog RTL code is working correctly you can synthesize it into standard cells. The result will be a gate-level netlist that only contains interconnected standard cells.

Once you have the script file ready, you can go ahead to synthesize the circuit:
```
dc_shell -f compile_dc.tcl
```
Design Compiler will run for a short time and create substantial amounts of output. When it is finished it will return to the command line. If there is an error it will specify the exact source of the error and the line number in the script that was responsible for the error. Typically there will be no errors. We can verify the initial estimations of area, timing, and power by reading into the following three files 'cell.rep', 'timing.rep', and 'power.rep'.

We can also look at the output of DC. As we said above, it is a gate-level Verilog netlist that only contains interconnected standard cells. The netlist is called 'accu.vh' and you can use any text editor to check its content. Note that the top-level module still has the name 'accu' and the names of the inputs and outputs have not changed. From the outside it is exactly the same circuit as you coded on the RTL level. But on the inside all functionality is now expressed only in terms of standard cells. A post-synthesis simulation can be performed by including Verilog models of the standard cells available from 'gscl45nm.v'. The command line is:
```
xrun gscl45nm.v accu_test.v accu.vh +access+r
```
Note how we re-used the original testbench from the RTL level simulation. That is an excellent way to ensure that the gate-level representation matches the RTL level. The simulation results should look similar to before.
