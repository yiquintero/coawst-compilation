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

```bash
# Load necessary modules
module load 2025
module load openmpi
module load netcdf-fortran

# Run the configuration script for MCT
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

Compile MCT and install to the paths specified in `Makefile.conf`.  
```bash
make
make install
```
> [!WARNING]
>  Make sure all directories and subdirectories in the installation paths (libdir and includedir in Makefile.conf) exist before running `make install`. 

You can verify that MCT was installed correctly by opening the installation directories or listing their contents:
```bash
ls /scratch/<netid>/MCT_install/lib
ls /scratch/<netid>/MCT_install/include
```

Finally, set the MCT environment variables specified in the COAWST User Manual. To do so, add the following lines to your `~/.bash_profile`:
```bash
setenv   MCT_INCDIR   /usr/include
setenv   MCT_LIBDIR   /usr/lib
```

Reload the bash profile in you current session to apply the changes immediately:

```bash
source ~/.bash_profile
```

> [!NOTE]
> `~/.bash_profile` is a shell configuration file that is executed whenever you start a new login shell. It is used to set environment variables and customize your shell environment. After editing it, always reload the profile to apply the changes in your current session immediately. If you do not reload it, the changes will only take effect in new login shells.
