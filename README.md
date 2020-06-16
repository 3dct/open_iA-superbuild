## open\_iA Superbuild

This document provides the currently simplest way of setting up a build environment for [open\_iA](https://github.com/3dct/open_iA). The build steps here were tested on Ubuntu 18.04.1 and 18.10, as well as on Windows 10. They might or might not work on other distributions and operating systems. Please let us know of your experience [in the issue tracker](https://github.com/3dct/open_iA-superbuild/issues). In case of problems, make sure to check the troubleshooting section below.

## Build steps

In short, these are the steps required to set up a build environment for open\_iA:
1. Install prerequisites - git, CMake, and C++ build toolchain, as well as libraries and headers for OpenCL, OpenGL and Qt
2. Clone  this repository to some folder on your machine
3. Run CMake on the checked out repository for the desired toolchain
4. Build with the toolchain configured in CMake
5. Done!

Detailed descriptions of these steps are given below.
Additional information can also be found in the [Windows build instructions](https://github.com/3dct/open_iA/wiki/Windows-Build) and the [Linux build instructions](https://github.com/3dct/open_iA/wiki/Linux-Build) in the open\_iA wiki.

### 1. Install Prerequisites
First, you need to install the following applications:
- CMake
- Git
- A build toolchain (compiler + build tool, e.g. make + g++ / ninja + clang / Visual Studio)
open\_iA also requires some headers and libraries:
- OpenGL
- optional: OpenCL
- Qt

On Linux, those packages should be available from the distribution repository. For example, on Ubuntu, execute:
```
  $ sudo apt install cmake g++ make git \
    libgl1-mesa-dev libxt-dev \
    ocl-icd-opencl-dev opencl-headers opencl-clhpp-headers \
    qtbase5-dev libqt5x11extras5-dev qttools5-dev qt5-default \
    libxt-dev libqt5opengl5-dev libqt5svg5-dev libqt5charts5-dev
```

On Windows, download and install:
- [CMake](https://cmake.org/)
- [git](https://git-scm.com/download/win)
- Visual Studio (e.g. the free [Community Edition](https://visualstudio.microsoft.com/de/vs/older-downloads) of VS 2015 or newer)
- OpenGL headers are included in the Windows SDK installed along with Visual Studio
- optional: an OpenCL SDK; if available, some GPU operations will be available; some modules require it. The generic OpenCL-ICD loader can be built from within the superbuild (via ENABLE_OPENCL). Or you can use an SDK best fitting your build system, e.g. the AMD OpenCL SDK for an AMD graphics card, the NVidia CUDA SDK for an NVidia graphics card, or the Intel OpenCL SDK for an onboard graphics card (but note that this ties your build to machines having similar compute capabilities)
- Open Source version of Qt 5 >= 5.9 (install 64 bit binaries for the respective Visual Studio version, recommended is e.g. 5.11.x or 5.12.x) 

### 2. Clone Repository

Through git executed on the shell:
- Open a shell in the folder where you want to place open\_iA.
- Clone this repository: `$ git clone https://github.com/3dct/open_iA-superbuild.git`

Alternatively, use a git GUI tool of your choice.

**Note:** On Windows, take care that the full path string of the folder where you place open\_iA is very short; ideally something like `C:\open_iA`. The exact possible maximum length is currently untested, but you should be on the safe side with paths shorter than 20 characters. For example `C:\open_iA\build` (16 characters) should be fine, while `C:\open_iA\superbuild-bin` (25 characters) is definitely too long (for someone interested in technical details: the path will be incorporated into command lines created by CMake; and on Windows, there is a limit of 32.768 characters on the length of a command; and in some created command lines, the chosen path appears many times).

### 3. CMake Configure+Generate

Run CMake; you have two options:
1. If you prefer the command line, open a shell in the cloned repository, and execute `$ cmake .`. In case of errors, we recommend to switch to option 2.
2. Run the graphical user interface version of CMake:
  - Enter the cloned repository folder under "Where is the source code"
  - Enter the folder you wish that vtk, itk and open\_iA sources and binaries shall be put in, under "Where to build the binaries".
  - Press "Configure"
  - When asked for a build environment:
    - On Linux, use the usually pre-selected "Unix Makefiles"
    - On Windows, use the 64 bit version of the respective Visual Studio toolchain, e.g. "Visual Studio 14 2015 Win64" for Visual Studio 2015 ("Visual Studio 14 2015" as generator and "x64" as platform on CMake versions >= 3.14)
  - If Qt is installed in a non-standard folder, you will see a corresponding error. You will have to adapt the Qt5\_DIR to point to the "lib/cmake/Qt5" subdirectory of the respective version/build directdory in the Qt installation folder (e.g. 5.11.2/msvc2015\_64).

### 4. Build Everything

Build everything. The specific steps required here differ for the build environment you configured in CMake:
- On Linux, enter the folder configured in CMake as "Where to build the binaries" in a shell, then run: `$ make -j 4`
- Under Windows, open the generated "open\_iA-superbuild.sln in Visual Studio, then build the solution.

### 5. Done!

When the build finished successfully, open\_iA executables will be built in the `open_iA/bin/bin` subfolder of the folder configured in CMake as "Where to build the binaries".

## Troubleshooting / Adaptations:

- The `$` sign in the code above indicates the shell prompt, do not enter this sign.

- For the prerequisites installation on Linux: the command to install packages as well as the names of the packages will differ on other distributions other than Ubuntu, please consult the manual or community of your distribution.

- The method described above sets up a basic build environment.
  - It will only build VTK and ITK libraries.
  - It builds the core executables as well as some of the advanced analysis modules.
  - If you require more modules or other library support, e.g. Eigen, OpenCL or HDF5 support, see the ENABLE_... flags
  - Support for the Astra reconstruction module is implemented (ENABLE_ASTRA), but still very much experimental, it works only under very specific circumstances.

- The superbuild is intended for setting up a build environment. For development of open_iA modules, it is recommended to switch to using the `open_iA/bin` subfolder of the superbuild binary folder as build / CMake base once the first superbuild has finished successfully.

- If CMake encounters an error, it probably will tell you that some library is missing which you still need to install.

- If you have more than 4 cores in your system, you will probably want to increase the number of parallel builds if you use the mmake system (is set to 4 above) To e.g. run a make build with 8 parallel threads, run
  `$ make -j 8`

- If taking the first option under "CMake Configure+Generate", and you want to put the build environment for open\_iA somewhere else as where you checked out the superbuild,
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
