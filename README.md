## open\_iA Superbuild

This document provides the currently simplest way of setting up a build environment for [open\_iA](https://github.com/3dct/open_iA). The build steps here were tested on Ubuntu 18.04.1 and 18.10, as well as on Windows 10. They might or might not work on other distributions and operating systems. Please let us know of your experience [in the issue tracker](https://github.com/3dct/open_iA-superbuild/issues). In case of problems, make sure to check the troubleshooting section below.

## Build steps

In short, these are the steps required to build open\_iA and the required vtk and itk libraries:
1. Install prerequisites - CMake, g++, build tool (e.g. make, ninja or Visual Studo), OpenCL, OpenGL and Qt libraries and headers
2. Clone  this repository to some folder on your machine
3. Run CMake on the checked out repository
4. Build with the build tool selected in CMake

If you are unclear about any of these setps, please see the more detailed descriptions below. Additional information can also be found in the [Windows build instructions](https://github.com/3dct/open_iA/wiki/Windows-Build) and the [Linux build instructions](https://github.com/3dct/open_iA/wiki/Linux-Build) in the open\_iA wiki.

### 1. Install Prerequisites
First, you need to install the following applications:
- CMake
- Git
- A build toolchain (compiler + build tool, e.g. make + g++ / ninja + clang / Visual Studio)
open\_iA also requires some headers and libraries:
- OpenGL
- OpenCL
- Qt

On Linux, those packages should be available from the distribution repository. For example, on Ubuntu, execute:
```
  $ sudo apt install cmake g++ make git \
    libgl1-mesa-dev libxt-dev \
    ocl-icd-opencl-dev opencl-headers opencl-clhpp-headers \
    qtbase5-dev libqt5x11extras5-dev qttools5-dev qt5-default \
    libxt-dev libqt5opengl5-dev
```

On Windows, download and install:
- [git](https://git-scm.com/download/win)
- [CMake](https://cmake.org/)
- Visual Studio 2015 (e.g. the free [Community Edition](https://visualstudio.microsoft.com/de/vs/older-downloads)
- Open GL headers are included in the Windows SDK installed along with Visual Studio
- An OpenCL SDK best fitting your build system, e.g. the AMD OpenCL SDK for an AMD graphics card, or the Intel OpenCL SDK
- Open Source version of Qt 5 >= 5.9 (install 64 bit binaries for Visual Studio 2015, recommended is e.g. 5.11.x or 5.12.x) 

### 2. Clone Repository

Through git executed on the shell:
- Open a shell in the folder where you want to place open\_iA. Preferrably, use a folder such that the overall path string is short.
- Clone this repository: `$ git clone https://github.com/3dct/open_iA-superbuild.git`

Alternatively, use a git GUI tool of your choice.

### 3. CMake Configuration

Run CMake; you have two options:
1. If you prefer the command line, open a shell in the cloned repository, and execute `$ cmake .`. In case of errors, we recommend to switch to option 2.
2. run the graphical user interface version of CMake:
  - Enter the cloned repository folder under "Where is the source code"
  - Enter the folder you wish that vtk, itk and open\_iA sources and binaries shall be put in, under "Where to build the binaries".
  - Press "Configure"
  - If Qt is installed in a non-standard folder, you will see a corresponding error. You will have to adapt the Qt5\_DIR to point to the "lib/cmake/Qt5" subdirectory of the respective version/build directdory in the Qt installation folder (e.g. 5.11.2/msvc2015\_64).

### 4. Build Everything

Build everything. The specific steps required here differ for the build environment you configured in CMake.
- On Linux with the "Unix Makefiles" toolchain for example, enter the folder configured in CMake as "Where to build the binaries" in a shell, then run: `$ make -j 4`
- Under Windows with "Visual Studio 14 2015 Win64", open the generated "open\_iA-superbuild.sln, then build the solution.

### 5. Done!

When the build finished successfully, open\_iA executables will be built in the `open_iA/bin/bin` subfolder of the folder configured in CMake as "Where to build the binaries".

## Troubleshooting / Adaptations:

- The `$` sign in the code above indicates the shell prompt, do not enter this sign.

-The command to install packages as well as the names of the packages to install will differ on other Linux distributions, please consult the manual or community of your distribution.

- The method described here currently only sets up VTK and ITK libraries
  - It will only enable the build of the core executables, but none of the advanced analysis modules. To enable these, after the build has finished, you will have to open another CMake, switch to the `open_iA/bin` binaries folder and enable the required modules.
  -  If you require a module with more elaborate prerequisites, such as the Astra reconstruction module or the HDF5 integration, you have to continue on the [detailed build instructions page](https://github.com/3dct/open_iA/wiki/Linux-Build) in the section of the respective library after setup here is finished.

- If CMake encounters an error, it probably will tell you that some library is missing which you still need to install.

- If you have more than 4 cores in your system, you will probably want to increase the number of parallel builds if you use the mmake system (is set to 4 above) To e.g. run a make build with 8 parallel threads, run
  `$ make -j 8`

- If you want to put the build environment for open\_iA somewhere else as where you checked out the superbuild,
  or if you want to keep the root folder of the superbuild clear (i.e. if you want to do a so-called out-of-source build),
  you can also call CMake from another folder; the relative path given as parameter just has to reference the superbuild directory.

  For example:
  If you checked out this repository to `/home/youruser/open_iA-superbuild`,
  and want to create the build environment, in `/home/youruser/open_iA_Env`, then, instead of step 5 above, execute:
  ```
  $ mkdir /home/youruser/open_iA_Env
  $ cd /home/youruser/open_iA_Env
  $ cmake /home/youruser/open_iA-superbuild
  ```

- If you encounter any error not mentioned here, please [post an issue](https://github.com/3dct/open_iA-superbuild/issues)!
