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


## Day 5 - Final steps for RTL2GDS using tritonRoute and openSTA

