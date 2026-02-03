# Compiling COAWST on DelftBlue
Instructions to compile [COAWST](https://github.com/DOI-USGS/COAWST) (Coupled Ocean-Atmosphere-Wave-Sediment Transport) on the [DelftBlue](https://doc.dhpc.tudelft.nl/delftblue/) supercomputer.

**Note:** These instructions are tailored for [COAWST v3.8](https://github.com/DOI-USGS/COAWST/releases/tag/COAWST_v3.8) and are meant to complement the official [User Manual](https://github.com/DOI-USGS/COAWST/blob/COAWST_v3.8/COAWST_User_Manual.doc) included in the COAWST repository.

## 1. Download the repository

Start by logging into DelftBlue and cloning the COWAST GitHub repository to your scratch repository:
```bash
cd /scratch/<netid>
git clone https://github.com/DOI-USGS/COAWST.git
cd COAWST
git checkout COAWST_v3.8
```
## 2. Compile MCT

COAWST depends on the MCT (Model Coupling Toolkit) library. To compile MCT, start by loading the compiler, MPI and NETCDF Fortran modules:

```bash
module load 2025
module load openmpi
module load netcdf-fortran
```

Move to the MCT directory and run the configuration script

```bash
cd /scratch/<netid>/COAWST/Lib/MCT
./configure # generates Makefile.conf
```

Edit the newly created `Makefile.conf` to allow Fortran type mismatches by setting the empty `FCFLAGS` as follows:
```makefile
FCFLAGS		=   -fallow-argument-mismatch 
```

Edit the installation paths for the MCT library at the bottom of `Makefile.conf`. You can set them wherever you like, but it is recommended to keep them in your scratch directory, at the same level as COWAST.
```makefile
libdir          = /scratch/<netid>/MCT_install/lib
includedir      = /scratch/<netid>/MCT_install/include
```

Create the directories and subdirectories in the installation paths (`libdir` and `includedir` in `Makefile.conf`).
```
mkdir -p /scratch/<netid>/MCT_install/{lib, include}
```


Compile MCT and install to the specified paths.  
```bash
make
make install
```

You can verify that MCT was installed correctly by opening the installation directories or listing their contents:
```bash
ls /scratch/<netid>/MCT_install/lib
ls /scratch/<netid>/MCT_install/include
```

Finally, set the MCT environment variables specified in the COAWST User Manual. To do so, add the following lines to your `~/.bash_profile`:
```bash
export MCT_INCDIR=/scratch/<netid>/MCT_install/include
export MCT_LIBDIR=/scratch/<netid>/MCT_install/lib
```

Reload the bash profile in you current session to apply the changes immediately:

```bash
source ~/.bash_profile
```

> [!TIP]
> `~/.bash_profile` is a shell configuration file that is executed whenever you start a new login shell. It is used to set environment variables and customize your shell environment. After editing it, always reload the profile to apply the changes in your current session immediately. If you do not reload it, the changes will only take effect in new login shells.

## 3. Compile SCRIP

COAWST relies on SCRIP (Spherical Coordinate Remapping and Interpolation Package) for grid remapping. To compile SCRIP, start by loading the compiler, MPI, and NetCDF modules:

```bash
module load 2025
module load gcc
module load openmpi
module load netcdf-fortran
module load netcdf-cxx
module load netcdf-cc
```

Set the Fortran compiler to `gfortran` in `Lib/SCRIP_COAWST/makefile`, leaving the other compiler options commented-out:

```makefile
FORT = gfortran
```

In `Compilers/Linux-gfortran.mk`, locate the section that defines the path to the NetCDF Fortran interface:

```makefile
ifdef USE_NETCDF4
        NF_CONFIG ?= nf-config
    NETCDF_INCDIR ?= $(shell $(NF_CONFIG) --prefix)/include
             LIBS += $(shell $(NF_CONFIG) --flibs)
           INCDIR += $(NETCDF_INCDIR) $(INCDIR)
else
```

and add the following lines **before** the `else` statement:

```makefile
NC_CONFIG ?= nc-config
NETCDF_C_INCDIR ?= $(shell $(NC_CONFIG) --prefix)/include
LIBS += $(shell $(NC_CONFIG) --libs)
INCDIR += $(NETCDF_C_INCDIR) $(INCDIR)
```

Finally, compile SCRIP

```bash
cd /scratch/<netid>/COAWST/Lib/SCRIP_COAWST
make
```

If the compilation is successful, an executable called `scrip_coawst` will be creted in the directory `Lib/SCRIP_COAWST`. 


## 4. Compile COAWST

With MCT and SCRIP successfully built, we can proceed with the COAWST compilation. COAWST is not a single executable but a framework that synchronizes data exchange between independent numerical models:
- **ROMS**: Regional Ocean Modeling System (Ocean)
- **WRF**: Weather Research and Forecasting (Atmosphere)
- **SWAN**: Simulating WAves Nearshore (Waves)
- **WaveWatch III**: Third-generation wave model (Waves)

Each model is an independent software package that can be run on its own. What COAWST does is facilitate the data exchange between these models using MCT and SCRIP.

<details>

<summary>The roles of MCT and SCRIP</summary>

While each model operates on its own grid, COAWST allows them to "talk" to one another using two specific tools:
- **MCT (Model Coupling Toolkit)**: Acts as the communication engine. It handles the parallel data exchange between the models. When the simulation is running, MCT is responsible for the "handshake," ensuring that variables reach the correct processor at the right time.
- **SCRIP (Spherical Coordinate Remapping and Interpolation Package)**: Acts as the grid translator. Because a point on the WRF grid rarely lines up perfectly with a point on the ROMS grid, SCRIP computes the interpolation weights. It determines exactly how to map data from one coordinate system to another.

**In short**: SCRIP tells the system where the data should go across different grids, and MCT actually delivers it during the simulation.

</details>

COAWST can be built with various combinations of these models. This guide focuses on building the application using **ROMS, WRF, and SWAN**.

### 4.1 Define compilation options

When compiling COAWST you must select the compilation options for each model. These options are specified via C-preprocessor `#define` statements in a `<project>.h` file. For this guide we are using the pre-configured **Sandy** example which makes use of ROMS, WRF and SWAN. The configuration file is located in `Projects/Sandy/sandy.h`.

If you were creating a new project, you would create a new sub-directory in `Projects/` and place your `<project>.h` inside it.

### 4.2 Update build_coawst.sh 

COAWST compilation is managed by the `build_coawst.sh` script located in the root of the repository. This script specifies which compiler and MPI implementation (if any) to use and tells the compiler where to find the source code and which models to activate.

To build a COAWST application that can run the Sandy example, update the following variables in `build_coawst.sh`:

```bash
# Line 133: set COAWST_APPLICATION to SANDY (uppercase)
export   COAWST_APPLICATION=SANDY

# Line 141: set MY_ROOT_DIR to the COAWST repository
export   MY_ROOT_DIR=/scratch/<netid>/COAWST

# Line 209: add # at the beginning of the line to disable the use of Intel MPI
# export         which_MPI=intel         # compile with mpiifort library

# Line 213: remove # from the beginning of the line to enable the use of OpenMPI
export         which_MPI=openmpi       # compile with OpenMPI library

# Lines 306-307: set MY_HEADER_DIR and MY_ANALYTICAL_DIR to the Sandy Project
export     MY_HEADER_DIR=${MY_PROJECT_DIR}/Projects/Sandy
export MY_ANALYTICAL_DIR=${MY_PROJECT_DIR}/Projects/Sandy
```

> [!TIP]
> In bash scripts, adding `#` at the start of a line turns it into a comment, meaning it will be ignored during execution.

### 4.3 Fix bug in SWAN/switch.pl

When compiling COAWST with modern versions of CMake, the build process for the SWAN model may hang indefinitely. This occurs because CMake now passes absolute file paths to the `SWAN/switch.pl` script. The legacy Perl logic incorrectly identifies hyphens within these directory paths (e.g. `/scratch/<netid>/phd-project/COAWST/`) as command-line flags, triggering an infinite loop.

To resolve this issue, locate the following `while` loop in line 18 of `switch.pl`:

```perl
while ( $ARGV[0]=~/-.*/ )
```

and replace it with the following:

```perl
while ( $ARGV[0] =~ /^-/ )
```

The addition of `^` ensures that Perl only treats the argument as a switch if the hyphen is the first character. This allows the script to correctly ignore hyphens that appear later in folder names or file extensions, allowing the loop to terminate and the compilation to proceed.

### 4.4 Load necessary modules

Load the the compiler, MPI, NetCDF, CMake and Perl modules.

```bash
module load 2025
module load gcc
module load openmpi
module load netcdf-fortran
module load netcdf-cxx
module load netcdf-c
module load cmake
module load perl
```

### 4.5 Create NetCDF layout

SCRIP and COAWST rely on the [netcdf](https://www.unidata.ucar.edu/software/netcdf) library, which provides APIs for Fortran, C and C++.

Older NetCDF installations placed all binaries, libraries, and header files for these languages in a single directory. Both SCRIP and COAWST expect this legacy layout when compiling.

Newer NetCDF versions install each language interface separately (e.g. `netcdf-fortran`, `netcdf-c`, `netcdf-cxx`), resulting in three different installation prefixes. This is why three NetCDF modules must be loaded above.

To avoid modifying SCRIP or COAWST build scripts (or using an outdated NetCDF version), we create a unified directory containing symbolic links to the relevant NetCDF files. This mimics the old layout and allows the build systems to work as expected.

> [!TIP]
> A _symbolic link_ (or _symlink_) is a special file that points to another file or directory. It behaves like a shortcut. When you access the symlink, the system redirects you to the target file. The standard way to create one is with `ln -s [target] [link]`.

1. Create a directory for NetCDF symlinks. We recommend creating a `utilities` directory at the root of the COAWST repository. This directory will act as a “fake” NetCDF installation prefix.

```bash
mkdir -p /scratch/<netid>/COAWST/utilities/{bin, lib, include}
```

2. Locate NetCDF installation paths. Use the configuration tools provided by NetCDF to locate the installation directories for the Fortran and C interfaces:

```bash
nf-config --prefix 
# Example output:
# /apps/arch/2025/software/linux-rhel8-cascadelake/gcc-13.3.0/netcdf-fortran-4.6.1-n2z33vpkn3jcrjacyrtudhif3bt7rxvd

nc-config --prefix 
# Example output:
# /apps/arch/2025/software/linux-rhel8-cascadelake/gcc-13.3.0/netcdf-c-4.9.2-eazho6ml6uxvu4qea4tnjjjzkjd2mtre
```

3. Create the symbolic links.

```bash
# Navigate to the new utilities folder
cd /scratch/<netid>/COAWST/utilities

# Create symbolic links for Fortran.
# Replace the netcdf-fortran path for the one returned by nf-config --prefix (step 2)
ln -sf /apps/arch/2025/software/linux-rhel8-cascadelake/gcc-13.3.0/netcdf-fortran-4.6.1-n2z33vpkn3jcrjacyrtudhif3bt7rxvd/bin/* bin/
ln -sf /apps/arch/2025/software/linux-rhel8-cascadelake/gcc-13.3.0/netcdf-fortran-4.6.1-n2z33vpkn3jcrjacyrtudhif3bt7rxvd/lib/* lib/
ln -sf /apps/arch/2025/software/linux-rhel8-cascadelake/gcc-13.3.0/netcdf-fortran-4.6.1-n2z33vpkn3jcrjacyrtudhif3bt7rxvd/include/* include/

# Create symbolic links for C.
# Replace the netcdf-c path for the one returned by nc-config --prefix (step 2)
ln -sf /apps/arch/2025/software/linux-rhel8-cascadelake/gcc-13.3.0/netcdf-c-4.9.2-eazho6ml6uxvu4qea4tnjjjzkjd2mtre/bin/* bin/
ln -sf /apps/arch/2025/software/linux-rhel8-cascadelake/gcc-13.3.0/netcdf-c-4.9.2-eazho6ml6uxvu4qea4tnjjjzkjd2mtre/lib/* lib/
ln -sf /apps/arch/2025/software/linux-rhel8-cascadelake/gcc-13.3.0/netcdf-c-4.9.2-eazho6ml6uxvu4qea4tnjjjzkjd2mtre/include/* include/
```

4. Set environment variables NETCDF and NETCDF_classic.

```bash
export NETCDF=/scratch/<netid>/COAWST/utilities
export NETCDF_classic=1
```

### 4.6 Compile COAWST

With all environment variables set and dependencies linked, you are ready to compile the main COAWST executable. This is done by running the `build_coawst.sh` script located in the root directory.

Because the Sandy example couples ROMS, WRF, and SWAN, compilation can take a very long time, often over an hour. To speed this up, it is highly recommended to use parallel compilation by passing the `-j` flag followed by the number of threads you wish to use.

```bash
cd /scratch/<netid>/COAWST
./build_coawst.sh -j 8
```

If the compilation is successful, an executable called `coawstM` should be created in the root of the repository.

> [!IMPORTANT]
> Compiling on a login node with many threads can impact performance for other users. If you are using a high number of threads (e.g., -j 16 or more), consider running the compilation within an interactive `srun` session or as a batch job.
