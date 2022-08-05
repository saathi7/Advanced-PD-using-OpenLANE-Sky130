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



## Day 2 - Good floorplan vs bad floorplan and introduction to library cells


## Day 3 - Design library cell using Magic Layout and ngspice characterization


## Day 4 - Pre-layout timing analysis and importance of good clock tree


## Day 5 - Final steps for RTL2GDS using tritonRoute and openSTA

