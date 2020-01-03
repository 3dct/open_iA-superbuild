cmake_minimum_required(VERSION 3.13)
# 3.13 is required because we use VS_DEBUGGER_COMMAND and VS_DEBUGGER_ENVIRONMENT target properties introduced with that version
project(open_iA_superbuild LANGUAGES CXX)

include(ExternalProject)

option (USE_GIT_REPOS "Whether to use git repositories for VTK and ITK libraries. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly." OFF)

option (BUILD_ALL "Enables all optional modules and filters (see also separate BUILD_xyz options)" OFF)
option (BUILD_FILTERS "Whether to build image processing filters (smoothing, segmentation, intensity transformations, geometric transformations, ...)" ON)
option (BUILD_TOOLS "Whether to build common tool modules, e.g. FeatureScout, 4DCT, GEMSe, Dynamic Volume Lines, FIAKER, ..." ON)
option (BUILD_TESTING "Whether to enable build of tests runners and the capability to submit CDash test runs" OFF)
#option (BUILD_ASTRA "Whether to build ASTRA reconstruction library and open_iA ASTRA module" OFF)
option (BUILD_VR "Whether to build VR module" OFF)
#option (BUILD_HDF5 "Whether to build HDF5 library and use it in open_iA" OFF)
option (BUILD_EIGEN "Whether to fetch and use eigen" OFF)
option (ENABLE_PRECOMPILE "Whether to build open_iA with precompiled headers enabled" OFF)

if (BUILD_ALL)
	SET (BUILD_FILTERS "ON" CACHE BOOL "" FORCE)
	SET (BUILD_TOOLS "ON" CACHE BOOL "" FORCE)
	SET (BUILD_TESTING "ON" CACHE BOOL "" FORCE)
	#SET (BUILD_ASTRA ON)
	SET (BUILD_VR "ON" CACHE BOOL "" FORCE)
	#SET (BUILD_HDF5 ON)
	SET (BUILD_EIGEN ON)
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
if (BUILD_VR)
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

set (ITK_CMAKE_ARGS 
	"-DBUILD_SHARED_LIBS:BOOL=On"
	"-DITK_USE_GPU:BOOL=On"
	"-DITK_USE_64BITS_IDS:BOOL=On"
	"-DModule_ITKReview:BOOL=On"
	"-DModule_ITKVtkGlue:BOOL=On"
	"-DModule_HigherOrderAccurateGradient:BOOL=On"
	"-DVTK_DIR:PATH=${VTK_BIN_DIR}"
	"-DQt5_DIR:PATH=${Qt5_DIR}")
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
		DEPENDS           VTK)
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
		DEPENDS           VTK)
endif()

set (OPEN_IA_CMAKE_ARGS
	"-DITK_DIR:PATH=${ITK_BIN_DIR}")
if (BUILD_FILTERS)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_CommonImageFilters:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_ExtractSurface:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_FeatureCharacteristics:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_MetaFilters:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_Metrics:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_Segmentation:BOOL=ON")
endif()
if (BUILD_TOOLS)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_4DCT:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_AdaptiveThreshold:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_BoneThickness:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_DynamicVolumeLines:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_FeatureScout:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_FiAKEr:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_FoamCharacterization:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_FuzzyFeatureTracking:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_GEMSe:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_GPU_DreamcasterTool:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_ModalityExplorer:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_ParameterExplorer:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_PolygonPrimitives:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_PorosityAnalyser:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_XRF:BOOL=ON")
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DQt5Charts_DIR:PATH=${Qt5_DIR}Charts")
endif()
if (BUILD_TESTING)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DopeniA_TESTING_ENABLED:BOOL=ON")
endif()
if (ENABLE_PRECOMPILE)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DopeniA_PRECOMPILE:BOOL=ON")
endif()
#if (BUILD_ASTRA)
#	SET (OPEN_IA_CMAKE_ARGS ${OPEN_IA_CMAKE_ARGS} "-DModule_AstraReconstruction:BOOL=ON")
#endif()
SET (openiA_DEPENDS ITK)
if (BUILD_VR)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DModule_VR:BOOL=ON"
		"-DOPENVR_ROOT_DIR:PATH=${OPENVR_BIN_DIR}")
endif()
if (BUILD_EIGEN)
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
	set_target_properties(open_iA PROPERTIES VS_DEBUGGER_COMMAND "${OPEN_IA_BIN_DIR_WIN}\\x64\\$(Configuration)\\open_iA.exe")
	STRING(REPLACE "/lib/cmake/Qt5" "/bin" Qt5_LIB_DIR ${Qt5_DIR})
	STRING(REPLACE "/" "\\" Qt5_LIB_DIR ${Qt5_LIB_DIR})
	STRING(REPLACE "/" "\\" VTK_WIN_DIR ${VTK_BIN_DIR})
	STRING(REPLACE "/" "\\" ITK_WIN_DIR ${ITK_BIN_DIR})
	set_target_properties(open_iA PROPERTIES VS_DEBUGGER_ENVIRONMENT "PATH=${Qt5_LIB_DIR};${VTK_WIN_DIR}\\bin\\$(Configuration);${ITK_WIN_DIR}\\bin\\$(Configuration)${ADDITIONAL_LIB_PATHS}")
endif()