cmake_minimum_required(VERSION 3.13)
if (CMAKE_MAJOR_VERSION GREATER 3 OR (CMAKE_MAJOR_VERSION EQUAL 3 AND CMAKE_MINOR_VERSION GREATER 15))
	# prevent all submodules to be fetched in ExternalProject_Add if GIT_SUBMODULES is ""
	CMAKE_POLICY(SET CMP0097 NEW)
endif()
# 3.13 is required because we use VS_DEBUGGER_COMMAND and VS_DEBUGGER_ENVIRONMENT target properties introduced with that version
project(open_iA_superbuild LANGUAGES CXX)

include(ExternalProject)

option (USE_GIT_REPOS "Whether to use git repositories for VTK and ITK libraries. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly." OFF)

option (ENABLE_ALL "Enables all optional modules and filters (see also separate ENABLE_xyz options)" OFF)
option (ENABLE_FILTERS "Whether to build image processing filters (smoothing, segmentation, intensity transformations, geometric transformations, ...)" ON)
option (ENABLE_TOOLS "Whether to build common tool modules, e.g. FeatureScout, 4DCT, GEMSe, Dynamic Volume Lines, FIAKER, ..." ON)
option (ENABLE_TEST "Whether to enable build of tests runners and the capability to submit CDash test runs" OFF)
option (ENABLE_ASTRA "Whether to build ASTRA reconstruction library and open_iA ASTRA module. This will also fetch and build boost, so enabling it will considerably increase the build time!" OFF)
option (ENABLE_VR "Whether to enable VR module" OFF)
option (ENABLE_EIGEN "Whether to fetch and use eigen" OFF)
#option (ENABLE_HDF5 "Whether to build HDF5 library and use it in open_iA" OFF)
option (ENABLE_PRECOMPILE "Whether to build open_iA with precompiled headers enabled" OFF)

#option (BUILD_VTK "Build VTK in the superbuild. If disabled, you need to set VTK_DIR to an existing VTK build" ON)
#option (BUILD_ITK "Build ITK in the superbuild. If disabled, you need to set ITK_DIR to an existing ITK build" ON)
#option (BUILD_ASTRA "Build Astra Toolbox in the superbuild. If disabled, you need to set ASTRA_DIR to an existing Astra Toolkit build" ON)
#option (FETCH_OPENVR "Fetches release of OpenVR SDK. If disabled, you need to set OPENVR_DIR to an existing OpenVR SDK directory" ON)
#option (FETCH_EIGEN "Fetches release of eigen. If disabled, you need to set EIGEN3_DIR to an existing Eigen 3.x directory" ON)
#option (BUILD_OPENCL "Build OpenCL ICD loader. If disabled, you need to set OPENCL_DIR to an existing OpenCL ICD loader build" ON)
set (BUILD_OPENCL "ON")
#option (BUILD_HDF5 "Build HDF5. If disabled, you need to set HDF5_DIR to an existing HDF5 build" ON)

if (ENABLE_ALL)
	SET (ENABLE_FILTERS "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_TOOLS "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_TEST "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_ASTRA "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_VR "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_EIGEN "ON" CACHE BOOL "" FORCE)
	#SET (ENABLE_HDF5 "ON" CACHE BOOL "" FORCE)
endif()

set (VTK_SHORT_VERSION "8.2")
set (VTK_LONG_VERSION  "${VTK_SHORT_VERSION}.0")
set (VTK_ZIP_SHA512 "FD6582EA20A0B339EC847FEFA18956F9919535C6CBE1C5D443E2B3E302D6F6045C4250B0A89719EC4D4309E25F05449DACF37973615FC59BED33714EC6081BF9")
set (VTK_GZIP_SHA512 "521BD4DABEDBC24B0E80A314A34ECD7554B04AF28A7973245E3A9CF99A09B995D1B8AC42305C8E53369F226A0A6DA3CDB29105BA2C90B46492736EF717760286")

set (ITK_VERSION  "5.0.1")
set (ITK_ZIP_SHA512 "f1a871742247e9596b39126ee70b9ab5e6e3dc6fabfc54fc2f84d1f174b51f4f9086eae5769f6bc1a28eaf81183ebe77d66ca0199d65e65ef689ab9344bd81f2")
set (ITK_GZIP_SHA512 "f36613ff72c513ded3d32504f71308a94fe75555cf9fd22b77485d1375601f6e1f1539cc5ac82a9e1e229bcf514a88ccb55122a7dfc74a6ae1b6604aa70bd814")

#set (OPEN_IA_GIT_TAG "202dfdb3c100babd81b011f280d7663351e28c5a") # = "2019.12"
set (OPEN_IA_GIT_TAG "develop")

FIND_PACKAGE(Qt5 COMPONENTS Widgets Xml Network Test OpenGL PrintSupport REQUIRED)

set (VTK_REPOSITORY "https://github.com/Kitware/VTK.git")
# set (VTK_REPOSITORY "https://gitlab.kitware.com/vtk/vtk.git")
set (ITK_REPOSITORY "https://github.com/InsightSoftwareConsortium/ITK.git")
# set (ITK_REPOSITORY "https://github.com/Kitware/ITK.git")
# set (ITK_REPOSITORY "http://itk.org/ITK.git")

# TODO:
#   - Download/install Qt?
#   - check behavior when there are changes in git (and tag not the one referenced in ExternalProject_Add) / offline
#   - astra
#   - hdf5
#   - eigen
set (VTK_URL "https://www.vtk.org/files/release/${VTK_SHORT_VERSION}/VTK-${VTK_LONG_VERSION}")
set (ITK_URL "https://github.com/InsightSoftwareConsortium/ITK/releases/download/v${ITK_VERSION}/InsightToolkit-${ITK_VERSION}")

set (OPENVR_VERSION "1.9.15")
set (OPENVR_URL "https://github.com/ValveSoftware/openvr/archive/v${OPENVR_VERSION}.zip")
set (OPENVR_ARCHIVE_SHA512 "CB047D92C383F5CF761FD5EE262DDA8755BF592AC827A8C7314294DDBA489E4709EB3AA756207382E1C16BB02DAED77EAA6CEBE48411BCE5A4C2957855FA75D9")

set (ASTRA_REPOSITORY "https://github.com/astra-toolbox/astra-toolbox.git")
set (ASTRA_TAG        "06322245a638e435d29bc8f027aed0bd976139d0") # version "1.9.9dev"
set (BOOST_VERSION    "1.72.0")
set (BOOST_ARCHIVE_SHA512 "BEB4D099E9427394D6AD0CE0D526EE2C6E96CC4CAF5DE4747D009B01CD3E517A988E79C0D4522E49217C6A7EFD551075FB18BFAC51F344CEA9978CDCFC57CC56") # for 1.72.0 zip
set (BOOST_MSVC_TOOLSET "msvc-14.2")
set (BOOST_LIB_SUFFIX "vc142")
set (BOOST_SHORTVER "1_72")

set (EIGEN_VERSION "3.3.7")
set (EIGEN_URL "https://gitlab.com/libeigen/eigen/-/archive/${EIGEN_VERSION}/eigen-${EIGEN_VERSION}.zip")
set (EIGEN_ARCHIVE_SHA512 "6C19DF0271503DDCE61724729031A3B885D86F35D1F24363A76F9E300234D3257FF56020ACE624B2C011E17CBA6D69A3014508C5E6DC3436BBE331A2129E2175") # for 3.3.7 zip

if (WIN32)
	SET (VTK_URL "${VTK_URL}.zip")
	set (ITK_URL "${ITK_URL}.zip")
	set (VTK_ARCHIVE_SHA512 ${VTK_ZIP_SHA512})
	set (ITK_ARCHIVE_SHA512 ${ITK_ZIP_SHA512})
else()
	set (VTK_URL "${VTK_URL}.tar.gz")
	set (ITK_URL "${ITK_URL}.tar.gz")
	set (VTK_ARCHIVE_SHA512 ${VTK_GZIP_SHA512})
	set (ITK_ARCHIVE_SHA512 ${ITK_GZIP_SHA512})
endif()

SET (ADDITIONAL_LIB_PATHS "") # for additional lib paths to be added to MSVC debugger environment path

set (VTK_CMAKE_ARGS
	"-DBUILD_SHARED_LIBS:BOOL=On"
	"-DVTK_LEGACY_REMOVE:BOOL=On"
	"-DModule_vtkGUISupportQt:BOOL=On"
	"-DModule_vtkGUISupportQtOpenGL:BOOL=On"
	"-DModule_vtkRenderingQt:BOOL=On"
	"-DModule_vtkViewsQt:BOOL=On"
	"-DQt5_DIR:PATH=${Qt5_DIR}")
IF (APPLE)
	SET (VTK_RENDER_BACKEND "OpenGL")
ELSE ()
	SET (VTK_RENDER_BACKEND "OpenGL2")
ENDIF()
LIST(APPEND VTK_CMAKE_ARGS "-DVTK_RENDERING_BACKEND:STRING=${VTK_RENDER_BACKEND}")
set (VTK_BIN_DIR "${CMAKE_BINARY_DIR}/vtk/bin-${VTK_LONG_VERSION}")
set (VTK_DEPENDS "")
if (ENABLE_VR)
	SET (OPENVR_BIN_DIR "${CMAKE_BINARY_DIR}/openvr-${OPENVR_VERSION}")
	ExternalProject_Add (OpenVR
		URL               ${OPENVR_URL}
		URL_HASH          "SHA512=${OPENVR_ARCHIVE_SHA512}"
		SOURCE_DIR        "${OPENVR_BIN_DIR}"
		BUILD_IN_SOURCE   1
        CONFIGURE_COMMAND ""
        BUILD_COMMAND     ""
		INSTALL_COMMAND   "")
	SET (VTK_DEPENDS OpenVR)
	LIST(APPEND VTK_CMAKE_ARGS
		"-DModule_vtkRenderingOpenVR:BOOL=ON"
		"-DOPENVR_ROOT_DIR:PATH=${OPENVR_BIN_DIR}")
	STRING(REPLACE "/" "\\" OPENVR_WIN_DIR "${OPENVR_BIN_DIR}")
	SET (ADDITIONAL_LIB_PATHS "${ADDITIONAL_LIB_PATHS};${OPENVR_WIN_DIR}\\bin\\win64")
endif()
if (USE_GIT_REPOS)
	ExternalProject_Add( VTK
		GIT_REPOSITORY    "${VTK_REPOSITORY}"
		GIT_TAG           "${VTK_LONG_VERSION}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_LONG_VERSION}"
		BINARY_DIR        "${VTK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
		CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
		INSTALL_COMMAND   ""
		DEPENDS           ${VTK_DEPENDS})
else()
	ExternalProject_Add( VTK
		URL               ${VTK_URL}
		URL_HASH          "SHA512=${VTK_ARCHIVE_SHA512}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_LONG_VERSION}"
		BINARY_DIR        "${VTK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
		CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
		INSTALL_COMMAND   ""
		DEPENDS           ${VTK_DEPENDS})
endif()

set (ITK_DEPENDS VTK)
if (BUILD_OPENCL)
	set (OPENCL_DIR "${CMAKE_BINARY_DIR}/OpenCL")
	set (OPENCL_C_HEADERS_REPO "https://github.com/KhronosGroup/OpenCL-Headers.git")
	set (OPENCL_CPP_HEADERS_REPO "https://github.com/KhronosGroup/OpenCL-CLHPP.git")
	set (OPENCL_ICD_REPO "https://github.com/KhronosGroup/OpenCL-ICD-Loader.git")
	ExternalProject_Add (OpenCL_C
		GIT_REPOSITORY    ${OPENCL_C_HEADERS_REPO}
		GIT_TAG           "master"
		SOURCE_DIR        "${OPENCL_DIR}/cheaders"
		BUILD_IN_SOURCE   1
        CONFIGURE_COMMAND ""
        BUILD_COMMAND     ""
		INSTALL_COMMAND   "")
	ExternalProject_Add (OpenCL_CPP
		GIT_REPOSITORY    ${OPENCL_CPP_HEADERS_REPO}
		GIT_TAG           "master"
		GIT_SUBMODULES    ""
		SOURCE_DIR        "${OPENCL_DIR}/cppheaders"
		BUILD_IN_SOURCE   1
        CONFIGURE_COMMAND ""
        BUILD_COMMAND     ""
		INSTALL_COMMAND   "")
	# Find way to skip re-build of OpenCL every time!
	ExternalProject_Add (OpenCL_ICD
		GIT_REPOSITORY    ${OPENCL_ICD_REPO}
		GIT_TAG          "master"
		GIT_SUBMODULES    ""
		SOURCE_DIR        "${OPENCL_DIR}/icdloader"
		BINARY_DIR        "${OPENCL_DIR}/icd-bin"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
		CMAKE_ARGS        "-DOPENCL_ICD_LOADER_REQUIRE_WDK:BOOL=OFF"
		INSTALL_COMMAND   ""
		DEPENDS           OpenCL_CPP OpenCL_C)
	add_custom_command(TARGET OpenCL_ICD PRE_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/cheaders/CL" "${OPENCL_DIR}/icdloader/inc/CL")
	add_custom_command(TARGET OpenCL_ICD PRE_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy "${OPENCL_DIR}/cppheaders/include/CL/cl.hpp" "${OPENCL_DIR}/cppheaders/include/CL/cl2.hpp" "${OPENCL_DIR}/icdloader/inc/CL/")
	add_custom_command(TARGET OpenCL_ICD POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/icdloader/inc/CL" "${OPENCL_DIR}/include/CL")
	# Find way to copy any/current configuration!
	#add_custom_command(TARGET OpenCL_ICD POST_BUILD
	#	COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/icd-bin/Debug" "${OPENCL_DIR}/lib")
	add_custom_command(TARGET OpenCL_ICD POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/icd-bin/Release" "${OPENCL_DIR}lib")
	LIST (APPEND ITK_DEPENDS OpenCL_ICD)
else()
	set(OPENCL_DIR "${OPENCL_DIR}"
		CACHE PATH
		"OpenCL ICD loader directory")
endif()

if (NOT OPENCL_DIR)
	message(SEND_ERROR "OPENCL_DIR not set! Either enable BUILD_OPENCL or set OPENCL_DIR to a valid OpenCL ICD loader directory!")
endif()

set (ITK_CMAKE_ARGS 
	"-DBUILD_SHARED_LIBS:BOOL=On"
	"-DITK_USE_GPU:BOOL=On"
	"-DITK_USE_64BITS_IDS:BOOL=On"
	"-DModule_ITKReview:BOOL=On"
	"-DModule_ITKVtkGlue:BOOL=On"
	"-DModule_HigherOrderAccurateGradient:BOOL=On"
	"-DVTK_DIR:PATH=${VTK_BIN_DIR}"
	"-DQt5_DIR:PATH=${Qt5_DIR}"
	"-DOPENCL_ROOT_DIR=${OPENCL_DIR}"
	"-DOPENCL_INCLUDE_DIRS=${OPENCL_DIR}/include"
	"-DOPENCL_LIBRARIES=${OPENCL_DIR}/lib/OpenCL.lib")
set (ITK_BIN_DIR "${CMAKE_BINARY_DIR}/itk/bin-${ITK_VERSION}")
if (USE_GIT_REPOS)
	ExternalProject_Add( ITK
		GIT_REPOSITORY    "${ITK_REPOSITORY}"
		GIT_TAG           "${ITK_VERSION}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src-${ITK_VERSION}"
		BINARY_DIR        "${ITK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
		CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
		INSTALL_COMMAND   ""
		DEPENDS           ${ITK_DEPENDS})
else()
	ExternalProject_Add( ITK
		URL               "${ITK_URL}"
		URL_HASH          "SHA512=${ITK_ARCHIVE_SHA512}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src-${ITK_VERSION}"
		BINARY_DIR        "${ITK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
		CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
		INSTALL_COMMAND   ""
		DEPENDS           ${ITK_DEPENDS})
endif()

if (ENABLE_ASTRA)
	set (ASTRA_TOOLBOX_DIR "${CMAKE_BINARY_DIR}/astra")
	# fetch and build boost...
	# inspired by https://github.com/apache/geode-native/blob/develop/dependencies/boost/CMakeLists.txt
	string(REPLACE "." "_" BOOST_VERSION_UNDERSCORE ${BOOST_VERSION})
	SET (BOOST_URL "https://dl.bintray.com/boostorg/release/${BOOST_VERSION}/source/boost_${BOOST_VERSION_UNDERSCORE}.zip")
	SET (B2_FLAGS "-j8 address-model=64 architecture=x86 link=static threading=multi runtime-link=shared --build-type minimal stage")
	if (WIN32)
		set ( BOOTSTRAP_COMMAND ${CMAKE_BINARY_DIR}\\boost\\bootstrap.bat )
		set ( B2_COMMAND ${CMAKE_BINARY_DIR}\\boost\\b2 )
		set ( B2_FLAGS ${B2_FLAGS} toolset=${BOOST_MSVC_TOOLSET} )
	else()
		set ( BOOTSTRAP_COMMAND ${CMAKE_BINARY_DIR}/boost/bootstrap.sh )
		set ( B2_COMMAND ${CMAKE_BINARY_DIR}/boost/b2 )
		set ( B2_FLAGS ${B2_FLAGS} )
	endif()
	SET (BOOST_BIN_DIR "${CMAKE_BINARY_DIR}/boost")
	SET (ASTRA_BIN_DIR "${CMAKE_BINARY_DIR}/astra")
	ExternalProject_Add( boost
		URL               ${BOOST_URL}
		URL_HASH          SHA512=${BOOST_ARCHIVE_SHA512}
		BUILD_IN_SOURCE   1
		SOURCE_DIR        "${BOOST_BIN_DIR}"
		CONFIGURE_COMMAND ${BOOTSTRAP_COMMAND}
		BUILD_COMMAND     ${B2_COMMAND} ${B2_FLAGS}
		INSTALL_COMMAND   ""  # ${B2_COMMAND} ${B2_FLAGS} install
	)
	if (WIN32)
		set (ASTRA_CONFIGURE_COMMAND "${GIT_EXECUTABLE} apply ${CMAKE_SOURCE_DIR}/vs-2019.patch")  # specific to VS 2019!
		set (ASTRA_BUILD_DIR "${ASTRA_BIN_DIR}/build/msvc")
		set (ASTRA_BUILD_COMMAND "devenv ${ASTRA_BUILD_DIR}/astra_vc14.vcxproj /Build")
	else()
		set (ASTRA_BUILD_DIR "${ASTRA_BIN_DIR}/build/linux")
		set (ASTRA_CONFIGURE_COMMAND "${ASTRA_BUILD_DIR}/autogen.sh && ${ASTRA_BIN_DIR}/build/linux/configure")
		set (ASTRA_BUILD_COMMAND "cd ${ASTRA_BUILD_DIR} && make -j 8")
	endif()
	# fetch and build astra:
	ExternalProject_Add( astra
		GIT_REPOSITORY    "${ASTRA_REPOSITORY}"
		GIT_TAG           "${ASTRA_TAG}"
		SOURCE_DIR        "${ASTRA_BIN_DIR}"
		BUILD_IN_SOURCE   1
		CONFIGURE_COMMAND "${ASTRA_CONFIGURE_COMMAND}"
		BUILD_COMMAND     "${ASTRA_BUILD_COMMAND}"
		DEPENDS           boost
	)
	add_custom_command(TARGET boost POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy_directory "${CMAKE_BINARY_DIR}/boost/boost" "${CMAKE_BINARY_DIR}/astra/lib/include/boost"
	)
	add_custom_command(TARGET boost POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E make_directory "${CMAKE_BINARY_DIR}/astra/lib/x64")
	set (BOOST_LIB_BASE "${CMAKE_BINARY_DIR}/boost/stage/lib/libboost_")
	add_custom_command(TARGET boost POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy "${BOOST_LIB_BASE}chrono-${BOOST_LIB_SUFFIX}-mt-gd-x64-${BOOST_SHORTVER}.lib" "${BOOST_LIB_BASE}chrono-${BOOST_LIB_SUFFIX}-mt-x64-${BOOST_SHORTVER}.lib" "${BOOST_LIB_BASE}date_time-${BOOST_LIB_SUFFIX}-mt-gd-x64-${BOOST_SHORTVER}.lib" "${BOOST_LIB_BASE}date_time-${BOOST_LIB_SUFFIX}-mt-x64-${BOOST_SHORTVER}.lib" "${BOOST_LIB_BASE}system-${BOOST_LIB_SUFFIX}-mt-gd-x64-${BOOST_SHORTVER}.lib" "${BOOST_LIB_BASE}system-${BOOST_LIB_SUFFIX}-mt-x64-${BOOST_SHORTVER}.lib" "${BOOST_LIB_BASE}thread-${BOOST_LIB_SUFFIX}-mt-gd-x64-${BOOST_SHORTVER}.lib" "${BOOST_LIB_BASE}thread-${BOOST_LIB_SUFFIX}-mt-x64-${BOOST_SHORTVER}.lib" "${CMAKE_BINARY_DIR}/astra/lib/x64"
	)
endif()


set (OPEN_IA_CMAKE_ARGS
	"-DITK_DIR:PATH=${ITK_BIN_DIR}"
	"-DOPENCL_ROOT_DIR=${OPENCL_DIR}"
	"-DOPENCL_INCLUDE_DIRS=${OPENCL_DIR}/include"
	"-DOPENCL_LIBRARIES=${OPENCL_DIR}/lib/OpenCL.lib")
if (ENABLE_FILTERS)
	LIST (APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_CommonImageFilters:BOOL=ON"
		"-DModule_ExtractSurface:BOOL=ON"
		"-DModule_FeatureCharacteristics:BOOL=ON"
		"-DModule_MetaFilters:BOOL=ON"
		"-DModule_Metrics:BOOL=ON"
		"-DModule_Segmentation:BOOL=ON")
endif()
if (ENABLE_TOOLS)
	LIST (APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_4DCT:BOOL=ON"
		"-DModule_AdaptiveThreshold:BOOL=ON"
		"-DModule_BoneThickness:BOOL=ON"
		"-DModule_DynamicVolumeLines:BOOL=ON"
		"-DModule_FeatureScout:BOOL=ON"
		"-DModule_FiAKEr:BOOL=ON"
		"-DModule_FoamCharacterization:BOOL=ON"
		"-DModule_FuzzyFeatureTracking:BOOL=ON"
		"-DModule_GEMSe:BOOL=ON"
		"-DModule_GPU_DreamcasterTool:BOOL=ON"
		"-DModule_ModalityExplorer:BOOL=ON"
		"-DModule_ParameterExplorer:BOOL=ON"
		"-DModule_PolygonPrimitives:BOOL=ON"
		"-DModule_PorosityAnalyser:BOOL=ON"
		"-DModule_XRF:BOOL=ON"
		"-DQt5Charts_DIR:PATH=${Qt5_DIR}Charts")
endif()
if (ENABLE_TEST)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DopeniA_TESTING_ENABLED:BOOL=ON")
endif()
if (ENABLE_PRECOMPILE)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DopeniA_PRECOMPILE:BOOL=ON")
endif()
SET (openiA_DEPENDS ITK)
if (ENABLE_ASTRA)
	LIST (APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_AstraReconstruction:BOOL=ON"
		"-DASTRA_TOOLBOX_DIR:PATH=${ASTRA_TOOLBOX_DIR}")
	LIST (APPEND openiA_DEPENDS astra)
endif()
if (ENABLE_VR)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_VR:BOOL=ON"
		"-DOPENVR_ROOT_DIR:PATH=${OPENVR_BIN_DIR}")
endif()
if (ENABLE_EIGEN)
	SET (EIGEN_BIN_DIR "${CMAKE_BINARY_DIR}/eigen-${EIGEN_VERSION}")
	ExternalProject_Add (eigen
		URL               ${EIGEN_URL}
		URL_HASH          "SHA512=${EIGEN_ARCHIVE_SHA512}"
		SOURCE_DIR        "${EIGEN_BIN_DIR}"
		BUILD_IN_SOURCE   1
        CONFIGURE_COMMAND ""
        BUILD_COMMAND     ""
		INSTALL_COMMAND   "")
	LIST (APPEND openiA_DEPENDS eigen)
	LIST(APPEND OPEN_IA_CMAKE_ARGS
		"-DEIGEN3_INCLUDE_DIR:PATH=${EIGEN_BIN_DIR}")
endif()
set (OPEN_IA_BIN_DIR "${CMAKE_BINARY_DIR}/open_iA/bin")
ExternalProject_Add( open_iA
	GIT_REPOSITORY    "https://github.com/3dct/open_iA.git"
	GIT_TAG           "${OPEN_IA_GIT_TAG}"
	SOURCE_DIR        "${CMAKE_BINARY_DIR}/open_iA/src"
	BINARY_DIR        "${OPEN_IA_BIN_DIR}"
	CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
	CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
	CMAKE_ARGS        "${OPEN_IA_CMAKE_ARGS}"
	INSTALL_COMMAND   ""
	DEPENDS           ${openiA_DEPENDS})


if (MSVC)
	string(REPLACE "/" "\\" OPEN_IA_BIN_DIR_WIN "${OPEN_IA_BIN_DIR}")
	STRING(REPLACE "/lib/cmake/Qt5" "/bin" Qt5_LIB_DIR ${Qt5_DIR})
	STRING(REPLACE "/" "\\" Qt5_LIB_DIR ${Qt5_LIB_DIR})
	STRING(REPLACE "/" "\\" VTK_WIN_DIR ${VTK_BIN_DIR})
	STRING(REPLACE "/" "\\" ITK_WIN_DIR ${ITK_BIN_DIR})
	set (openia_debug_cmd  "${OPEN_IA_BIN_DIR_WIN}\\x64\\$(Configuration)\\open_iA.exe")
	set (openia_debug_path "${Qt5_LIB_DIR};${VTK_WIN_DIR}\\bin\\$(Configuration);${ITK_WIN_DIR}\\bin\\$(Configuration)${ADDITIONAL_LIB_PATHS}")
	set_target_properties(open_iA PROPERTIES VS_DEBUGGER_COMMAND "${openia_debug_cmd}")
	set_target_properties(open_iA PROPERTIES VS_DEBUGGER_ENVIRONMENT "PATH=${openia_debug_path}")
	set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT open_iA)
endif()