# Advanced-PD-using-OpenLANE-Sky130

## Introduction
[To be updated later]

## Day 1 - Inception of open-source EDA, OpenLANE and Sky130 PDK

### Directory Structure
The main directory is the "openlane_working_dir" under which we have two sub-directories
  1. openlane - the working directory where we will be running our openlane environment
  2. pdks - this directory contains all the PDK (process design kit) related files which can be used with openlane to model our designs.

![Dir Structure](https://user-images.githubusercontent.com/32140302/183107323-3621ac9f-4c17-4fd3-b000-947dd2f24c68.jpg)

### Intializing the Openlane Environment

  1. Before we can invoke openlane we need to run docker.
```
docker run -it -v $(pwd):/openLANE_flow -v $PDK_ROOT:$PDK_ROOT -e PDK_ROOT=$PDK_ROOT -u $(id -u $USER):$(id -g $USER) openlane:rc2
```
OR try running just `docker` if you run into any issues.

  2. Next the Openlane flow is invoked using `flow.tcl` which can be run in two modes :
     * Autonomous mode - default option where full flow from RTL-to-GDSII is run automatically
     * Interactve mode - use `-interactive` option to run a step-by-step flow

  3. Following this we import the required version of openlane which helps to get all the different tool related dependencies
```
package require openlane 0.9
```

  4. Finally we setup the design of our choice work. Using the configuration file, this merges the tech LEF and cell LEF files and buids a complete directory structure for our run .
```
prep -design picorv32a
```
![Setup](https://user-images.githubusercontent.com/32140302/183119503-98feeb97-838d-48a6-8313-ef8a1cb527f8.jpg)

### Synthesis

In OpenLANE flow this is achieved using `run_synthesis` command which runs a combination of 3 tools to build a gate-level optimized netlist from the RTL. 
  * abc - technology mapping
  * yosys - rtl synthesis
  * opensta - timing analysis of the generated netlist

![Synthesis stats](https://user-images.githubusercontent.com/32140302/183127043-a95b14f3-86b0-4ecc-829e-1aedf2c41674.jpg)

## Day 2 - Good floorplan vs bad floorplan and introduction to library cells

### Floorplanning

In OpenLANE flow, flooplanning is done using `run_floorplan` command which fixes the location of any pre-placed cells (like memories, analog IPs, etc. alongwith the physical-only cells like De-Coupling capacitors, Endcap cells, Tap cells, etc.) in the defined design core. IO pin location is also fixed during this step. Additionally the power mesh (power distribution network) is built while floorplanning.
The designer must take into consideration multiple factors (technology specific as well as design specific) while defining different set of parameters which control the floorplan for a design. This might become an iterative process since floorplan is a very critical step of the PnR flow.


  * To review floorplan in GUI form `magic` tool can be used
```
## For DEF with pre-placed cells and pins placed
magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech lef read tmp/merged.lef def read results/floorplan/picorv32a.floorplan.def &

## For DEF with pre-placed cells, pins placed and the PDN
magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech lef read tmp/merged.lef def read tmp/floorplan/12-pdn.def &
```
![Floorplan with PDN - Full view](https://user-images.githubusercontent.com/32140302/183156086-79544904-9df1-4ae8-b965-0a60d3e996d3.jpg)

![Floorplan with PDN - Zoomed In](https://user-images.githubusercontent.com/32140302/183156090-185e90d5-9ace-44df-9c6d-c84b4e7a02a8.jpg)

### Placement

In OpenLANE flow, placement is done using `run_placement` command which optimizes the netlist and places the standard cells. The main focus in this flow is to reduce Half-Parameter Wire Length and the objective is to decrease the value of overflow.
There are multiple options available using which design optimization at placement stage can be controlled. Like we can control if the placer be timing driven and/or routability driven. Also we can set a target placement density to control cell spread in the core. The designer can control if and how driver resizing should take place.

![Placement - DEF View](https://user-images.githubusercontent.com/32140302/183212744-9fc579f5-4c2c-4fde-8fd4-d93ea94e19e3.jpg)

![Placement Stats](https://user-images.githubusercontent.com/32140302/183212765-c06df61b-aa1b-414c-bcf3-71bffa9bfe30.jpg)


## Day 3 - Design library cell using Magic Layout and ngspice characterization

### CMOS Inverter in Magic

For the purpose of this lab, we are using a custom cell layout of CMOS Inverter already created by Nickson Jose and we use it perform spice extractions and post-layout spice simulations.
The required lib files and .mag file are cloned into our workspace via the git repository https://github.com/nickson-jose/vsdstdcelldesign and also copied the magic tech file in our workspace for ease of access. The inverter is now ready to be loaded in magic GUI.
Magic tool's interactive DRC mode makes creation of any new cell layout very convenient.

![CMOS Inv - setup](https://user-images.githubusercontent.com/32140302/183227344-bbcee9db-4327-417f-afea-69e285d61f2c.jpg)

![CMOS Inv - magic](https://user-images.githubusercontent.com/32140302/183227437-17840df3-9fbe-4f41-baa0-7ff8a0a302e1.jpg)

### Extracting SPICE Netlist from Cell Layout

First an extraction file `sky130_inv.ext` is created using `extract all`

![sky130_inv_ext](https://user-images.githubusercontent.com/32140302/183232196-fb443b9c-2237-4e77-973c-61c77d40fc03.jpg)

Then a spice model `sky130_inv.spice` with parasitic values is created by running `ext2spice cthresh 0 rthresh 0` followed by the command `ext2spice` .

![sky130_inv_spice](https://user-images.githubusercontent.com/32140302/183232197-32b12da7-15b8-4cad-97fe-a62ad09ec30a.jpg)


### Transient Analysis and Cell Characterization using NGSPICE

Before doing the transient analysis, we need to build a spice deck from the spice model file by adding the power connections, adding transient input signals and defining the parameters for required analysis.

![spice deck](https://user-images.githubusercontent.com/32140302/183233939-1a915443-e606-42c6-afc7-3c8d0a50207b.jpg)

Now we load this updated spice deck in ngspice using the command `ngspice sky130_inv.spice` and inverter input/output vs time plot is obtained using `plot y vs time A` .
This plot can now be used to calculate the cell characteristics.

***Transition<sub>rise</sub> = 58.32ps***

***Transition<sub>fall</sub> = -42.92ps***

***PropagationDelay<sub>rise</sub> = 56.43ps***

***PropagationDelay<sub>fall</sub> = 24.8ps***

![ngspice](https://user-images.githubusercontent.com/32140302/183233937-f626c365-9991-4c27-af65-562b66e6ae21.jpg)

![transient analysis plot](https://user-images.githubusercontent.com/32140302/183233940-48e32da4-37ff-4924-a22e-b8add3c0a75f.jpg)


## Day 4 - Pre-layout timing analysis and importance of good clock tree

### Creating Std Cell LEF

PnR tool mainly requires just the PR boundary, metal and pin information of any cell to perform optimizations and provide a GDSII file from the input synthesized netlist. This abstracted information to the tool is provided to the tool in form of a LEF (library exchange format) file and it provides sufficient information to the tool to perform routing. Additionally LEF file helps to protect the IP since no cnnectivity information is disclosed.

For each technology, tracks information is provided by the foundary and they are necessary because routes for each layer can only go over their respective tracks.

![tracks](https://user-images.githubusercontent.com/32140302/183273661-5c3e4b38-b02f-4cd3-90db-4529e3101196.jpg)

There are many guidelines need to be taken care of when generating LEF for any cell in order to ensure routability of the design and avoid any DRCs :
  * Input nd output ports must lie on the intersection of the intersection of the vertical and horizontal tracks.
  * Width of the cell should be an odd multiple of x-pitch of the layer.
  * Height of the cell should be an odd multiple of the y-pitch of the layer.
  * All cell ports (pins) have proper definition (layer, direction, usage etc)

In magic, we set the grid size according to track info to ensure cell alignment is proper.

```
grid 0.46um 0.34um 0.23um 0.17um
```

![grid](https://user-images.githubusercontent.com/32140302/183273664-049c6198-868b-47e2-97ec-b323e60327c5.jpg)

Then we need to ensure that all ports have proper port definitions.

![portA](https://user-images.githubusercontent.com/32140302/183274723-fb375c6a-6ebc-4d53-94ea-d991ce91d627.jpg)
![portY](https://user-images.githubusercontent.com/32140302/183274722-1a4d04d5-c4f5-4da5-96d5-d131acffb9fa.jpg)
![portVDD](https://user-images.githubusercontent.com/32140302/183274724-3152bb46-683e-4c10-8536-11efab8b7f40.jpg)
![portVSS](https://user-images.githubusercontent.com/32140302/183274725-ef96c71c-7300-4e03-b0af-36d2d06a446c.jpg)

Finally we can write our lef file using `lef write` command.

```
VERSION 5.7 ;
  NOWIREEXTENSIONATPIN ON ;
  DIVIDERCHAR "/" ;
  BUSBITCHARS "[]" ;
MACRO sky130_ssinv
  CLASS CORE ;
  FOREIGN sky130_ssinv ;
  ORIGIN 0.000 0.000 ;
  SIZE 1.380 BY 2.720 ;
  SITE unithd ;
  PIN A
    DIRECTION INPUT ;
    USE SIGNAL ;
    ANTENNAGATEAREA 0.165600 ;
    PORT
      LAYER li1 ;
        RECT 0.060 1.180 0.510 1.690 ;
    END
  END A
  PIN Y
    DIRECTION OUTPUT ;
    USE SIGNAL ;
    ANTENNADIFFAREA 0.287800 ;
    PORT
      LAYER li1 ;
        RECT 0.760 1.960 1.100 2.330 ;
        RECT 0.880 1.690 1.050 1.960 ;
        RECT 0.880 1.180 1.330 1.690 ;
        RECT 0.880 0.760 1.050 1.180 ;
        RECT 0.780 0.410 1.130 0.760 ;
    END
  END Y
  PIN VPWR
    DIRECTION INOUT ;
    USE POWER ;
    PORT
      LAYER nwell ;
        RECT -0.200 1.140 1.570 3.040 ;
      LAYER li1 ;
        RECT -0.200 2.580 1.430 2.900 ;
        RECT 0.180 2.330 0.350 2.580 ;
        RECT 0.100 1.970 0.440 2.330 ;
      LAYER mcon ;
        RECT 0.230 2.640 0.400 2.810 ;
        RECT 1.000 2.650 1.170 2.820 ;
      LAYER met1 ;
        RECT -0.200 2.480 1.570 2.960 ;
    END
  END VPWR
  PIN VGND
    DIRECTION INOUT ;
    USE GROUND ;
    PORT
      LAYER li1 ;
        RECT 0.100 0.410 0.450 0.760 ;
        RECT 0.150 0.210 0.380 0.410 ;
        RECT 0.000 -0.150 1.460 0.210 ;
      LAYER mcon ;
        RECT 0.210 -0.090 0.380 0.080 ;
        RECT 1.050 -0.090 1.220 0.080 ;
      LAYER met1 ;
        RECT -0.110 -0.240 1.570 0.240 ;
    END
  END VGND
END sky130_ssinv
END LIBRARY
```

### Synthesizing with Custom Inverter

In order to use this cell in our OpenLANE flow, we need to add the related libs and LEF in our flow.

```
## while in your openlane directory follow the below steps
## copy the LEF we just created
cp -f vsdstdcelldesign/sky130_ssinv.lef designs/picorv32a/src/.
## copy the lib files containing the cell characterizations at different corners (these can be created using GUNA)
cp -f vsdstdcelldesign/libs/sky130_fd_sc_hd__* designs/picorv32a/src/.
## update our custom inverter name in the libs we just copied
sed -i -E 's,sky130_vsdinv,sky130_ssinv,g' designs/picorv32a/src/sky130_fd_sc_hd__*
```
Next we need to update the onfiguration file for our design so that our custom LEF and libs are read in the OpenLANE flow.

![updated config file](https://user-images.githubusercontent.com/32140302/183280731-b1b88abc-6fd4-4b5e-8a6c-824508e72466.jpg)

Now we can setup the OpenLANE flow and run synthesis to check if our custom cell is now picked in the flow or not.

```
## setup flow
docker
./flow.tcl -interactive
package require openlane 0.9
prep -design picorv32a -tag 07-08_06-58 -overwrite

## ensure custom lef is read in the flow
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs

## synthesize the design
run_synthesis
```

![updated Synthesis stats](https://user-images.githubusercontent.com/32140302/183280732-a7b0b4f6-36a6-476a-a43b-ffed677d9fc1.jpg)

### Synthesis Strategy

As per the design requirements, we can play around with multiple options/parameters provided in the OpenLANE flow and fine tune our QOR (quality of results).

For a trial, I updated one such setting. Although timing improved with this startegy but area bloat is seen, which shows the classic trade-off between timing and area.
Considering such PPA (power performance area) trade-offs, a designer has to build custom startegies for each design.

![synthesis2](https://user-images.githubusercontent.com/32140302/183284066-b7f91465-bd0e-4487-80b7-7a455f12c1f1.jpg)

![synthesis2 stats](https://user-images.githubusercontent.com/32140302/183284065-c4ca229c-33dd-44e2-915e-77e7920055d7.jpg)

### Performing Synthesis STA

For running sta using OpenSTA, we first need to create a SDC `my_base.sdc` with same values as used in our synthesis runs and create a configuration file `pre_sta.conf`. Now sta can be performed using `sta pre_sta.conf` .


```
## my_base.sdc


set ::env(CLOCK_PORT) clk
set ::env(CLOCK_PERIOD) 24.73
set ::env(SYNTH_DRIVING_CELL) sky130_fd_sc_hd__inv_8
set ::env(SYNTH_DRIVING_CELL_PIN) Y
set ::env(SYNTH_CAP_LOAD) 17.65
create_clock [get_ports $::env(CLOCK_PORT)]  -name $::env(CLOCK_PORT)  -period $::env(CLOCK_PERIOD)
set IO_PCT  0.2
set input_delay_value [expr $::env(CLOCK_PERIOD) * $IO_PCT]
set output_delay_value [expr $::env(CLOCK_PERIOD) * $IO_PCT]
puts "\[INFO\]: Setting output delay to: $output_delay_value"
puts "\[INFO\]: Setting input delay to: $input_delay_value"


set clk_indx [lsearch [all_inputs] [get_port $::env(CLOCK_PORT)]]
#set rst_indx [lsearch [all_inputs] [get_port resetn]]
set all_inputs_wo_clk [lreplace [all_inputs] $clk_indx $clk_indx]
#set all_inputs_wo_clk_rst [lreplace $all_inputs_wo_clk $rst_indx $rst_indx]
set all_inputs_wo_clk_rst $all_inputs_wo_clk


# correct resetn
set_input_delay $input_delay_value  -clock [get_clocks $::env(CLOCK_PORT)] $all_inputs_wo_clk_rst
#set_input_delay 0.0 -clock [get_clocks $::env(CLOCK_PORT)] {resetn}
set_output_delay $output_delay_value  -clock [get_clocks $::env(CLOCK_PORT)] [all_outputs]

# TODO set this as parameter
set_driving_cell -lib_cell $::env(SYNTH_DRIVING_CELL) -pin $::env(SYNTH_DRIVING_CELL_PIN) [all_inputs]
set cap_load [expr $::env(SYNTH_CAP_LOAD) / 1000.0]
puts "\[INFO\]: Setting load to: $cap_load"
set_load  $cap_load [all_outputs]
```

```
## pre_sta.conf

set_cmd_units -time ns -capacitance pF -current mA -voltage V -resistance kOhm -distance um
read_liberty -min /home/sharmasarthak96/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib
read_liberty -max /home/sharmasarthak96/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib
read_verilog /home/sharmasarthak96/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/07-08_06-58/results/synthesis/picorv32a.synthesis.v
link_design picorv32a
read_sdc /home/sharmasarthak96/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/my_base.sdc
report_checks -path_delay min_max -fields {slew trans net cap input_pin}
report_tns
report_wns
```

![synth sta](https://user-images.githubusercontent.com/32140302/183287035-cde0c36d-ed8f-4202-af15-654fcb9132ac.jpg)

### Floorplan & Placement

Again with our custom cell synthesized netlist we do floorplanning and placement.

```
### since faced error running floorplan with run_floorplan, we run floorplan interactively referring the steps from scripts/tcl_commands/floorplan.tcl
init_floorplan
place_io
apply_def_template
global_placement_or
basic_macro_placement
tap_decap_or
scrot_klayout -layout $::env(CURRENT_DEF)
run_power_grid_generation

### placement
run_placement
```

![placement2 stats](https://user-images.githubusercontent.com/32140302/183289836-d65bcb1e-eaa1-4325-b5f2-ad16b69c49d5.jpg)


### Clock Tree Synthesis

CTS is the backbone of any design as it ensures that clock is distributed to all sequential elements properly with minimal clock skew. This is done in OpenLANE flow by TritonCTS and the command is `run_cts` .

![cts stats 1](https://user-images.githubusercontent.com/32140302/183300868-dd668569-6412-4f0d-893b-df028301e684.jpg)

```
## For post-CTS STA using openroad
## **CTS tree in TritonCTS is built considering single corner at a time (typical by default). Hence may see timing violations at slow/fast corners.**

openroad
read_lef /openLANE_flow/designs/picorv32a/runs/07-08_06-58/tmp/merged.lef
read_def /openLANE_flow/designs/picorv32a/runs/07-08_06-58/results/cts/picorv32a.cts.def
write_db picorv32a_cts.db
read_db picorv32a_cts.db

read_verilog /openLANE_flow/designs/picorv32a/runs/07-08_06-58/results/synthesis/picorv32a.synthesis_cts.v
read_liberty $::env(LIB_SYNTH_COMPLETE)
### read_liberty -min $::env(LIB_FASTEST)
### read_liberty -max $::env(LIB_SLOWEST)

link_design picorv32a

read_sdc /openLANE_flow/designs/picorv32a/src/my_base.sdc

set_propagated_clock [all_clocks]

report_checks -path_delay min_max -fields {slew trans net cap input_pin} -format full_clock_expanded >> /openLANE_flow/designs/picorv32a/runs/07-08_06-58/reports/cts/cts_timing.openroad.typical.rpt

```

![cts stats 2](https://user-images.githubusercontent.com/32140302/183300867-a552bb0a-5fd5-49e0-bab2-064cc0dc1fa4.jpg)

## Day 5 - Final steps for RTL2GDS using tritonRoute and openSTA

### Routing

The final step of PnR is routing which in OpenLANE flow is performed using `run_routing`.

![route stats 1](https://user-images.githubusercontent.com/32140302/183305425-4b07415b-5272-466e-af57-c8f4aa336e27.jpg)

![route stats 2](https://user-images.githubusercontent.com/32140302/183305300-a9208b0d-4b98-4aac-835e-1aa417c7381c.jpg)

![route](https://user-images.githubusercontent.com/32140302/183305708-66b07b24-cad8-4fcf-a9bc-e67dac71cc38.jpg)

## SPEF Generation

Although the updated OpenLANE flow, automatically generates the SPEF file if needed SPEF can be generated by following the below mentioned commands.

```
cd ~/Desktop/work/tools/openlane_working_dir/openlane/scripts/spef_extractor
python3 main.py designs/picorv32a/runs/07-08_06-58/tmp/merged.lef ~/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/07-08_06-58/tmp/merged.lef ~/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/07-08_06-58/results/routing/picorv32a.def
```

## Acknowledgements

  * [Kunal Ghosh](https://github.com/kunalg123)
  * [Nickson Jose](https://github.com/nickson-jose)
  * [Google Sky130](https://github.com/google/skywater-pdk)
  * [Efabless](https://github.com/efabless)
  * [OpenLANE](https://github.com/The-OpenROAD-Project/OpenLane)
  * [OpenROAD](https://github.com/The-OpenROAD-Project)
  * [Yosys - picorv32a](https://github.com/YosysHQ/picorv32)


