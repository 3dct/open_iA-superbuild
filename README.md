## open\_iA Superbuild

This document provides the currently simplest way of setting up a build environment for [open\_iA](https://github.com/3dct/open_iA). The build steps here were tested on Ubuntu 18.04.1 and 18.10, as well as on Windows 10. They might or might not work on other distributions and operating systems. Please let us know of your experience [in the issue tracker](https://github.com/3dct/open_iA-superbuild/issues). In case of problems, make sure to check the troubleshooting section below.

## Build steps

In short, these are the steps required to set up a build environment for open\_iA:
1. Install prerequisites - git, CMake, and C++ build toolchain, as well as libraries and headers for OpenGL and Qt (and optionally OpenCL)
2. Clone  this repository to some folder on your machine
3. Run CMake on the checked out repository for the desired toolchain
4. Build with the toolchain configured in CMake
5. Done! For developing with open\_iA, go to the `open_iA` subfolder of the superbuild binary folder: the source repository is in `src`, the build directory in `bin`!

Detailed descriptions of these steps are given below.
Additional information can also be found in the [Windows build instructions](https://github.com/3dct/open_iA/wiki/Windows-Build) and the [Linux build instructions](https://github.com/3dct/open_iA/wiki/Linux-Build) in the open\_iA wiki.

### 1. Install Prerequisites
First, you need to install the following applications:
- CMake (>= 3.14.0)
- Git
- A build toolchain (build tool and compiler, e.g. make and g++ / clang on Linux; or Visual Studio on Windows)

open\_iA also requires some headers and libraries:
- OpenGL
- optional: OpenCL
- Qt

On Linux, those packages should be available from the distribution repository. For example, on Ubuntu, execute:
```
  $ sudo apt install cmake cmake-qt-gui g++ make git \
    libgl1-mesa-dev libxt-dev \
    ocl-icd-opencl-dev opencl-headers opencl-clhpp-headers \
    qtbase5-dev libqt5x11extras5-dev qttools5-dev \
    libxt-dev libqt5opengl5-dev libqt5svg5-dev libqt5charts5-dev \
    libxcursor-dev libsdl2-dev
```
In case you want to use clang instead of gcc, you can remove `g++` from the above command line, and additionally execute:
```
  $ sudo apt install clang libomp-dev
```
(OpenMP is required, and installs as separate package for clang). On versions of Ubuntu prior to 21.04, you might have to also install the `qt5-default` package in order to make Qt 5 the default.

On Windows, download and install:
- [CMake](https://cmake.org/)
- [git](https://git-scm.com/download/win)
- Visual Studio (e.g. the free [Community Edition](https://visualstudio.microsoft.com/de/vs/older-downloads)). We currently **recommend to use Visual Studio 2019**; Visual Studio 2022 should also work, but could cause problems when using anything CUDA-related.
- OpenGL headers are included in the Windows SDK installed along with Visual Studio
- Qt:
	- Go to the [Qt for Open Source Development](https://www.qt.io/download-open-source) page.
	- Under "Looking for Qt binaries?", you should see a button to download the "Qt Online Installer" (if it doesn't show up, try another browser).
	- In the installer, log in to your Qt account (or create one if you haven't got one yet)
    - In the "Select Components" step:
		- expand the "Qt" tree item
		- decide on a version
			- current recommendation is 5.15.2 at the moment; any version >= 5.9.9 should work
			- **Note on Qt versions >= 6**: These are supported, but require to use VTK 9.1 or later, or git from its master branch, see `VTK_VERSION`, `VTK_USE_GIT_REPO` and `VTK_GIT_TAG` options below.
			- **Note on Visual Studio 2022:** pre-built binaries are only provided for VS 2019, but they seem to work on VS 2022 as well.
		- for that version, expand the tree and **only check** the "MSVC 2019 64-bit" option, as well as the "Qt Charts" option under "Additional Libraries"; make sure no other version or option than those two is checked (unless you require them for something else than open\_iA, of course).
- optional: an OpenCL SDK; if available, some GPU operations will be available; some modules require it. The generic OpenCL-ICD loader can be built from within the superbuild (via `ENABLE_OPENCL`). Or you can use an SDK best fitting your build system, e.g. the AMD OpenCL SDK for an AMD graphics card, the NVidia CUDA SDK for an NVidia graphics card, or the Intel OpenCL SDK for an onboard graphics card (but note that this ties your build to machines having similar compute capabilities)


### 2. Clone Repository

- Open a shell (terminal) in the folder where you want to place the open\_iA superbuild.
- Clone the superbuild repository: `$ git clone https://github.com/3dct/open_iA-superbuild.git`

Alternatively, use a git GUI tool of your choice for checking out the superbuild repository.

**Note:** On Windows, take care that the full path string of the folder where you place open\_iA is very short; ideally something like `C:\open_iA`. The exact possible maximum length is currently untested, but you should be on the safe side with paths shorter than 20 characters. For example `C:\open_iA\build` (16 characters) should work, while `C:\open_iA\superbuild-bin` (25 characters) is definitely too long (for someone interested in technical details: the path will be incorporated into command lines created by CMake; and on Windows, there is a limit of 32.768 characters on the length of a command; and in some created command lines, the chosen path appears many times).

### 3. CMake Configure+Generate

Run CMake; you have two options:
1. If you prefer the command line, open a shell inside the cloned repository, and execute `$ cmake .`. In case of errors, we recommend to switch to option 2.
2. Run the graphical user interface version of CMake:
  - Enter the cloned repository folder under "Where is the source code"
  - Enter the folder you wish that vtk, itk and open\_iA sources and binaries shall be put in, under "Where to build the binaries".
  - If you have a folder where you are caching or want to cache downloaded archives of any of the libraries built by the superbuild, adapt `ARCHIVE_DIR`. This can save download bandwidth, as existing source code archives are taken from there and not downloaded again (default: /archives subfolder of your binary folder)
  - Press "Configure"
  - When asked for a build environment:
    - On Linux, use the usually pre-selected "Unix Makefiles"
    - On Windows, use the 64 bit version of the respective Visual Studio toolchain, e.g. "Visual Studio 16 2019 Win64" for Visual Studio 2019 ("Visual Studio 16 2019" as generator and "x64" as platform on CMake versions >= 3.14)
  - If Qt is installed in a non-standard folder, you will see a corresponding error. You will have to adapt the `QT_DIR` to point to the "lib/cmake/Qt5" (/"lib/cmake/Qt6") subdirectory of the respective version/build directory in the Qt installation folder (e.g. `5.15.2/msvc2019_64/lib/cmake/Qt5`).
  - In case you want to change some default options (for example to enable more than the default set of modules), check [Configuration Options](#configuration-options).

### 4. Build Everything

Build everything. The specific steps required here differ for the build environment you configured in CMake:
- With a "Unix Makefiles" generator on Linux for example, enter the folder configured in CMake as "Where to build the binaries" in a shell, then run: `$ make -j 4`
- Under Windows with "Visual Studio ..." generators, open the generated `open_iA-superbuild.sln` in Visual Studio, then build the solution.

### 5. Done!

When the build finished successfully, open\_iA executables will be built in the `open_iA/bin/bin` subfolder of the folder configured in CMake as "Where to build the binaries".

**Note:** The superbuild is intended for setting up a build environment with all dependencies.
For development in open\_iA or its modules, you therefore subsequently don't need to run the full superbuild anymore.
Once the first superbuild has finished successfully, to start developing in open\_iA, navigate to the `open_iA` subfolder.
In its `src` subfolder, you will find a git repository set up for you with all of open\_iA's sources.
Use the `bin` subfolder as basis for building and for CMake Configure/Generate runs.

You only need to go back to executing a build on the whole superbuild folder if you need to change something in one of the libraries that open\_iA depends on.


## Configuration Options


### Enabling specific features/modules

- `ENABLE_AI` - Whether to build AI module; requires ONNX runtime, which will be fetched automatically; on Windows, you can change whether CUDA or DirectML backend is chosen with the option `AI_ONNX_USE_CUDA` (default: disabled) **Note:** Make sure to install a fitting CUDA version; there are built in checks / messages regarding the required CUDA version for the chosen ONNX runtime version. The latest versions (1.9.0 and 1.10.0) currently require CUDA 11.4, which only supports Visual Studio <= 2019.
- `ENABLE_ASTRA` - Whether to build the [astra toolbox](https://www.astra-toolbox.com) reconstruction library and open_iA ASTRA module. This will also fetch and build boost, so enabling it will considerably increase the build time. (default: disabled). **Note:** Make sure to install a fitting CUDA version; on Windows, make sure it works together with the version of Visual Studio you are using!
- `ENABLE_EIGEN` - Whether to fetch and use eigen (default: disabled)
- `ENABLE_FILTERS` - Whether to build image processing filters (smoothing, segmentation, intensity transformations, geometric transformations, ...) (default: enabled)
- `ENABLE_HDF5` - Whether to fetch and build HDF5 library and use it in open_iA (default: disabled). **Note:** The currently latest HDF5 version (1.12.1) does not come with support for building on Visual Studio 2022, use Visual Studio <= 2019 if you want to use HDF5!
- `ENABLE_OPENCL` - Enables OpenCL; the DreamCaster tool depends is only enabled if this setting is enabled; enabling this option also enables some GPU-optimized ITK filters (default: disabled)
- `ENABLE_PRECOMPILE` - Whether to build open_iA with precompiled headers enabled (default: disabled; NOT included in `ENABLE_ALL`)
- `ENABLE_TOOLS` - Whether to build common tool modules, e.g. FeatureScout, 4DCT, GEMSe, Dynamic Volume Lines, FIAKER, ... (default: enabled)
- `ENABLE_TEST` - Whether to enable build of tests runners and the capability to submit CDash test runs (default: disabled)
- `ENABLE_VR` - Whether to build VR module; requires OpenVR SDK, which will be fetched automatically; also boost (includes) are required (default: disabled).
- `ENABLE_ALL` Enables all optional modules and filters (see also the ENABLE_xyz options above; all except for ENABLE_PRECOMPILE are enabled if this is set to on. Note that unchecking this box again does NOT have any direct effect; it will not automatically set these options to unchecked or their state before. But you will have to uncheck the option if you want to disable any of the single ENABLE_xyz options affected by this setting, otherwise they will be re-enabled on next 'Configure' run) (default: disabled)


### Library build options

- `ARCHIVE_DIR` - A cache directory for downloaded library archives. Set to a directory with existing archives to avoid re-downloading files already present on the local machine / in the local network.
- `AI_ONNX_USE_CUDA` - Whether to use the CUDA version of the ONNX runtime; if disabled, use DirectML. Only available on Windows (on Linux, DirectML is not available) (default: disabled)
- `BUILD_TYPE` - The type of build to do (Release, Debug, RelWithDebInfo or MinSizeRel); only available on single-configuration generators (e.g. Unix Makefiles, ninja), not on multi-configuration generators (Visual Studio, XCode) (default: Release)
- `BUILD_ASTRA` - Whether to build ASTRA in the superbuild (provided that `ENABLE_ASTRA` is enabled). If disabled, you need to set ASTRA_DIR to an existing ASTRA build (default: enabled)
- `BUILD_BOOST` - Whether to build BOOST in the superbuild (provided that either `ENABLE_ASTRA` or `ENABLE_VR` is enabled). If disabled, you need to either have a boost installation or a boost build available, and you will be required to set `BOOST_DIR` accordingly (default: enabled)
- `BUILD_VTK` - Whether to build VTK in the superbuild. If disabled, you need to set VTK_DIR to an existing VTK build (default: enabled)
- `BUILD_ITK` - Whether to build ITK in the superbuild. If disabled, you need to set ITK_DIR to an existing ITK build (default: enabled)
- `ITK_USE_GIT_REPO` - Whether to use git repository for ITK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly (since cloning the full repository is much slower than downloading and extracting a release archive).
- `ITK_VERSION` - The ITK version to build, in case `BUILD_ITK` is enabled and `ITK_USE_GIT_REPO` is disabled; if `ITK_USE_GIT_REPO` is enabled, use `ITK_GIT_TAG` instead.
- `ITK_GIT_TAG` - The ITK git tag to use in build, in case `BUILD_ITK` is enabled and `ITK_USE_GIT_REPO` is enabled; if `ITK_USE_GIT_REPO` is disabled, use `ITK_VERSION` instead.
- `OPENVR_VERSION` - The OpenVR version to fetch, if `ENABLE_VR` is enabled.
- `VTK_GIT_TAG` - The VTK git tag to use in build, in case `BUILD_VTK` is enabled and `VTK_USE_GIT_REPO` is enabled; if `VTK_USE_GIT_REPO` is disabled, use `VTK_VERSION` instead.
- `VTK_SMP_TYPE` - Choose the SMP implementation to use in the VTK build to speed up filters with parallel implementations - available are sequential (no parallelization), OpenMP, and TBB (Intel Thread Building Blocks - will require installation of Intel OneAPI SDK) (default: OpenMP)
- `VTK_USE_GIT_REPO` - Whether to use git repository for VTK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly (since cloning the full repository is much slower than downloading and extracting a release archive).
- `VTK_VERSION` - The VTK version to build, in case `BUILD_VTK` is enabled and `VTK_USE_GIT_REPO` is disabled; if `VTK_USE_GIT_REPO` is enabled, use `VTK_GIT_TAG` instead.



## Troubleshooting:

- The `$` sign in any code above indicates the shell prompt, do not enter this sign.

- If there is a problem in building open\_iA itself, and the problem is not mentioned in this section, please also check the troubleshooting sections of our detailed [Windows](https://github.com/3dct/open_iA/wiki/Windows-Build#troubleshooting) and [Linux](https://github.com/3dct/open_iA/wiki/Linux-Build#troubleshooting) build instructions.

- CMake might encounter an error telling you that some library is missing which you still need to install. If that happens for a library not mentioned in the prerequisites above, please let us know [via the superbuild issue tracker](https://github.com/3dct/open_iA-superbuild/issues).

- Ninja as build tool is known to cause problems at the moment with the superbuild; it produces error messages such as `ninja: error: 'OpenCL\_ICD', needed by 'ITK-prefix/src/ITK-stamp/ITK-mkdir', missing and no known rule to make it`. Workaround: Use "Unix Makefiles" generator instead of "Ninja".

- For the prerequisites installation on Linux: the command to install packages as well as the names of the packages will differ on distributions other than Ubuntu, please consult the manual or community of your distribution.

- The method described above sets up a basic build environment.
  - It will only build VTK and ITK libraries.
  - It builds the core executables as well as some of the advanced analysis modules.
  - If you require more modules or other library support, e.g. Eigen, OpenCL or HDF5 support, see the `ENABLE_...` flags in the [Configuration Options](#configuration-options).

- If you use the Unix Makefile generator and have more than 4 cores in your system, you will probably want to increase the number of parallel builds (it is set to 4 in step 4 above). To e.g. run a make build with 8 parallel threads, run
  `$ make -j 8`
  It is recommended to choose here the number of processor cores available on your system (or twice that number if you processor supports Hyper-Threading).

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

- On Ubuntu (22.04, but maybe also other versions), there is a known incompatibility between CUDA 11.5 as installed from the package sources and boost, resulting in the astra build reporting `configure: error: Boost and CUDA versions are incompatible. You probably have to upgrade boost.`, similar [as reported here](https://github.com/NVlabs/instant-ngp/issues/119). This happens even when using the latest boost version available (1.78 at the time of writing this); it is fixed with CUDA 11.6. So, in order to use CUDA on Ubuntu, install the latest CUDA 11.6 release with the "runfile" installer.

- If you encounter any error not mentioned here, please [post an issue](https://github.com/3dct/open_iA-superbuild/issues)!