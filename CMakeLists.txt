cmake_minimum_required(VERSION 3.1)
project(open_iA_superbuild LANGUAGES CXX)

include(ExternalProject)

SET (CMAKE_GENERATOR "Unix Makefiles")
option (BUILD_SHARED_LIBS "Build shared libraries" ON)

set (VTK_SHORT_VERSION "8.1")
set (VTK_LONG_VERSION  "${VTK_SHORT_VERSION}.2")

set (ITK_SHORT_VERSION "4.13")
set (ITK_LONG_VERSION  "${ITK_SHORT_VERSION}.1")

set (VTK_URL "https://www.vtk.org/files/release/${VTK_SHORT_VERSION}/VTK-${VTK_LONG_VERSION}.tar.gz")
set (ITK_URL "https://sourceforge.net/projects/itk/files/itk/${ITK_SHORT_VERSION}/InsightToolkit-${ITK_LONG_VERSION}.tar.xz/download")

set (VTK_CMAKE_ARGS
	"-DBUILD_SHARED_LIBS=On"
	"-DVTK_LEGACY_REMOVE=On"
	"-DModule_vtkGUISupportQt=On"
	"-DModule_vtkGUISupportQtOpenGL=On"
	"-DModule_vtkRenderingQt=On"
	"-DModule_vtkViewsQt=On"
	"-DVTK_RENDERING_BACKEND=OpenGL2"
	"-DVTK_Qt_VERSION=5")
set (VTK_BIN_DIR "${CMAKE_BINARY_DIR}/vtk/bin-${VTK_LONG_VERSION}")
ExternalProject_Add( VTK
	URL	              ${VTK_URL}
	SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_LONG_VERSION}"
	BINARY_DIR        "${VTK_BIN_DIR}"
	CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
	CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
	INSTALL_COMMAND   "")

set (ITK_CMAKE_ARGS 
    "-DBUILD_SHARED_LIBS:BOOL=On"
	"-DITK_USE_GPU:BOOL=On"
	"-DITK_USE_64BITS_IDS:BOOL=On"
	"-DModule_ITKReview:BOOL=On"
	"-DModule_ITKVtkGlue:BOOL=On"
	"-DVTK_DIR=${VTK_BIN_DIR}")
set (ITK_BIN_DIR "${CMAKE_BINARY_DIR}/itk/bin-${ITK_LONG_VERSION}")
ExternalProject_Add( ITK
	URL	              ${ITK_URL}
	SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src-${ITK_LONG_VERSION}"
	BINARY_DIR        "${ITK_BIN_DIR}"
	CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
	CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
	INSTALL_COMMAND   ""
	DEPENDS           VTK)

set (OPEN_IA_CMAKE_ARGS
	"-DITK_DIR=${ITK_BIN_DIR}")
ExternalProject_Add( open_iA
	GIT_REPOSITORY    "https://github.com/3dct/open_iA.git"
	GIT_TAG           4623f34d9ae1ecddef36975e3bff3d36a5d2039d
	SOURCE_DIR        "${CMAKE_BINARY_DIR}/open_iA/src"
	BINARY_DIR        "${CMAKE_BINARY_DIR}/open_iA/bin"
	CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
	CMAKE_ARGS        "${OPEN_IA_CMAKE_ARGS}"
	INSTALL_COMMAND   ""
	DEPENDS           ITK)
