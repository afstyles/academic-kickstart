---
title: Compiling NEMO 4.0.1 on Monsoon/NEXCS
summary: A guide to compiling NEMO 4.0 on Monsoon/NEXCS.
date: "2020-10-28T00:00:00Z"

reading_time: false  # Show estimated reading time?
share: true  # Show social sharing links?
profile: true  # Show author profile?
comments: false  # Show comments?

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
---
***
**Note:** This guide has been updated to work with NEMO 4.0.1 using Intel compilers.
If for any reason you want to look at my original guide for an earlier release using GCC compilers click [here](/nemo-compilation-archive/)
***
 
These are my personal notes on how to set up and compile NEMO 4.0.1 on Monsoon/NEXCS.

When compiling NEMO I could not find any guide for this particular system but used the following excellent guides for inspiration:

* [Julian Mak's](https://nemo-related.readthedocs.io/en/latest/compilation_notes/nemo40.html) guide to compiling NEMO 4.0 on local machines and Oxford ARC.

* [Christopher Bull's](http://christopherbull.com.au/nemo/nemo-wed12-01/) guide for compiling NEMO on ARCHER

As these guides were so helpful, I feel it is only fair to make a similar contribution.

I must also give a huge thanks to Dr. Andrew Coward from the National Oceanography Centre who has helped me to update this guide. 

## Step 1 - Loading modules

Once you are logged on to the Met Office lander, connect to NEXCS using
```shell
ssh xcs-c
```
On logging in you need to load the correct compilers and libraries. You will not be able to compile XIOS or NEMO without loading the correct modules. You can do this by running the following shell script.
```shell
#/bin/bash
#!
module swap PrgEnv-cray/5.2.82 PrgEnv-intel/5.2.82
module swap intel/15.0.0.090 intel/18.0.5.274
module load cray-hdf5-parallel/1.10.2.0
module load cray-netcdf-hdf5parallel/4.6.1.3
```
It will also be useful to load an editor
```shell
module load nano
```
***
**Note:** When you log in to NEXCS a set of essential modules are loaded by default. Do not use `module purge` before loading the above modules. 
***
Your loaded modules should look like the following:
```shell
astyles@xcslc0:~> module list
Currently Loaded Modulefiles:
  1) moose-client-wrapper                   9) craype-network-aries                  17) cray-libsci/13.0.1                    25) alps/5.2.4-2.0502.9774.31.11.ari
  2) python/v2.7.9                         10) gcc/4.8.1                             18) udreg/2.3.2-1.0502.10518.2.17.ari     26) rca/1.0.0-2.0502.60530.1.62.ari
  3) metoffice/tempdir                     11) intel/18.0.5.274                      19) ugni/6.0-1.0502.10863.8.29.ari        27) atp/1.7.5
  4) metoffice/userenv                     12) craype/2.2.1                          20) pmi/5.0.5-1.0000.10300.134.8.ari      28) PrgEnv-intel/5.2.82
  5) subversion-1.8/1.8.19                 13) craype-haswell                        21) dmapp/7.0.1-1.0502.11080.8.76.ari     29) cray-hdf5-parallel/1.10.2.0
  6) modules/3.2.10.5                      14) cray-mpich/7.0.4                      22) gni-headers/4.0-1.0502.10859.7.8.ari  30) cray-netcdf-hdf5parallel/4.6.1.3
  7) eswrap/1.3.3-1.020200.1280.0          15) pbs/18.2.5.20190913061152             23) xpmem/0.1-2.0502.64982.5.3.ari
  8) switch/1.0-1.0502.60522.1.61.ari      16) nano/2.8.6                            24) dvs/2.5_0.9.0-1.0502.2188.1.116.ari
```

## Step 2 - XIOS 2.5

XIOS controls the Input/Output for NEMO and needs to be compiled first. For NEMO 4.0 I use XIOS 2.5 but the method is very similar for other versions.

First navigate to the data directory on NEXCS and make a folder for XIOS
```shell
cd $DATADIR
mkdir XIOS
```
Inside the folder download XIOS 2.5
```shell
cd XIOS
svn checkout -r 1566 http://forge.ipsl.jussieu.fr/ioserver/svn/XIOS/branchs/xios-2.5 xios-2.5
```
We need to make sure XIOS is looking in the right place for the modules loaded in Step 1. We copy and rename the following files in the arch folder
```shell
cd xios-2.5/arch/ 
cp arch-XC30_Cray.env arch-XC30_NEXCS.env
cp arch-XC30_Cray.fcm arch-XC30_NEXCS.fcm 
cp arch-XC30_Cray.path arch-XC30_NEXCS.path
```
Then edit the files so they look like the following
```shell
#arch-XC30_NEXCS.env
export HDF5_INC_DIR=${HDF5_DIR}/include
export HDF5_LIB_DIR=${HDF5_DIR}/lib

export NETCDF_INC_DIR=${NETCDF_DIR}/include
export NETCDF_LIB_DIR=${NETCDF_DIR}/lib
```
***
**Note:** If you are using a different set of modules or worried that this path is wrong. The commands `which nc-config` and `which h5copy` will give you a path of the form `directory/bin`.  The paths used above should then be `directory/GNU/4.9/include` and similar for `lib`
***
```shell
#arch-XC30_NEXCS.fcm
################################################################################
###################                Projet XIOS               ###################
################################################################################

# Cray XC build instructions for XIOS/xios-1.0
# These files have been tested on
# Archer (XC30), ECMWF (XC30), and the Met Office (XC40) using the Cray PrgEnv.
# One must also:
#    module load cray-netcdf-hdf5parallel/4.3.2
# There is a bug in the CC compiler prior to cce/8.3.7 using -O3 or -O2
# The workarounds are not ideal:
# Use -Gfast and put up with VERY large executables
# Use -O1 and possibly suffer a significant performance loss.
#
# Mike Rezny Met Office 23/03/2015

%CCOMPILER      cc
%FCOMPILER      ftn
%LINKER         CC

%BASE_CFLAGS    -DMPICH_SKIP_MPICXX -h msglevel_4 -h zero -h noparse_templates
%PROD_CFLAGS    -O3 -DBOOST_DISABLE_ASSERTS
%DEV_CFLAGS     -g -O2
%DEBUG_CFLAGS   -g

%BASE_FFLAGS    -warn all -zero
%PROD_FFLAGS    -O3 -fp-model precise -warn all -zero
%DEV_FFLAGS     -g -O2
%DEBUG_FFLAGS   -g

%BASE_INC       -D__NONE__
%BASE_LD        -D__NONE__

%CPP            cpp
%FPP            cpp -P -CC
%MAKE           gmake
```
***
```shell
#arch-XC30_NEXCS.path
NETCDF_INCDIR="-I $NETCDF_INC_DIR"
NETCDF_LIBDIR='-Wl,"--allow-multiple-definition" -Wl,"-Bstatic" -L $NETCDF_LIB_DIR'
NETCDF_LIB="-lnetcdf -lnetcdff"

MPI_INCDIR=""
MPI_LIBDIR=""
MPI_LIB=""

#HDF5_INCDIR="-I $HDF5_INC_DIR"
HDF5_LIBDIR="-L $HDF5_LIB_DIR"
HDF5_LIB="-lhdf5_hl -lhdf5 -lz"

OASIS_INCDIR=""
OASIS_LIBDIR=""
OASIS_LIB=""

#OASIS_INCDIR="-I$PWD/../../prism/X64/build/lib/psmile.MPI1"
#OASIS_LIBDIR="-L$PWD/../../prism/X64/lib"
#OASIS_LIB="-lpsmile.MPI1 -lmpp_io"

```
Then you need to edit `bld.cfg` in `xios-2.5/`
```shell
cd $DATADIR/XIOS/xios-2.5
```
and change all occurrences of `src_netcdf` to `src_netcdf4` (two in total)

You are now ready to compile XIOS. In `xios-2.5/`
```shell
./make_xios --full --prod --arch XC30_NEXCS -j2
```
If this build is successful then XIOS is ready to use. If not,  check over the arch files and make sure the flags are correct and in the right order. Also make sure you have the correct modules loaded. Getting XIOS to compile is by far the most difficult part of the process.

## Step 3  - NEMO 4.0
Now we have XIOS ready to go we can start working on NEMO 4.0.

Go back to the data directory  and create a new folder for NEMO
```shell
cd $DATADIR
mkdir NEMO
```
Then download NEMO 4.0.1 in the folder.
```shell
cd NEMO
svn co https://forge.ipsl.jussieu.fr/nemo/svn/NEMO/releases/release-4.0.1
```
Now we need to copy and update the arch files for NEMO like we did in XIOS.
```shell
cd release-4.0.1/arch/
cp arch-linux_ifort.fcm arch-XC_MONSOON_INTEL.fcm
```
The file `arch-XC_MONSOON_INTEL.fcm` needs to look like
```shell
#arch-XC_MONSOON_INTEL.fcm
%NCDF_HOME           $NETCDF_DIR
%HDF5_HOME           $HDF5_DIR
%XIOS_HOME           $DATADIR/XIOS/xios-2.5
#OASIS_HOME

%NCDF_INC            -I%NCDF_HOME/include -I%HDF5_HOME/include
%NCDF_LIB            -L%HDF5_HOME/lib -L%NCDF_HOME/lib -lnetcdff -lnetcdf -lhdf5_hl -lhdf5 -lz
%XIOS_INC            -I%XIOS_HOME/inc
%XIOS_LIB            -L%XIOS_HOME/lib -lxios
#OASIS_INC           -I%OASIS_HOME/build/lib/mct -I%OASIS_HOME/build/lib/psmile.MPI1
#OASIS_LIB           -L%OASIS_HOME/lib -lpsmile.MPI1 -lmct -lmpeu -lscrip

%CPP                 cpp
%FC                  ftn
%FCFLAGS             -integer-size 32 -real-size 64 -O3 -fp-model source -zero -fpp -warn all
%FFLAGS              -integer-size 32 -real-size 64 -O3 -fp-model source -zero -fpp -warn all
%LD                  CC -Wl,"--allow-multiple-definition"
%FPPFLAGS            -P -C -traditional
%LDFLAGS
%AR                  ar
%ARFLAGS             -rvs
%MK                  gmake
%USER_INC            %XIOS_INC %NCDF_INC
%USER_LIB            %XIOS_LIB %NCDF_LIB
#USER_INC            %XIOS_INC %OASIS_INC %NCDF_INC
##USER_LIB            %XIOS_LIB %OASIS_LIB %NCDF_LIB

%CC                  cc
%CFLAGS              -O0
```
## Step 4 - Compiling the GYRE_PISCES configuration
Now we need a configuration to compile. In this example I use GYRE_PISCES which is a reference configuration in NEMO as it is a relatively simple configuration. Other reference configurations are listed [here](https://forge.ipsl.jussieu.fr/nemo/chrome/site/doc/NEMO/guide/html/configurations.html) and the process will be similar.

Copy the reference configuration you like to use by doing the following.
```shell
cd $DATADIR/NEMO/release-4.0.1/cfgs
mkdir GYRE_testing
rsync -arv GYRE_PISCES/* GYRE_testing
```
Then go into `GYRE_testing`, rename the `cpp_GYRE_PISCES.fcm` file.
```shell
cd GYRE_testing
mv cpp_GYRE_PISCES.fcm cpp_GYRE_testing.fcm
```
Then edit the same file and replace `key_top` with `key_nosignedzero`.
***
In `cfgs/` edit the file `ref_cfgs.txt` and add your configuration to the bottom of the list. For this example the file looks like. 
```shell
AGRIF_DEMO OCE ICE NST
AMM12 OCE
C1D_PAPA OCE
GYRE_BFM OCE TOP
GYRE_PISCES OCE TOP
ORCA2_OFF_PISCES OCE TOP OFF
ORCA2_OFF_TRC OCE TOP OFF
ORCA2_SAS_ICE OCE ICE NST SAS
ORCA2_ICE_PISCES OCE TOP ICE NST
SPITZ12 OCE ICE
GYRE_testing OCE TOP
```
***
**Note:** Make sure the OCE/TOP/... flags match the reference configuration you are using
***
You are now ready to **compile nemo**. Make sure you have all the modules in Step 1 loaded. Go back to the base directory and do the following
```shell
cd $DATADIR/NEMO/release-4.0.1
./makenemo -j2 -r GYRE_testing -m linux_NEXCS
```
If all has gone well you have successfully compiled NEMO!

## Step 5 - Using NEMO on NEXCS
Now that you have compiled NEMO, you probably want to run it.

NEMO runs in parallel and will output a data file for each thread used. Therefore we need the `REBUILD_NEMO` tool to recombine the outputs into a single file.

To do this go to the tools folder and execute the following command:

```shell
cd $DATADIR/NEMO/nemo4.0-9925/tools
./maketools -n REBUILD_NEMO -m linux_NEXCS
```
***
**Note:** If you get an error about `iargc_` and/or `getarg` not being external variables then go to the following file `tools/REBUILD_NEMO/src/rebuild_nemo.F90` and comment out these two lines.
```shell
INTEGER, EXTERNAL :: iargc
 ...
 external :: getarg
```
Depending on the compiler used these may or may not be treated as external variables.
***
Now go back to your configuration and create a symbolic link to `xios_server.exe`
```shell
cd $DATADIR/NEMO/release-4.0.1/cfgs/GYRE_testing/EXP00
ln -s $DATADIR/XIOS/xios-2.5/bin/xios_server.exe .
```
Also modify `iodef.xml` and set `using_server` to `true`
```shell
<variable id="using_server"              type="bool">true</variable>
```
Make an `OUTPUTS/` and `RESTARTS/` directory in `EXP00`
```shell
mkdir OUTPUTS RESTARTS
```
You also need to add a shell script for post processing to `EXP00`. The one I use below is a modified version of [Julian Mak's post processing script](https://nemo-related.readthedocs.io/en/latest/compilation_notes/Oxford_ARC.html). It is too long to show on this page but you can find it [here](https://github.com/afstyles/nemo4.0-NEXCS/blob/master/cfgs/GYRE_testing/EXP00/postprocess.sh) and it should work for this example. The only values that need changing are `NUM_DOM`(how many nodes is NEMO using) and `NUM_CPU` (how many cpus in total).

To run a job on NEXCS you need to use a submission script like the one below. Yet again this is a modification of [Julian's submission script](https://nemo-related.readthedocs.io/en/latest/compilation_notes/Oxford_ARC.html).
```shell
#!/bin/bash
#!
##subm_script.pbs
#PBS -q nexcs
#PBS -A user ##Username
#PBS -l select=3 ##(Number of nodes running NEMO + Number of nodes running XIOS)
#PBS -l walltime=00:10:00 ##Maximum runtime
#PBS -N GYRE_testing ##Name of run in queue
#PBS -o testing.output ##Name of output file
#PBS -e testing.error ##Name of error file
#PBS -j oe
#PBS -V

cd $DATADIR/NEMO/release-4.0.1/cfgs/GYRE_testing/EXP00/

echo " _ __   ___ _ __ ___   ___         "
echo "| '_ \ / _ \ '_ ' _ \ / _ \        "
echo "| | | |  __/ | | | | | (_) |       "
echo "|_| |_|\___|_| |_| |_|\___/  v4.0  "

export OCEANCORES=64 #Number of cores used (32 per node running NEMO) 
export XIOSCORES=2 #Number of cores to run XIOS (=number of nodes running NEMO)

ulimit -c unlimited
ulimit -s unlimited

aprun -b -n $XIOSCORES -N 2 ./xios_server.exe : -n $OCEANCORES -N 32 ./nemo

#===============================================================
# POSTPROCESSING
#===============================================================

# kills the daisy chain if there are errors

if grep -q 'E R R O R' ocean.output ; then

  echo "E R R O R found, exiting..."
  echo "  ___ _ __ _ __ ___  _ __  "
  echo " / _ \ '__| '__/ _ \| '__| "
  echo "|  __/ |  | | | (_) | |    "
  echo " \___|_|  |_|  \___/|_|    "
  echo "check out ocean.output or stdouterr to see what the deal is "

  exit
else
  echo "going into postprocessing stage..."
  # cleans up files, makes restarts, moves files, resubmits this pbs

  bash ./postprocess.sh >& cleanup.log
  exit
fi
```
This script uses two nodes (each with 32 cores) to run NEMO and an additional node to run XIOS.
***
**Note:** For the postprocessing to work  the number of nodes used for NEMO (not XIOS) must = `NUM_DOM` and the number of cores for NEMO = `NUM_CPU`
***
Submit this script by doing
```shell
qsub subm_script.pbs
```
Check the status of the submitted job using `qstat`

If all goes well you should have outputs in the `OUTPUTS/` folder. If not, check `testing.output` and `ocean.output` to see what the issue is.
***
**Note:** Make sure you have the modules mentioned in Step 1 loaded when submitting
***
**Note:** If you have an error referring to `namberg` have a look in `namelist_ref`and look for a comment character `!` right next to a variable.
```shell
.true.!comment
``` 
If found, add a space.
```shell
.true. !comment
```
***

Congratulations, you have successfully compiled and run NEMO!