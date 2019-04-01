cmake_minimum_required(VERSION 3.1)
project(open_iA_superbuild LANGUAGES CXX)

include(ExternalProject)

option (USE_GIT_REPOS "Whether to use git repositories for VTK and ITK libraries. If disabled (default), the release archives will be used instead." OFF)

set (VTK_SHORT_VERSION "8.2")
set (VTK_LONG_VERSION  "${VTK_SHORT_VERSION}.0")
set (VTK_ZIP_SHA256 "002d1fd708f2f3bfad3749dd061e4a8854a6efabcd70b0b14ca8be3c6ff4f257")
set (VTK_GZIP_SHA256 "34c3dc775261be5e45a8049155f7228b6bd668106c72a3c435d95730d17d57bb")

set (ITK_SHORT_VERSION "4.13")
set (ITK_LONG_VERSION  "${ITK_SHORT_VERSION}.2")
set (ITK_ZIP_SHA256 "b71e47262ded8d26f85ae50fc31334dad6c528dc5fa28dad8734d29a34b184ea")
set (ITK_XZ_SHA256 "dabe05234d24635c96fd9b1a5f5e3afc34e11f49a31e5107db19bf61da758fa5")

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
	set (VTK_ARCHIVE_SHA256 ${VTK_ZIP_SHA256})
	set (ITK_ARCHIVE_SHA256 ${ITK_ZIP_SHA256})
else()
	set (VTK_URL "https://www.vtk.org/files/release/${VTK_SHORT_VERSION}/VTK-${VTK_LONG_VERSION}.tar.gz")
	set (ITK_URL "https://netcologne.dl.sourceforge.net/project/itk/itk/${ITK_SHORT_VERSION}/InsightToolkit-${ITK_LONG_VERSION}.tar.xz")
	set (VTK_ARCHIVE_SHA256 ${VTK_GZIP_SHA256})
	set (ITK_ARCHIVE_SHA256 ${ITK_XZ_SHA256})
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
