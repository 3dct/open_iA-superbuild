## open_iA Superbuild

The goal of this repository is to simplify setting up the build environment of [open_iA](https://github.com/3dct/open_iA).
The instructions here therefore can be seen as an alternative to [the detailed build instructions in the open_iA repository](https://github.com/3dct/open_iA/wiki/Linux-Build).
The build steps here were tested only on Ubuntu 18.04.1 so far, and might fail on other distributions (and very likely on other operating systems). We will continue to improve them, please keep your eyes open for updates! See also the troubleshooting section below.

## Build steps

1. Install required applications, headers and libraries:
  - CMake
  - g++
  - make
  - git
  - OpenCL development headers & libraries
  - OpenGL development headers & libraries
  - Qt libraries & headers
  - On Ubuntu, execute:
```
  $ sudo apt install cmake g++ make git \
    libgl1-mesa-dev libxt-dev \
    ocl-icd-opencl-dev opencl-headers opencl-clhpp-headers \
    qtbase5-dev libqt5x11extras5-dev qttools5-dev qt5-default \
    libxt-dev libqt5opengl5-dev
```
  - The command to install packages as well as the names of the packages to install will differ on other distributions, please consult the manual or community of your distribution.

2. Open a shell in the folder where you want to place open_iA (preferrably, use a short folder name, e.g. directly in "/home/youruser")

3. Clone this repository:
   `$ git clone https://github.com/3dct/open_iA-superbuild.git`

4. Enter the newly created working copy:
   `$ cd open_iA-superbuild`

5. Execute cmake:
   `$ cmake .`

6. Build everything:
   `$ make -j 4`
  
7. When finished successfully, open_iA executables will be built in the `open_iA/bin/bin` subdirectory of where you issued `make`!

## Troubleshooting / Adaptations:

- The `$` sign in the code above indicates the shell prompt, do not enter this sign.

- The method described here currently only sets up VTK and ITK libraries
  - It will only enable the build of the core executables, but none of the advanced analysis modules. To enable these, after the build has finished, you will have to open another CMake, switch to the `open_iA/bin` binaries folder and enable the required modules.
  -  If you require a module with more elaborate prerequisites, such as the Astra reconstruction module or the HDF5 integration, you have to continue on the [detailed build instructions page](https://github.com/3dct/open_iA/wiki/Linux-Build) in the section of the respective library after setup here is finished.

- If CMake encounters an error, it probably will tell you that some library is missing which you still need to install.

- If there is any error you cannot fix yourself, feel free to [post an issue](https://github.com/3dct/open_iA-superbuild/issues)!

- If you have more than 4 cores in your system, you will probably want to increase the number 4 in step 6 above to that number, e.g.
  `$ make -j 8`

- If you'd like to rename the `open_iA-superbuild` folder created in step 3 above:
  - Either add the desired folder name as additional parameter to the command line given above, e.g.:
    `$ git clone https://github.com/3dct/open_iA-superbuild.git my_open_iA`
  
  - Or rename the folder directly after cloning, i.e., after step 3, do:
    `$ mv open_iA-superbuild my_open_iA`
    
- If you want to put the build environment for open_iA somewhere else as where you checked out the superbuild,
  or if you want to keep the root folder of the superbuild clear (i.e. if you want to do a so-called out-of-source build),
  you can also call CMake from another folder; the relative path given as parameter just has to reference the superbuild directory.

  For example:
  If you checked out this repository to /home/youruser/open_iA-superbuild,
  and want to create the build environment, in /home/youruser/open_iA_Env, then, instead of step 5 above, execute:
  ```
  $ mkdir /home/youruser/open_iA_Env
  $ cd /home/youruser/open_iA_Env
  $ cmake /home/youruser/open_iA-superbuild
  ```
- You can change the build tool by modifying the CMakeLists.txt in the open_iA-superbuild folder after step 3.
  Search for the line `SET (CMAKE_GENERATOR "Unix Makefiles")` and change it to e.g. `SET (CMAKE_GENERATOR "ninja")`.
  Note that you probably need to execute the cmake command twice before this change will be used everywhere!
