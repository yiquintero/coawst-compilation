# Compiling COAWST on DelftBlue
Instructions to compile [COAWST](https://github.com/DOI-USGS/COAWST) on the [DelftBlue](https://doc.dhpc.tudelft.nl/delftblue/) supercomputer.

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

COAWST depends on the Model Coupling Toolkit (MCT) library. To compile MCT, start by loading the compiler, MPI and NETCDF Fortran modules:

```bash
module load 2025
module load openmpi
module load netcdf-fortran
```

Move to the MCT directory and run the configuration script

```bash
cd Lib/MCT
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

COAWST relies on SCRIP for grid remapping. To compile SCRIP, start by loading the compiler, MPI, and NetCDF modules:

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
cd Lib/SCRIP_COAWST
make
```

If the compilation is successful, an executable called `scrip_coawst` will be creted in the directory `Lib/SCRIP_COAWST`. 


## Compile COAWST

**NetCDF layout**

SCRIP and COAWST rely on the [netcdf](https://www.unidata.ucar.edu/software/netcdf) library, which provides APIs for Fortran, C and C++.

Older NetCDF installations placed all binaries, libraries, and header files for these languages in a single directory. Both SCRIP and COAWST expect this legacy layout when compiling.

Newer NetCDF versions install each language interface separately (e.g. `netcdf-fortran`, `netcdf-c`, `netcdf-cxx`), resulting in three different installation prefixes. This is why three NetCDF modules must be loaded above.

To avoid modifying SCRIP or COAWST build scripts (or using an outdated NetCDF version), we create a unified directory containing symbolic links to the relevant NetCDF files. This mimics the old layout and allows the build systems to work as expected.

> [!TIP]
> A _symbolic link_ (or _symlink_) is a special file that points to another file or directory. It behaves like a shortcut. When you access the symlink, the system redirects you to the target file. The standard way to create one is with `ln -s [target] [link]`.

1. Create a directory for NetCDF symlinks. We recommend creating a `utilities` directory at the root of the COAWST repository. This directory will act as a “fake” NetCDF installation prefix.

```bash
mkdir -p utilities/{bin, lib, include}
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
ln -sf /apps/arch/2023r1/software/linux-rhel8-skylake_avx512/gcc-8.5.0/netcdf-fortran-4.6.0-7ets55p5c7nuask3ah6ejyuvdqq6canp/lib/* lib/

ln -sf /apps/arch/2023r1/software/linux-rhel8-skylake_avx512/gcc-8.5.0/netcdf-c-4.9.0-di5a6gyhmgbmapai34ran7zzco5jjj2j/lib/* lib/

ln -sf /apps/arch/2023r1/software/linux-rhel8-skylake_avx512/gcc-8.5.0/netcdf-c-4.9.0-di5a6gyhmgbmapai34ran7zzco5jjj2j/include/* include/

ln -sf /apps/arch/2023r1/software/linux-rhel8-skylake_avx512/gcc-8.5.0/netcdf-c-4.9.0-di5a6gyhmgbmapai34ran7zzco5jjj2j/bin/* bin/

ln -sf /apps/arch/2023r1/software/linux-rhel8-skylake_avx512/gcc-8.5.0/netcdf-fortran-4.6.0-7ets55p5c7nuask3ah6ejyuvdqq6canp/include/* include/

ln -sf /apps/arch/2023r1/software/linux-rhel8-skylake_avx512/gcc-8.5.0/netcdf-fortran-4.6.0-7ets55p5c7nuask3ah6ejyuvdqq6canp/bin/* bin/
```

