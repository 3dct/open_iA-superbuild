## open\_iA Superbuild

This document provides the currently simplest way of setting up a build environment for [open\_iA](https://github.com/3dct/open_iA). The build steps here were tested on Ubuntu 18.04.1 and 18.10, as well as on Windows 10. They might or might not work on other distributions and operating systems. In case of problems, make sure to check the [Troubleshooting section](#Troubleshooting) below. Please let us know of your experience [in the issue tracker](https://github.com/3dct/open_iA-superbuild/issues).

## Build steps

In short, these are the steps required to set up a build environment for open\_iA:
1. Install prerequisites - git, CMake, and C++ build toolchain, as well as libraries and headers for OpenGL and Qt (and optionally OpenCL)
2. Clone  this repository to some folder on your machine
3. Run CMake on the checked out repository for the desired toolchain
4. Build with the toolchain configured in CMake
5. You're all set up: For further developments in open\_iA, use the `open_iA` subfolder of the superbuild binary folder.

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

On Linux, those packages should be available from the distribution repository.
  - For example, on Ubuntu 22.04, execute:
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
    OpenMP is required, and installs as separate package for clang. On versions of Ubuntu prior to 21.04, you might have to also install the `qt6-default` package in order to make Qt 6 the default.

  - On Fedora (36), the command required to install the required packages is:
    ```
    $ sudo dnf install automake boost-devel clang cmake \
      cmake-gui libomp-devel libtool libxkbcommon-devel \
      make ninja-build mesa-libGL-devel ocl-icd-devel \
      opencl-headers qt6-qtbase-devel qt6-qtcharts-devel \
      qt6-qtsvg-devel qt6-qttools-devel
    ```
    (installing both ninja and make build tools, and the clang compiler, in this example)

  - If you require any CUDA-dependant module (the Astra Reconstruction and the AI module currently),
    please refer to guides for how to install CUDA on your distribution. Also, make sure that the CUDA
    version that you install fits to other required libraries; see [the onnx CUDA execution provider compatibility list](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html), as well as the notes
    on boost and CUDA in the [Troubleshooting section](#Troubleshooting) below.

On Windows, download and install:
- [CMake](https://cmake.org/)
- [git](https://git-scm.com/download/win)
- Visual Studio (e.g. the free [Community Edition](https://visualstudio.microsoft.com/de/vs/older-downloads)); we recommend to use Visual Studio 2022.
- OpenGL headers are included in the Windows SDK installed along with Visual Studio
- Qt:
	- Go to the [Qt for Open Source Development](https://www.qt.io/download-open-source) page.
	- Under "Looking for Qt binaries?", you should see a button to download the "Qt Online Installer" (if it doesn't show up, try another browser).
	- In the installer, log in to your Qt account (or create one if you haven't got one yet)
    - In the "Select Components" step:
		- expand the "Qt" tree item
		- decide on a version
			- current recommendation is to use version 6.4(.3 or whichever is the latest in that branch)
			- **Note on Qt versions >= 6**: Use VTK 9.1 or later, or git from its master branch, see `VTK_VERSION`, `VTK_USE_GIT_REPO` and `VTK_GIT_TAG` options below.
			- **Note on Visual Studio 2022:** pre-built binaries are only provided for VS 2019, but these work with VS 2022 as well.
		- for that version, expand the tree and **only check** the "MSVC 2019 64-bit" option, as well as the "Qt Charts", "Qt HTTP Server (TP)" (only available for Qt >= 6.4) and "Qt WebSockets" options under "Additional Libraries"; make sure no other version or option than those two is checked (unless you require them for something else than open\_iA, of course).
- optional: an OpenCL SDK; if available, some GPU operations will be available; some modules require it. The generic OpenCL-ICD loader can be built from within the superbuild (via `ENABLE_OPENCL`). Or you can use an SDK best fitting your build system, e.g. the AMD OpenCL SDK for an AMD graphics card, the NVidia CUDA SDK for an NVidia graphics card, or the Intel OpenCL SDK for an onboard graphics card (but note that this ties your build to machines having similar compute capabilities)


### 2. Clone Repository

- Open a shell (terminal) in the folder where you want to place the open\_iA superbuild.
- Clone the superbuild repository: `$ git clone https://github.com/3dct/open_iA-superbuild.git`

Alternatively, use a git GUI tool of your choice for checking out the superbuild repository.
**Note:** As a precaution, the path you clone to should be short (under 20 characters); see also notes in the next section.


### 3. CMake Configure+Generate

Run CMake; you have two options:
1. If you prefer the command line, open a shell inside the cloned repository, and execute `$ cmake .`. In case of errors, we recommend to switch to option 2.
2. Run the graphical user interface version of CMake:
  - Enter the cloned repository folder under "Where is the source code"
  - Enter the folder you wish that vtk, itk and open\_iA sources and binaries shall be put in, under "Where to build the binaries".
    **Note:** On Windows, take care that the full path string of the folder specified in "Where you build the binaries" is very short; ideally something like `C:\open_iA_sb`. The exact possible maximum length is currently untested, but we have tested that it's safe with paths shorter than 20 characters. For example `C:\open_iA\build` (16 characters) should work, while `C:\open_iA\superbuild-bin` (25 characters) is definitely too long (for someone interested in technical details: the path will be incorporated into command lines created by CMake; and on Windows, there is a limit of 32.768 characters on the length of a command; and in some created command lines, the chosen path appears many times).
  - If you have a folder where you are caching or want to cache downloaded archives of any of the libraries built by the superbuild, adapt `ARCHIVE_DIR`. This can save download bandwidth, as existing source code archives are taken from there and not downloaded again (default: /archives subfolder of your binaries folder)
  - Press "Configure"
  - When asked for a build environment:
    - On Linux, use the usually pre-selected "Unix Makefiles"; "ninja" should also work, if you have ninja build tools installed (see prerequisites above)
    - On Windows, use the 64 bit version of the respective Visual Studio toolchain, e.g. "Visual Studio 17 2022" as generator (and "x64" as platform, but this is default anyway).
  - If Qt is installed in a non-standard folder, you will see a corresponding error. You will have to adapt the `QT_DIR` to point to the "lib/cmake/Qt6" (/"lib/cmake/Qt6") subdirectory of the respective version/build directory in the Qt installation folder (e.g. `6.4.3/msvc2019_64/lib/cmake/Qt6`).
  - In case you want to change some default options (for example to enable more than the default set of modules), check [Configuration Options](#configuration-options).

### 4. Build Everything

Build everything. The specific steps required here differ for the build environment you configured in CMake:
- With a "Unix Makefiles" generator on Linux for example, enter the folder configured in CMake as "Where to build the binaries" in a shell, then run: `$ make -j 4`
- Under Windows with "Visual Studio ..." generators, open the generated `open_iA-superbuild.sln` in Visual Studio, then build the solution.

### 5. Done!

When the build finishes successfully, open\_iA executables will be built in a subfolder to `open_iA/bin` configured in CMake as "Where to build the binaries".

How to run the open_iA you just built:

  - Under Linux, the executables are located under `open_iA/bin/bin`, and you can just execute the `open_iA` / `open_iA_cmd` executables directly (from the shell or via double-click in a file manager)
  - Under Windows, the executables are located `open_iA/bin/x64/CONFIGURATION` (with CONFIGURATION one of Release, Debug, RelWithDebInfo or MinSizeRel). Only the executables in the CONFIGURATION that was selected in Visual Studio when you built will be available. If you try to start the executable directly, you will get an error that dll's (Qt, VTK, ...) cannot be found. You have two options:
      - Use the .bat files (open_iA_gui_CONFIGURATION.bat) in `open_iA/bin/x64` - these will set the PATH environment accordingly so that the needed dll files are found
      - Run directly from the Visual Studio solution: in the superbuild, right-click on the "open_iA" project and select "Set as Startup Project"; then select "Start without Debugging" from the "Debug" menu (or press Ctrl+F5); or open the open_iA.sln in the `open_iA/bin/bin` subfolder, and proceed as with the superbuild solution (that is, mark "open_iA" as startup project, and run it via Ctrl+F5).


### Continuing from here

The superbuild is intended for setting up a build environment with all libraries that open_iA depends on.
For development in open\_iA or its modules, you subsequently don't need to (and in fact **should not**) run the full superbuild anymore.
Once the superbuild has finished successfully, focus your attention on the `open_iA` subfolder.
In its `src` subfolder, you will find a git repository set up for you with all of open\_iA's sources.
Use the `bin` subfolder as basis for building and for CMake Configure/Generate runs (that is, in the CMake GUI, enter 
Under Windows, you can open the .sln file generated in that folder to see open_iA and all its libraries and modules as separate sub-projects.

**You only ever need to go back to executing a build on the whole superbuild folder if you need to change something in one of the libraries that open\_iA depends on.**

You can continue with [our guide on how to develop a module](https://github.com/3dct/open_iA/wiki/Developing-a-simple-module).



## Configuration Options


### Enabling specific features/modules

- `ENABLE_AI` - Whether to build AI module; requires ONNX runtime, which will be fetched automatically; on Windows, you can change whether CUDA or DirectML backend is chosen with the option `AI_ONNX_USE_CUDA` (default: disabled) **Note:** AI support is currently under construction, as the onnx runtime interface and required libraries changed considerably recently; it will probably not work at this moment.
- `ENABLE_ASTRA` - Whether to build the [astra toolbox](https://www.astra-toolbox.com) reconstruction library and open\_iA ASTRA module. This will also fetch and build boost, so enabling it will considerably increase the build time. (default: disabled). **Note:** Make sure to install a fitting CUDA version; on Windows, make sure it works together with the version of Visual Studio you are using, on Linux, make sure it's compatible with the used compiler!
- `ENABLE_EIGEN` - Whether to fetch and use eigen (default: disabled)
- `ENABLE_FILTERS` - Whether to build image processing filters (smoothing, segmentation, intensity transformations, geometric transformations, ...) (default: enabled)
- `ENABLE_HDF5` - Whether to fetch and build HDF5 library and use it in open\_iA (default: disabled).
- `ENABLE_OPENCL` - Enables OpenCL; the DreamCaster tool depends is only enabled if this setting is enabled; enabling this option also enables some GPU-optimized ITK filters (default: disabled)
- `ENABLE_PRECOMPILE` - Whether to build open\_iA with precompiled headers enabled (default: disabled; NOT included in `ENABLE_ALL`)
- `ENABLE_TOOLS` - Whether to build common tool modules, e.g. FeatureScout, 4DCT, GEMSe, Dynamic Volume Lines, FIAKER, ... (default: enabled)
- `ENABLE_TEST` - Whether to enable build of tests runners and the capability to submit CDash test runs (default: disabled)
- `ENABLE_VR` - Whether to build VR module; requires OpenVR SDK, which will be fetched automatically; also boost (includes) are required (default: disabled).
- `ENABLE_ALL` Enables all optional modules and filters (see also the ENABLE\_xyz options above; all except for `ENABLE_PRECOMPILE` are enabled if this is set to on. Note that unchecking this box again does NOT have any direct effect; it will not automatically set these options to unchecked or their state before. But you will have to uncheck the option if you want to disable any of the single `ENABLE_xyz` options affected by this setting, otherwise they will be re-enabled on next 'Configure' run) (default: disabled)


### Library build options

- `ARCHIVE_DIR` - A cache directory for downloaded library archives. Set to a directory with existing archives to avoid re-downloading files already present on the local machine / in the local network.
- `AI_ONNX_USE_CUDA` - Whether to use the CUDA version of the ONNX runtime; if disabled, use DirectML. Only available on Windows (on Linux, DirectML is not available) (default: disabled)
- `BUILD_TYPE` - The type of build to do (Release, Debug, RelWithDebInfo or MinSizeRel); only available on single-configuration generators (e.g. Unix Makefiles, ninja), not on multi-configuration generators (Visual Studio, XCode) (default: Release)
- `BUILD_ASTRA` - Whether to build ASTRA in the superbuild (provided that `ENABLE_ASTRA` is enabled). If disabled, you need to set `ASTRA_DIR` to an existing ASTRA build (default: enabled)
- `BUILD_BOOST` - Whether to build BOOST in the superbuild (provided that either `ENABLE_ASTRA` or `ENABLE_VR` is enabled). If disabled, you need to either have a boost installation or a boost build available, and you will be required to set `BOOST_DIR` accordingly (default: enabled)
- `BUILD_VTK` - Whether to build VTK in the superbuild. If disabled, you need to set `VTK_DIR` to an existing VTK build (default: enabled)
- `BUILD_ITK` - Whether to build ITK in the superbuild. If disabled, you need to set `ITK_DIR` to an existing ITK build (default: enabled)
- `ITK_USE_GIT_REPO` - Whether to use git repository for ITK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly (since cloning the full repository is much slower than downloading and extracting a release archive).
- `ITK_VERSION` - The ITK version to build, in case `BUILD_ITK` is enabled and `ITK_USE_GIT_REPO` is disabled; if `ITK_USE_GIT_REPO` is enabled, use `ITK_GIT_TAG` instead.
- `ITK_GIT_TAG` - The ITK git tag to use in build, in case `BUILD_ITK` is enabled and `ITK_USE_GIT_REPO` is enabled; if `ITK_USE_GIT_REPO` is disabled, use `ITK_VERSION` instead.
- `OPENVR_VERSION` - The OpenVR version to fetch, if `ENABLE_VR` is enabled.
- `VTK_GIT_TAG` - The VTK git tag to use in build, in case `BUILD_VTK` is enabled and `VTK_USE_GIT_REPO` is enabled; if `VTK_USE_GIT_REPO` is disabled, use `VTK_VERSION` instead.
- `VTK_SMP_TYPE` - Choose the SMP implementation to use in the VTK build to speed up filters with parallel implementations - available are sequential (no parallelization), OpenMP, and TBB (Intel Thread Building Blocks - will require installation of Intel OneAPI SDK) (default: OpenMP)
- `VTK_USE_GIT_REPO` - Whether to use git repository for VTK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly (since cloning the full repository is much slower than downloading and extracting a release archive).
- `VTK_VERSION` - The VTK version to build, in case `BUILD_VTK` is enabled and `VTK_USE_GIT_REPO` is disabled; if `VTK_USE_GIT_REPO` is enabled, use `VTK_GIT_TAG` instead.



## Troubleshooting

- The `$` sign in any code above indicates the shell prompt, do not enter this sign.

- If there is a problem in building open\_iA itself, and the problem is not mentioned in this section, please also check the troubleshooting sections of our detailed [Windows](https://github.com/3dct/open_iA/wiki/Windows-Build#troubleshooting) and [Linux](https://github.com/3dct/open_iA/wiki/Linux-Build#troubleshooting) build instructions.

- CMake might encounter an error telling you that some library is missing which you still need to install. If that happens for a library not mentioned in the prerequisites above, please let us know [via the superbuild issue tracker](https://github.com/3dct/open_iA-superbuild/issues).

- Ninja as build tool can cause problems with the superbuild; it produces error messages such as `ninja: error: 'OpenCL\_ICD', needed by 'ITK-prefix/src/ITK-stamp/ITK-mkdir', missing and no known rule to make it`. Workaround: Use "Unix Makefiles" generator instead of "Ninja".

- For the prerequisites installation on Linux: the command to install packages as well as the names of the packages will differ on distributions other than Ubuntu, please consult the manual or community of your distribution.

- The method described above sets up a basic build environment.
  - By default, it will only build VTK and ITK libraries.
  - It builds the core executables as well as some of the advanced analysis modules.
  - If you require more modules or other library support, e.g. Eigen, OpenCL or HDF5 support, see the `ENABLE_...` flags in the [Configuration Options](#configuration-options).

- If you use the Unix Makefile generator and have more than 4 cores in your system, you will probably want to increase the number of parallel builds (it is set to 4 in step 4 above). To e.g. run a make build with 8 parallel threads, run
  `$ make -j 8`
  Instead of the exemplary number 8 above, it is recommended to specify the number of processor cores available on your system (or twice that number if you processor supports Hyper-Threading).

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

- On Windows, open_iA might refuse to build the AstraReconstruction module if only the Debug configuration was built (output window message: `CMake Warning at cmake/Modules/Modules.cmake:58 (message): Module_AstraReconstruction requires ASTRA_TOOLBOX_FOUND to be TRUE disabling the option!`). As a workaround, build the Release configuration, afterwards finding Astra should work fine (there can be errors when building multiple astra configurations though, see next point).

- On Windows, when switching configuration and building the astra library a second time, the build will probably fail due to the patch step being applied again, with output such as this:
  ```
  3> Performing patch step for 'astra'
  3> CUSTOMBUILD : error : patch failed: astra_vc14.vcxproj:37
  3> CUSTOMBUILD : error : astra_vc14.vcxproj: patch does not apply
  ```
  As a workaround, go to the superbuild folder, right click on the astra folder and select "Git Bash Here". In the opening command prompt, enter `git restore .` followed by enter. Then try the build again, this time it should run through fine.

- With eigen, you might encounter a compilation error with AVX2 option enabled; e.g. on Visual Studio: `C3861: '_cvtss_sh': identifier not found`. There is [an issue](https://gitlab.com/libeigen/eigen/-/issues/2395) for it in eigen issue tracker.

- If you encounter any error not mentioned here, please [post an issue](https://github.com/3dct/open_iA-superbuild/issues)!
