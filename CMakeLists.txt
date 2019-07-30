cmake_minimum_required(VERSION 3.1)
project(open_iA_superbuild LANGUAGES CXX)

include(ExternalProject)

option (USE_GIT_REPOS "Whether to use git repositories for VTK and ITK libraries. If disabled (default), the release archives will be used instead." OFF)
#option (BUILD_ASTRA "Whether to build astra" OFF)
#option (USE_HDF5 "Whether to build HDF5 library and use it in open_iA" OFF)
#option (USE_EIGEN "Whether to fetch and use eigen" OFF)

set (VTK_SHORT_VERSION "8.2")
set (VTK_LONG_VERSION  "${VTK_SHORT_VERSION}.0")
set (VTK_ZIP_SHA256 "002d1fd708f2f3bfad3749dd061e4a8854a6efabcd70b0b14ca8be3c6ff4f257")
set (VTK_GZIP_SHA256 "34c3dc775261be5e45a8049155f7228b6bd668106c72a3c435d95730d17d57bb")

set (ITK_VERSION  "5.0.1")
set (ITK_ZIP_SHA512 "f1a871742247e9596b39126ee70b9ab5e6e3dc6fabfc54fc2f84d1f174b51f4f9086eae5769f6bc1a28eaf81183ebe77d66ca0199d65e65ef689ab9344bd81f2")
set (ITK_GZIP_SHA512 "f36613ff72c513ded3d32504f71308a94fe75555cf9fd22b77485d1375601f6e1f1539cc5ac82a9e1e229bcf514a88ccb55122a7dfc74a6ae1b6604aa70bd814")

set (OPEN_IA_GIT_TAG "a6372f8c9f1d9b1051345f10523ab2b9c306da09") # = "2019.07"

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
if (WIN32)
	SET (VTK_URL "${VTK_URL}.zip")
	set (ITK_URL "${ITK_URL}.zip")
	set (VTK_ARCHIVE_SHA256 ${VTK_ZIP_SHA256})
	set (ITK_ARCHIVE_SHA512 ${ITK_ZIP_SHA512})
else()
	set (VTK_URL "${VTK_URL}.tar.gz")
	set (ITK_URL "${ITK_URL}.tar.gz")
	set (VTK_ARCHIVE_SHA256 ${VTK_GZIP_SHA256})
	set (ITK_ARCHIVE_SHA512 ${ITK_GZIP_SHA512})
endif()

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
if (USE_GIT_REPOS)
	ExternalProject_Add( VTK
		GIT_REPOSITORY    "${VTK_REPOSITORY}"
		GIT_TAG           "${VTK_LONG_VERSION}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_LONG_VERSION}"
		BINARY_DIR        "${VTK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
		CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
		INSTALL_COMMAND   "")
else()
	ExternalProject_Add( VTK
		URL               ${VTK_URL}
		URL_HASH          "SHA256=${VTK_ARCHIVE_SHA256}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_LONG_VERSION}"
		BINARY_DIR        "${VTK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
		CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
		INSTALL_COMMAND   "")
endif()

set (ITK_CMAKE_ARGS 
	"-DBUILD_SHARED_LIBS:BOOL=On"
	"-DITK_USE_GPU:BOOL=On"
	"-DITK_USE_64BITS_IDS:BOOL=On"
	"-DModule_ITKReview:BOOL=On"
	"-DModule_ITKVtkGlue:BOOL=On"
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
ExternalProject_Add( open_iA
	GIT_REPOSITORY    "https://github.com/3dct/open_iA.git"
	GIT_TAG           "${OPEN_IA_GIT_TAG}"
	SOURCE_DIR        "${CMAKE_BINARY_DIR}/open_iA/src"
	BINARY_DIR        "${CMAKE_BINARY_DIR}/open_iA/bin"
	CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
	CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
	CMAKE_ARGS        "${OPEN_IA_CMAKE_ARGS}"
	INSTALL_COMMAND   ""
	DEPENDS           ITK)
