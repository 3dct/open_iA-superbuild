cmake_minimum_required(VERSION 3.1)
project(open_iA_superbuild LANGUAGES CXX)

include(ExternalProject)

option (USE_GIT_REPOS "Whether to use git repositories for VTK and ITK libraries. If disabled (default), the release archives will be used instead." OFF)

set (VTK_SHORT_VERSION "8.1")
set (VTK_LONG_VERSION  "${VTK_SHORT_VERSION}.2")

set (ITK_SHORT_VERSION "4.13")
set (ITK_LONG_VERSION  "${ITK_SHORT_VERSION}.1")

FIND_PACKAGE(Qt5 COMPONENTS Widgets Xml Network Test OpenGL PrintSupport REQUIRED)

set (VTK_REPOSITORY "https://github.com/Kitware/VTK.git")
# set (VTK_REPOSITORY "https://gitlab.kitware.com/vtk/vtk.git")
set (ITK_REPOSITORY "https://github.com/InsightSoftwareConsortium/ITK.git")
# set (ITK_REPOSITORY "https://github.com/Kitware/ITK.git")
# set (ITK_REPOSITORY "http://itk.org/ITK.git")

# TODO:
#   - Think about calling Qt installer?
#   - check behavior when there are changes in git (and tag not the one referenced in ExternalProject_Add) / offline
if (WIN32)
	set (VTK_URL "https://www.vtk.org/files/release/${VTK_SHORT_VERSION}/VTK-${VTK_LONG_VERSION}.zip")
	set (ITK_URL "https://netcologne.dl.sourceforge.net/project/itk/itk/${ITK_SHORT_VERSION}/InsightToolkit-${ITK_LONG_VERSION}.zip")
	set (VTK_ARCHIVE_SHA256 "5B7B1C7D3CA1E2F8F02E891800F64627A894673EB5B47354EECC28FCBEF4815E")
	set (ITK_ARCHIVE_SHA256 "7F3AE1200604608D5E8C2066CC60B7ED836C7DB56F670544D8B88829A78D92A4")
else()
	set (VTK_URL "https://www.vtk.org/files/release/${VTK_SHORT_VERSION}/VTK-${VTK_LONG_VERSION}.tar.gz")
	set (ITK_URL "https://netcologne.dl.sourceforge.net/project/itk/itk/${ITK_SHORT_VERSION}/InsightToolkit-${ITK_LONG_VERSION}.tar.xz")
	set (VTK_ARCHIVE_SHA256 "0995FB36857DD76CCFB8BB07350C214D9F9099E80B1E66B4A8909311F24FF0DB")
	set (ITK_ARCHIVE_SHA256 "54890862CD1FB420C86EB24F2B229E18FDAEBC48E0398BBE9C564BBBDED58C5C")
endif()

set (VTK_CMAKE_ARGS
	"-DBUILD_SHARED_LIBS:BOOL=On"
	"-DVTK_LEGACY_REMOVE:BOOL=On"
	"-DModule_vtkGUISupportQt:BOOL=On"
	"-DModule_vtkGUISupportQtOpenGL:BOOL=On"
	"-DModule_vtkRenderingQt:BOOL=On"
	"-DModule_vtkViewsQt:BOOL=On"
	"-DVTK_RENDERING_BACKEND:STRING=OpenGL2"
	"-DQt5_DIR:PATH=${Qt5_DIR}")
IF (APPLE)
	LIST(APPEND VTK_CMAKE_ARGS "-DVTK_RENDERING_BACKEND:STRING=OpenGL")
endif()
set (VTK_BIN_DIR "${CMAKE_BINARY_DIR}/vtk/bin-${VTK_LONG_VERSION}")
if (USE_GIT_REPOS)
	ExternalProject_Add( VTK
		GIT_REPOSITORY    "${VTK_REPOSITORY}"
		GIT_TAG           "${VTK_LONG_VERSION}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_LONG_VERSION}"
		BINARY_DIR        "${VTK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
		INSTALL_COMMAND   "")
else()
	ExternalProject_Add( VTK
		URL               ${VTK_URL}
		URL_HASH          "SHA256=${VTK_ARCHIVE_SHA256}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_LONG_VERSION}"
		BINARY_DIR        "${VTK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
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
set (ITK_BIN_DIR "${CMAKE_BINARY_DIR}/itk/bin-${ITK_LONG_VERSION}")
if (USE_GIT_REPOS)
	ExternalProject_Add( ITK
		GIT_REPOSITORY    "${ITK_REPOSITORY}"
		GIT_TAG           "${ITK_LONG_VERSION}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src-${ITK_LONG_VERSION}"
		BINARY_DIR        "${ITK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
		INSTALL_COMMAND   ""
		DEPENDS           VTK)
else()
	ExternalProject_Add( ITK
		URL               "${ITK_URL}"
		URL_HASH          "SHA256=${ITK_ARCHIVE_SHA256}"
		SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src-${ITK_LONG_VERSION}"
		BINARY_DIR        "${ITK_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
		INSTALL_COMMAND   ""
		DEPENDS           VTK)
endif()

set (OPEN_IA_CMAKE_ARGS
	"-DITK_DIR:PATH=${ITK_BIN_DIR}")
ExternalProject_Add( open_iA
	GIT_REPOSITORY    "https://github.com/3dct/open_iA.git"
	GIT_TAG           4623f34d9ae1ecddef36975e3bff3d36a5d2039d
	SOURCE_DIR        "${CMAKE_BINARY_DIR}/open_iA/src"
	BINARY_DIR        "${CMAKE_BINARY_DIR}/open_iA/bin"
	CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
	CMAKE_ARGS        "${OPEN_IA_CMAKE_ARGS}"
	INSTALL_COMMAND   ""
	DEPENDS           ITK)
