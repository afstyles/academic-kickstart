---
title: Compiling NEMO 4.0 on NEXCS
summary: A guide to compiling NEMO 4.0 on NEXCS.
date: "2020-05-28T00:00:00Z"

reading_time: false  # Show estimated reading time?
share: true  # Show social sharing links?
profile: true  # Show author profile?
comments: false  # Show comments?

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
---

 
These are my personal notes on how to set up and compile NEMO 4.0 on NEXCS.

NEXCS is the NERC partition of the Met Office Cray XC40 supercomputer and has a similar infrastructure to Monsoon2.

When compiling NEMO I could not find any guide for this particular system but used the following excellent guides for inspiration:

* [Julian Mak's](https://nemo-related.readthedocs.io/en/latest/compilation_notes/nemo40.html) guide to compiling NEMO 4.0 on local machines and Oxford ARC.

* [Christopher Bull's](http://christopherbull.com.au/nemo/nemo-wed12-01/) guide for compiling NEMO on ARCHER

As these guides were so helpful, I feel it is only fair to make a similar contribution.

## Step 1 - Loading modules

Once you are logged on to the Met Office lander, connect to NEXCS using
```shell
ssh xcs-c
```
On logging in you need to load the correct compilers and libraries. You will not be able to compile XIOS or NEMO without loading the correct modules. First load the netcdf and hdf5 libraries.
```shell
module load cray-netcdf-hdf5parallel/4.4.1
module load cray-hdf5-parallel/1.10.0
```
Switch to a more recent version of mpich.
```shell
module swap cray-mpich/7.0.4 cray-mpich/7.3.1
```
Finally switch from the Cray environment to the GNU environment
```shell
module swap PrgEnv-cray/5.2.82 PrgEnv-gnu/5.2.82
```
It will also be useful to load an editor
```shell
module load nano
```
***
**Note:** When you log in to NEXCS a set of essential modules are loaded by default. Do not use `module purge` before loading the above modules. 
***
You loaded modules should look like the following:
```shell
user@xcslc0:~> module list
  1) moose-client-wrapper                   9) craype-network-aries                  17) cray-hdf5-parallel/1.10.0             25) dvs/2.5_0.9.0-1.0502.2188.1.116.ari
  2) python/v2.7.9                         10) gcc/4.9.1                             18) cray-libsci/13.0.1                    26) alps/5.2.4-2.0502.9774.31.11.ari
  3) metoffice/tempdir                     11) craype/2.2.1                          19) udreg/2.3.2-1.0502.10518.2.17.ari     27) rca/1.0.0-2.0502.60530.1.62.ari
  4) metoffice/userenv                     12) craype-haswell                        20) ugni/6.0-1.0502.10863.8.29.ari        28) atp/1.7.5
  5) subversion-1.8/1.8.19                 13) cray-mpich/7.3.1                      21) pmi/5.0.5-1.0000.10300.134.8.ari      29) PrgEnv-gnu/5.2.82
  6) modules/3.2.10.5                      14) pbs/18.2.5.20190913061152             22) dmapp/7.0.1-1.0502.11080.8.76.ari
  7) eswrap/1.3.3-1.020200.1280.0          15) nano/2.8.6                            23) gni-headers/4.0-1.0502.10859.7.8.ari
  8) switch/1.0-1.0502.60522.1.61.ari      16) cray-netcdf-hdf5parallel/4.4.1        24) xpmem/0.1-2.0502.64982.5.3.ari
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
cp arch-GCC_LINUX.env arch-GCC_NEXCS.env
cp arch-GCC_LINUX.fcm arch-GCC_NEXCS.fcm 
cp arch-GCC_LINUX.path arch-GCC_NEXCS.path
```
Then edit the files so they look like the following
```shell
#arch-GCC_NEXCS.env
export HDF5_INC_DIR=/opt/cray/hdf5/1.10.0/GNU/4.9/include
export HDF5_LIB_DIR=/opt/cray/hdf5/1.10.0/GNU/4.9/lib

export NETCDF_INC_DIR=/opt/cray/netcdf/4.4.1/GNU/4.9/include
export NETCDF_LIB_DIR=/opt/cray/netcdf/4.4.1/GNU/4.9/lib
```
***
**Note:** If you are using a different set of modules or worried that this path is wrong. The commands `which nc-config` and `which h5copy` will give you a path of the form `directory/bin`.  The paths used above should then be `directory/GNU/4.9/include` and similar for `lib`
***
```shell
#arch-GCC_NEXCS.fcm
################################################################################
################### Projet XIOS ###################
################################################################################

%CCOMPILER cc
%FCOMPILER ftn
%LINKER CC

%BASE_CFLAGS -ansi -w
%PROD_CFLAGS -O3 -DBOOST_DISABLE_ASSERTS
%DEV_CFLAGS -g -O2
%DEBUG_CFLAGS -g

%BASE_FFLAGS -D__NONE__
%PROD_FFLAGS -O3
%DEV_FFLAGS -g -O2
%DEBUG_FFLAGS -g

%BASE_INC -D__NONE__
%BASE_LD -lstdc++

%CPP cpp
%FPP cpp -P
%MAKE gmake
```
***
```shell
#arch-GCC_NEXCS.path
NETCDF_INCDIR="-I $NETCDF_INC_DIR"
NETCDF_LIBDIR="-Wl,'--allow-multiple-definition' -L$NETCDF_LIB_DIR"
NETCDF_LIB="-lnetcdff -lnetcdf"

MPI_INCDIR=""
MPI_LIBDIR=""
MPI_LIB=""

HDF5_INCDIR="-I $HDF5_INC_DIR"
HDF5_LIBDIR="-L $HDF5_LIB_DIR"
HDF5_LIB="-lhdf5_hl -lhdf5 -lhdf5 -lz"
```
***
Then you need to edit `bld.cfg` in `xios-2.5/`
```shell
cd $DATADIR/XIOS/xios-2.5
```
and change all occurrences of `src_netcdf` to `src_netcdf4` (two in total)

You are now ready to compile XIOS. In `xios-2.5/`
```shell
./make_xios --full --prod --arch GCC_NEXCS -j2
```
If this build is successful then XIOS is ready to use. If not,  check over the arch files and make sure the flags are correct and in the right order. Also make sure you have the correct modules loaded. Getting XIOS to compile is by far the most difficult part of the process.

## Step 3  - NEMO 4.0
Now we have XIOS ready to go we can start working on NEMO 4.0.

Go back to the data directory  and create a new folder for NEMO
```shell
cd $DATADIR
mkdir NEMO
```
Then download NEMO 4.0 in the folder.
```shell
cd NEMO
svn checkout -r 9925 http://forge.ipsl.jussieu.fr/nemo/svn/NEMO/trunk nemo4.0-9925
```
Now we need to copy and update the arch files for NEMO like we did in XIOS.
```shell
cd nemo4.0-9925/arch/
cp arch-linux_ifort.fcm arch-linux_NEXCS.fcm
```
The file `arch-linux_NEXCS.fcm` needs to look like
```shell
#arch-linux_NEXCS.fcm

%NCDF_HOME           /opt/cray/netcdf/4.4.1/GNU/4.9
%HDF5_HOME           /opt/cray/hdf5/1.10.0/GNU/4.9
%XIOS_HOME           /projects/nexcs-n02/user/XIOS/xios-2.5

%NCDF_INC            -I%NCDF_HOME/include -I%HDF5_HOME/include
%NCDF_LIB            -L%NCDF_HOME/lib -lnetcdf -lnetcdff -lstdc++
%XIOS_INC            -I%XIOS_HOME/inc
%XIOS_LIB            -L%XIOS_HOME/lib -lxios -lstdc++

%CPP                 cpp
%FC                  ftn
%FCFLAGS             -fdefault-real-8 -O3 -funroll-all-loops -fcray-pointer -cpp -ffree-line-length-none
%FFLAGS              %FCFLAGS
%LD                  %FC                                                                                                                                                                               %FPPFLAGS            -P -C -traditional
%LDFLAGS
%AR                  ar
%ARFLAGS             -rs
%MK                  make
%USER_INC            %XIOS_INC %NCDF_INC
%USER_LIB            %XIOS_LIB %NCDF_LIB

%CC                  cc
```
## Step 4 - Compiling the GYRE_PISCES configuration
Now we need a configuration to compile. In this example I use GYRE_PISCES which is a reference configuration in NEMO as it is a relatively simple configuration. Other reference configurations are listed [here](https://forge.ipsl.jussieu.fr/nemo/chrome/site/doc/NEMO/guide/html/configurations.html) and the process will be similar.

Copy the reference configuration you like to use by doing the following.
```shell
cd $DATADIR/NEMO/nemo4.0-9925/cfgs
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
cd $DATADIR/NEMO/nemo4.0-9925
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
**Note:** If you get an error about `iargc_` and/or `getarg` not being external variables then go to the following file `tools/REBUILD_NEMO/src/rebuild_nemo.F90` and comment out these two lines
```shell
INTEGER, EXTERNAL :: iargc
 ...
 external :: getarg
```
These are not external variables in Fortran 90.
***
Now go back to your configuration and create a symbolic link to `xios_server.exe`
```shell
cd $DATADIR/NEMO/nemo4.0-9925/cfgs/GYRE_testing/EXP00
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

cd $DATADIR/NEMO/nemo4.0-9925/cfgs/GYRE_testing/EXP00/

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