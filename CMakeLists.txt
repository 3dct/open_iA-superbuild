cmake_minimum_required(VERSION 3.13)
if (CMAKE_MAJOR_VERSION GREATER 3 OR (CMAKE_MAJOR_VERSION EQUAL 3 AND CMAKE_MINOR_VERSION GREATER 15))
	# prevent all submodules to be fetched in ExternalProject_Add if GIT_SUBMODULES is ""
	CMAKE_POLICY(SET CMP0097 NEW)
endif()
# 3.13 is required because we use VS_DEBUGGER_COMMAND and VS_DEBUGGER_ENVIRONMENT target properties introduced with that version
project(open_iA_superbuild LANGUAGES CXX)

include(ExternalProject)

option (VTK_USE_GIT_REPO "Whether to use git repository for VTK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly." OFF)
option (ITK_USE_GIT_REPO "Whether to use git repository for VTK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly." OFF)

option (ENABLE_ALL "Enables all optional modules and filters (see also separate ENABLE_xyz options)" OFF)
option (ENABLE_OPENCL "Enables OpenCL; the DreamCaster tool depends is only enabled if this setting is enabled; enabling this option also enables some GPU-optimized ITK filters" OFF)
option (ENABLE_FILTERS "Whether to build image processing filters (smoothing, segmentation, intensity transformations, geometric transformations, ...)" ON)
option (ENABLE_TOOLS "Whether to build common tool modules, e.g. FeatureScout, 4DCT, GEMSe, Dynamic Volume Lines, FIAKER, ..." ON)
option (ENABLE_TEST "Whether to enable build of tests runners and the capability to submit CDash test runs" OFF)
option (ENABLE_ASTRA "Whether to build ASTRA reconstruction library and open_iA ASTRA module. This will also fetch and build boost, so enabling it will considerably increase the build time!" OFF)
option (ENABLE_VR "Whether to enable VR module" OFF)
option (ENABLE_EIGEN "Whether to fetch and use eigen" OFF)
#option (ENABLE_HDF5 "Whether to build HDF5 library and use it in open_iA" OFF)
option (ENABLE_PRECOMPILE "Whether to build open_iA with precompiled headers enabled" OFF)

option (BUILD_VTK "Build VTK in the superbuild. If disabled, you need to set VTK_DIR to an existing VTK build" ON)
option (BUILD_ITK "Build ITK in the superbuild. If disabled, you need to set ITK_DIR to an existing ITK build" ON)
#option (BUILD_BOOST "Build Boost in the superbuild. If disabled, you need to set BOOST_DIR to an existing Astra Toolkit build" ON)
#option (FETCH_OPENVR "Fetches release of OpenVR SDK. If disabled, you need to set OPENVR_DIR to an existing OpenVR SDK directory" ON)
#option (FETCH_EIGEN "Fetches release of eigen. If disabled, you need to set EIGEN3_DIR to an existing Eigen 3.x directory" ON)
#option (BUILD_HDF5 "Build HDF5. If disabled, you need to set HDF5_DIR to an existing HDF5 build" ON)

if (ENABLE_ALL)
	SET (ENABLE_FILTERS "ON" CACHE BOOL "" FORCE)
	set (ENABLE_OPENCL "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_TOOLS "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_TEST "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_ASTRA "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_VR "ON" CACHE BOOL "" FORCE)
	SET (ENABLE_EIGEN "ON" CACHE BOOL "" FORCE)
	#SET (ENABLE_HDF5 "ON" CACHE BOOL "" FORCE)
endif()

FIND_PACKAGE(Qt5 COMPONENTS Widgets Xml Network Test OpenGL PrintSupport REQUIRED)


## VTK

set (VTK_RENDER_BACKEND_OPTIONS "None" "OpenGL" "OpenGL2")
list (FIND VTK_RENDER_BACKEND_OPTIONS "${VTK_RENDER_BACKEND}" backend_index)
if (${backend_index} EQUAL -1)
	set (VTK_RENDER_BACKEND_DEFAULT "OpenGL2")
	if (DEFINED VTK_RENDER_BACKEND)
		MESSAGE(WARNING "No valid VTK backend selected (VTK_RENDER_BACKEND), setting back to ${VTK_RENDER_BACKEND_DEFAULT}")
	endif()
	set (VTK_RENDER_BACKEND "${VTK_RENDER_BACKEND_DEFAULT}" CACHE STRING "Choose the render backend for VTK." FORCE)
	set_property(CACHE VTK_RENDER_BACKEND PROPERTY STRINGS ${VTK_RENDER_BACKEND_OPTIONS})
endif()

set (VTK_VERSION_OPTIONS "8.1.2" "8.2.0")
list (FIND VTK_VERSION_OPTIONS "${VTK_VERSION}" vtk_version_index)
if (${vtk_version_index} EQUAL -1)
	set (VTK_VERSION_DEFAULT "8.2.0")
	if (DEFINED VTK_VERSION)
		MESSAGE(WARNING "Invalid VTK_VERSION, resetting to default ${VTK_VERSION_DEFAULT}!")
	endif()
	set (VTK_VERSION "${VTK_VERSION_DEFAULT}" CACHE STRING "Choose the VTK version to build." FORCE)
	set_property(CACHE VTK_VERSION PROPERTY STRINGS ${VTK_VERSION_OPTIONS})
endif()
if ("${VTK_VERSION}" STREQUAL "8.2.0")
	set (VTK_ZIP_SHA512  "FD6582EA20A0B339EC847FEFA18956F9919535C6CBE1C5D443E2B3E302D6F6045C4250B0A89719EC4D4309E25F05449DACF37973615FC59BED33714EC6081BF9")
	set (VTK_GZIP_SHA512 "521BD4DABEDBC24B0E80A314A34ECD7554B04AF28A7973245E3A9CF99A09B995D1B8AC42305C8E53369F226A0A6DA3CDB29105BA2C90B46492736EF717760286")
	if ("${VTK_RENDER_BACKEND}" STREQUAL "OpenGL")
		MESSAGE(SEND_ERROR "OpenGL backend not available in VTK 8.2.0. Please either choose OpenGL2 backend or VTK version 8.1.2!")
	endif()
elseif ("${VTK_VERSION}" STREQUAL "8.1.2")
	set (VTK_ZIP_SHA512  "A6782D29C3E3998A9DAFCE317DD885718E55EC20E368E00C585DE910BA98B1625C6F5F589CAE55CDBC30CE98FEF22C094DA58D4E827485F6E2DBCEE0B96B76DE")
	set (VTK_GZIP_SHA512 "C9FC498804FF9BB433BF79F4B14D9BB6134B026AEA46ED12AC40434BFDD8063C42E75D3E0CFA88D171FA271001C9444D66A471EDC9B22DF83A9109733ABCDFA0")
endif()

STRING (SUBSTRING "${VTK_VERSION}" 0 3 VTK_SHORT_VERSION)
set (VTK_REPOSITORY "https://github.com/Kitware/VTK.git")
# set (VTK_REPOSITORY "https://gitlab.kitware.com/vtk/vtk.git")
set (VTK_URL "https://www.vtk.org/files/release/${VTK_SHORT_VERSION}/VTK-${VTK_VERSION}")

## ITK

set (ITK_VERSION_OPTIONS "4.13.2" "5.0.1")
list (FIND ITK_VERSION_OPTIONS "${ITK_VERSION}" itk_version_index)
if (${itk_version_index} EQUAL -1)
	set (ITK_VERSION_DEFAULT  "5.0.1")
	if (DEFINED ITK_VERSION)
		MESSAGE(WARNING "Invalid ITK_VERSION, resetting to default ${ITK_VERSION_DEFAULT}!")
	endif()
	set (ITK_VERSION "${ITK_VERSION_DEFAULT}" CACHE STRING "Choose the ITK version to build." FORCE)
	set_property(CACHE ITK_VERSION PROPERTY STRINGS ${ITK_VERSION_OPTIONS})
endif()
if ("${ITK_VERSION}" STREQUAL "5.0.1")
	set (ITK_ZIP_SHA512  "f1a871742247e9596b39126ee70b9ab5e6e3dc6fabfc54fc2f84d1f174b51f4f9086eae5769f6bc1a28eaf81183ebe77d66ca0199d65e65ef689ab9344bd81f2")
	set (ITK_GZIP_SHA512 "f36613ff72c513ded3d32504f71308a94fe75555cf9fd22b77485d1375601f6e1f1539cc5ac82a9e1e229bcf514a88ccb55122a7dfc74a6ae1b6604aa70bd814")
elseif ("${ITK_VERSION}" STREQUAL "4.13.2")
	set (ITK_ZIP_SHA512  "AB4934953E662FE7F039910FE3DA79C83784026970A0779A8148847401308077B672B5107861FBFBCC8F852249F76BB9A8C1615D28813EE375AC1B3371BC27DB")
	set (ITK_GZIP_SHA512 "053E7109D3BBB0FF9191AF4A3650D58B35BC486D7629E3351D09A67A52379681E9FEC960C8235B56915E2E039503B31439E789A1A2EE306BA7EF67413DAAEA78")
endif()
set (ITK_REPOSITORY "https://github.com/InsightSoftwareConsortium/ITK.git")
# set (ITK_REPOSITORY "https://github.com/Kitware/ITK.git")
# set (ITK_REPOSITORY "http://itk.org/ITK.git")
set (ITK_URL "https://github.com/InsightSoftwareConsortium/ITK/releases/download/v${ITK_VERSION}/InsightToolkit-${ITK_VERSION}")


# TODO:
#   - Download/install Qt?
#   - check behavior when there are changes in git (and tag not the one referenced in ExternalProject_Add) / offline
#   - astra
#   - hdf5
#   - eigen

set (OPENVR_VERSION "1.9.15")
set (OPENVR_URL "https://github.com/ValveSoftware/openvr/archive/v${OPENVR_VERSION}.zip")
set (OPENVR_ARCHIVE_SHA512 "CB047D92C383F5CF761FD5EE262DDA8755BF592AC827A8C7314294DDBA489E4709EB3AA756207382E1C16BB02DAED77EAA6CEBE48411BCE5A4C2957855FA75D9")

set (ASTRA_REPOSITORY "https://github.com/astra-toolbox/astra-toolbox.git")
set (ASTRA_TAG        "06322245a638e435d29bc8f027aed0bd976139d0") # version "1.9.9dev"
set (BOOST_VERSION    "1.72.0")
set (BOOST_ARCHIVE_SHA512 "BEB4D099E9427394D6AD0CE0D526EE2C6E96CC4CAF5DE4747D009B01CD3E517A988E79C0D4522E49217C6A7EFD551075FB18BFAC51F344CEA9978CDCFC57CC56") # for 1.72.0 zip
if (MSVC)
	if (MSVC_VERSION LESS 1900)
		message(SEND_ERROR "Visual Studio < 2015 is not supported, please use a newer version!")
	elseif (MSVC_VERSION LESS 1910)
		set (MSVC_TOOLSET "v140")
		set (BOOST_MSVC_TOOLSET "msvc-14.0")
		set (BOOST_LIB_SUFFIX "vc140")
	elseif (MSVC_VERSION LESS 1920)
		set (MSVC_TOOLSET "v141")
		set (BOOST_MSVC_TOOLSET "msvc-14.1")
		set (BOOST_LIB_SUFFIX "vc141")
	elseif (MSVC_VERSION LESS 1930)
		set (MSVC_TOOLSET "v142")
		set (BOOST_MSVC_TOOLSET "msvc-14.2")
		set (BOOST_LIB_SUFFIX "vc142")
	else()
		message(SEND_ERROR "Visual Studio > 2019 is not supported (yet), please use an older version!")
	endif()
endif()
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
set (ITK_DEPENDS "")

if (BUILD_VTK)
	set (VTK_CMAKE_ARGS
		"-DBUILD_SHARED_LIBS:BOOL=On"
		"-DBUILD_TESTING:BOOL=Off"
		"-DVTK_LEGACY_REMOVE:BOOL=On"
		"-DModule_vtkGUISupportQt:BOOL=On"
		"-DModule_vtkGUISupportQtOpenGL:BOOL=On"
		"-DModule_vtkRenderingQt:BOOL=On"
		"-DModule_vtkViewsQt:BOOL=On"
		"-DQt5_DIR:PATH=${Qt5_DIR}")
	LIST(APPEND VTK_CMAKE_ARGS "-DVTK_RENDER_BACKEND:STRING=${VTK_RENDER_BACKEND}")
	set (VTK_DIR "${CMAKE_BINARY_DIR}/vtk/bin-${VTK_VERSION}" CACHE PATH
		"VTK build folder." FORCE)
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
	IF (VTK_USE_GIT_REPO)
		ExternalProject_Add( VTK
			GIT_REPOSITORY    "${VTK_REPOSITORY}"
			GIT_TAG           "v${VTK_VERSION}"
			#UPDATE_COMMAND    ""  # to force cmake to stick to the tag specified above, otherwise it will always go to latest 'release' branch commit
			SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src.git"
			BINARY_DIR        "${VTK_DIR}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
			INSTALL_COMMAND   ""
			DEPENDS           ${VTK_DEPENDS})
	else()
		ExternalProject_Add( VTK
			URL               ${VTK_URL}
			URL_HASH          "SHA512=${VTK_ARCHIVE_SHA512}"
			SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_VERSION}"
			BINARY_DIR        "${VTK_DIR}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
			INSTALL_COMMAND   ""
			DEPENDS           ${VTK_DEPENDS})
	endif()
	set (ITK_DEPENDS VTK)
else()
	set (VTK_DIR "${VTK_DIR}" CACHE PATH
		"VTK build folder.")
endif()

if (ENABLE_OPENCL)
	option (BUILD_OPENCL "Build OpenCL ICD loader. If disabled, you need to set OPENCL_DIR to an existing OpenCL ICD loader build" ON)
	if (BUILD_OPENCL)
		set (OPENCL_DIR "${CMAKE_BINARY_DIR}/OpenCL" CACHE PATH
			"OpenCL ICD loader directory" FORCE)
		set (OPENCL_C_HEADERS_REPO "https://github.com/KhronosGroup/OpenCL-Headers.git")
		set (OPENCL_CPP_HEADERS_REPO "https://github.com/KhronosGroup/OpenCL-CLHPP.git")
		set (OPENCL_ICD_REPO "https://github.com/KhronosGroup/OpenCL-ICD-Loader.git")
		set (OPENCL_TAG "v2020.03.13")
		set (OPENCL_CPP_TAG "v2.0.11")
		ExternalProject_Add (OpenCL_C
			GIT_REPOSITORY    ${OPENCL_C_HEADERS_REPO}
			GIT_TAG           "${OPENCL_TAG}"
			SOURCE_DIR        "${OPENCL_DIR}/cheaders"
			BUILD_IN_SOURCE   1
			CONFIGURE_COMMAND ""
			BUILD_COMMAND     ""
			INSTALL_COMMAND   "")
		ExternalProject_Add (OpenCL_CPP
			GIT_REPOSITORY    ${OPENCL_CPP_HEADERS_REPO}
			GIT_TAG           "${OPENCL_CPP_TAG}"
			GIT_SUBMODULES    ""
			SOURCE_DIR        "${OPENCL_DIR}/cppheaders"
			BUILD_IN_SOURCE   1
			CONFIGURE_COMMAND ""
			BUILD_COMMAND     ""
			INSTALL_COMMAND   "")
		# Find way to skip re-build of OpenCL every time!
		
		# _fetch project is there to clone OpenCL repository
		# after that, headers from C/CPP projects need to be copied into repo;
		# that's what happens in pre_build step for OpenCL_ICD project below
		ExternalProject_Add (OpenCL_ICD_fetch
			GIT_REPOSITORY    ${OPENCL_ICD_REPO}
			GIT_TAG          "${OPENCL_TAG}"
			GIT_SUBMODULES    ""
			SOURCE_DIR        "${OPENCL_DIR}/icdloader"
			BUILD_IN_SOURCE   1
			CONFIGURE_COMMAND ""
			BUILD_COMMAND     ""
			INSTALL_COMMAND   ""
			DEPENDS           OpenCL_CPP OpenCL_C)
		ExternalProject_Add (OpenCL_ICD
			DOWNLOAD_COMMAND  "" # to make CMake ignore not-yet existing SOURCE_DIR
			SOURCE_DIR        "${OPENCL_DIR}/icdloader"
			BINARY_DIR        "${OPENCL_DIR}/icd-bin"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "-DOPENCL_ICD_LOADER_REQUIRE_WDK:BOOL=OFF"
			INSTALL_COMMAND   ""
			DEPENDS           OpenCL_ICD_fetch)
			
		add_custom_command(TARGET OpenCL_ICD PRE_BUILD
			COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/cheaders/CL" "${OPENCL_DIR}/icdloader/inc/CL")
		add_custom_command(TARGET OpenCL_ICD PRE_BUILD
			COMMAND ${CMAKE_COMMAND} -E copy "${OPENCL_DIR}/cppheaders/include/CL/cl.hpp" "${OPENCL_DIR}/cppheaders/include/CL/cl2.hpp" "${OPENCL_DIR}/icdloader/inc/CL/")
		add_custom_command(TARGET OpenCL_ICD POST_BUILD
			COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/icdloader/inc/CL" "${OPENCL_DIR}/include/CL")
		# Find way to use different configurations properly...?
		add_custom_command(TARGET OpenCL_ICD POST_BUILD
			COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/icd-bin/$(Configuration)" "${OPENCL_DIR}/lib")
		LIST (APPEND ITK_DEPENDS OpenCL_ICD)
	else()
		set(OPENCL_DIR "${OPENCL_DIR}" CACHE PATH
			"OpenCL ICD loader directory")
	endif()
	SET (OPENCL_INCLUDE_DIRS "${OPENCL_DIR}/include")
	SET (OPENCL_LIBRARIES "${OPENCL_DIR}/lib/${CMAKE_SHARED_LIBRARY_PREFIX}OpenCL")
	if (WIN32)
		SET (OPENCL_LIBRARIES "${OPENCL_LIBRARIES}.lib") # only Windows has the peculiarity of requiring to link against the stub .lib file
	else()
		SET (OPENCL_LIBRARIES "${OPENCL_LIBRARIES}${CMAKE_SHARED_LIBRARY_SUXFIX}")
	endif()

	if (NOT OPENCL_DIR)
		message(SEND_ERROR "OPENCL_DIR not set! Either enable BUILD_OPENCL or set OPENCL_DIR to a valid OpenCL ICD loader directory!")
	endif()
endif()

SET (openiA_DEPENDS "")
if (BUILD_ITK)
	set (ITK_CMAKE_ARGS
		"-DBUILD_SHARED_LIBS:BOOL=On"
		"-DBUILD_TESTING:BOOL=Off"
		"-DITK_USE_GPU:BOOL=${ENABLE_OPENCL}"
		"-DITK_USE_64BITS_IDS:BOOL=On"
		"-DModule_ITKReview:BOOL=On"
		"-DModule_ITKVtkGlue:BOOL=On"
		"-DModule_HigherOrderAccurateGradient:BOOL=On"
		"-DModule_SplitComponents:BOOL=On"
		"-DVTK_DIR:PATH=${VTK_DIR}"
		"-DQt5_DIR:PATH=${Qt5_DIR}")
	if (ENABLE_OPENCL)
		LIST (APPEND ITK_CMAKE_ARGS
		"-DOPENCL_ROOT_DIR=${OPENCL_DIR}"
		"-DOPENCL_INCLUDE_DIRS=${OPENCL_INCLUDE_DIRS}"
		"-DOPENCL_LIBRARIES=${OPENCL_LIBRARIES}")
	endif()
	set (ITK_DIR "${CMAKE_BINARY_DIR}/itk/bin-${ITK_VERSION}" CACHE PATH "ITK build folder." FORCE)
	if (ITK_USE_GIT_REPO)
		ExternalProject_Add( ITK
			GIT_REPOSITORY    "${ITK_REPOSITORY}"
			GIT_TAG           "v${ITK_VERSION}"
			SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src.git"
			BINARY_DIR        "${ITK_DIR}"
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
			BINARY_DIR        "${ITK_DIR}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
			INSTALL_COMMAND   ""
			DEPENDS           ${ITK_DEPENDS})
	endif()
	SET (openiA_DEPENDS ITK)
else()
	set (ITK_DIR "${ITK_DIR}" CACHE PATH
		"ITK build folder.")
endif()

if (ENABLE_ASTRA)
	option (BUILD_ASTRA "Build Astra Toolbox in the superbuild. If disabled, you need to set ASTRA_DIR to an existing Astra Toolkit build" ON)
	if (BUILD_ASTRA)
		set (ASTRA_DIR "${CMAKE_BINARY_DIR}/astra" CACHE PATH
			"Astra toolbox directory" FORCE)
		if (WIN32)
			option (ASTRA_PATCH "Whether to patch Astra Toolbox project files. Only required to be set first time sources are fetched, disable later." ON)
		endif()
		FIND_PACKAGE(CUDA REQUIRED)
		if (CUDA_FOUND)
			MESSAGE(STATUS "CUDA: ${CUDA_VERSION} in ${CUDA_TOOLKIT_ROOT_DIR}.")
		endif()
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
		SET (BOOST_DIR "${CMAKE_BINARY_DIR}/boost")
		ExternalProject_Add( boost
			URL               ${BOOST_URL}
			URL_HASH          SHA512=${BOOST_ARCHIVE_SHA512}
			BUILD_IN_SOURCE   1
			SOURCE_DIR        "${BOOST_DIR}"
			CONFIGURE_COMMAND ${BOOTSTRAP_COMMAND}
			BUILD_COMMAND     ${B2_COMMAND} ${B2_FLAGS}
			INSTALL_COMMAND   ""  # ${B2_COMMAND} ${B2_FLAGS} install
		)
		if (WIN32)
			CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/vs.patch.template" "${CMAKE_BINARY_DIR}/vs.patch")
			# find way to only do this once:
			if (ASTRA_PATCH)
				set (ASTRA_PATCH_COMMAND "${GIT_EXECUTABLE}" apply "${CMAKE_BINARY_DIR}/vs.patch")
			else()
				set (ASTRA_PATCH_COMMAND "")
			endif()
			set (ASTRA_CONFIGURE_COMMAND "")
			set (ASTRA_BUILD_COMMAND "${CMAKE_MAKE_PROGRAM}" "${ASTRA_DIR}/astra_vc14.sln" -t:astra_vc14 -p:Configuration=Release_CUDA -p:Platform=x64)
		else()
			set (ASTRA_PATCH_COMMAND "")
			set (ASTRA_BUILD_DIR "${ASTRA_DIR}/build/linux")
			set (ASTRA_CONFIGURE_COMMAND "${ASTRA_BUILD_DIR}/autogen.sh && ${ASTRA_BUILD_DIR}/configure")
			set (ASTRA_BUILD_COMMAND "cd ${ASTRA_BUILD_DIR} && make -j 8")
		endif()
		# fetch and build astra:
		ExternalProject_Add( astra
			GIT_REPOSITORY    "${ASTRA_REPOSITORY}"
			GIT_TAG           "${ASTRA_TAG}"
			SOURCE_DIR        "${ASTRA_DIR}"
			BUILD_IN_SOURCE   1
			PATCH_COMMAND     "${ASTRA_PATCH_COMMAND}"
			CONFIGURE_COMMAND "${ASTRA_CONFIGURE_COMMAND}"
			BUILD_COMMAND     "${ASTRA_BUILD_COMMAND}"
			INSTALL_COMMAND   ""
			DEPENDS           boost
		)
		#find way to only do this only once
		add_custom_command(TARGET boost POST_BUILD
			COMMAND ${CMAKE_COMMAND} -E copy_directory "${BOOST_DIR}/boost" "${ASTRA_DIR}/lib/include/boost")
		add_custom_command(TARGET boost POST_BUILD
			COMMAND ${CMAKE_COMMAND} -E make_directory "${ASTRA_DIR}/lib/x64")
		set (BOOST_LIB_BASE "${BOOST_DIR}/stage/lib/libboost_")
		set (BOOST_SUFF1 "-${BOOST_LIB_SUFFIX}-mt-gd-x64-${BOOST_SHORTVER}.lib")
		set (BOOST_SUFF2 "-${BOOST_LIB_SUFFIX}-mt-x64-${BOOST_SHORTVER}.lib")
		add_custom_command(TARGET boost POST_BUILD
			COMMAND ${CMAKE_COMMAND} -E copy "${BOOST_LIB_BASE}chrono${BOOST_SUFF1}" "${BOOST_LIB_BASE}chrono${BOOST_SUFF2}" "${BOOST_LIB_BASE}date_time${BOOST_SUFF1}" "${BOOST_LIB_BASE}date_time${BOOST_SUFF2}" "${BOOST_LIB_BASE}system${BOOST_SUFF1}" "${BOOST_LIB_BASE}system${BOOST_SUFF2}" "${BOOST_LIB_BASE}thread${BOOST_SUFF1}" "${BOOST_LIB_BASE}thread${BOOST_SUFF2}" "${ASTRA_DIR}/lib/x64"
		)
	else()
		set(ASTRA_DIR "${ASTRA_DIR}" CACHE PATH
			"Astra toolbox directory")
	endif()
endif()


set (OPEN_IA_CMAKE_ARGS
	"-DITK_DIR:PATH=${ITK_DIR}")
if (ENABLE_OPENCL)
	LIST (APPEND OPEN_IA_CMAKE_ARGS
	"-DOPENCL_ROOT_DIR=${OPENCL_DIR}"
	"-DOPENCL_INCLUDE_DIRS=${OPENCL_INCLUDE_DIRS}"
	"-DOPENCL_LIBRARIES=${OPENCL_LIBRARIES}")
endif()
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
		"-DModule_DreamCaster:BOOL=${ENABLE_OPENCL}"
		"-DModule_DynamicVolumeLines:BOOL=ON"
		"-DModule_FeatureScout:BOOL=ON"
		"-DModule_FiAKEr:BOOL=ON"
		"-DModule_FoamCharacterization:BOOL=ON"
		"-DModule_FuzzyFeatureTracking:BOOL=ON"
		"-DModule_GEMSe:BOOL=ON"
		"-DModule_ModalityExplorer:BOOL=ON"
		"-DModule_ParameterExplorer:BOOL=ON"
		"-DModule_PolygonPrimitives:BOOL=ON"
		"-DModule_PorosityAnalyser:BOOL=ON"
		"-DModule_TripleHistogramTF:BOOL=ON"
		"-DModule_Uncertainty:BOOL=ON"
		"-DModule_XRF:BOOL=ON"
		"-DQt5Charts_DIR:PATH=${Qt5_DIR}Charts")
endif()
if (ENABLE_TEST)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DopeniA_TESTING_ENABLED:BOOL=ON")
endif()
if (ENABLE_PRECOMPILE)
	LIST (APPEND OPEN_IA_CMAKE_ARGS "-DopeniA_PRECOMPILE:BOOL=ON")
endif()
if (ENABLE_ASTRA)
	LIST (APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_AstraReconstruction:BOOL=ON"
		"-DASTRA_TOOLBOX_DIR:PATH=${ASTRA_DIR}")
	if (BUILD_ASTRA)
		LIST (APPEND openiA_DEPENDS astra)
	endif()
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


set (OPEN_IA_GIT_TAG_OPTIONS "develop" "master" "2019.12" "2019.07")
list (FIND OPEN_IA_GIT_TAG_OPTIONS "${OPEN_IA_GIT_TAG}" open_ia_tag_index)
if (${open_ia_tag_index} EQUAL -1)
	set (OPEN_IA_GIT_TAG_DEFAULT "develop")
	if (DEFINED OPEN_IA_GIT_TAG)
		MESSAGE(WARNING "Invalid OPEN_IA_GIT_TAG, resetting to default ${OPEN_IA_GIT_TAG_DEFAULT}!")
	endif()
	set (OPEN_IA_GIT_TAG "${OPEN_IA_GIT_TAG_DEFAULT}" CACHE STRING "Choose the open_iA version to build." FORCE)
	set_property(CACHE OPEN_IA_GIT_TAG PROPERTY STRINGS ${OPEN_IA_GIT_TAG_OPTIONS})
endif()
if ("${OPEN_IA_GIT_TAG}" STREQUAL "2019.07" AND "${ITK_VERSION}" STREQUAL "5.0.1")
	MESSAGE(SEND_ERROR "open_iA version ${OPEN_IA_GIT_TAG} is not compatible with ITK version ${ITK_VERSION}!")
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
	STRING(REPLACE "/" "\\" VTK_WIN_DIR ${VTK_DIR})
	STRING(REPLACE "/" "\\" ITK_WIN_DIR ${ITK_DIR})
	set (openia_cmd  "${OPEN_IA_BIN_DIR_WIN}\\x64\\$(Configuration)\\open_iA.exe")
	set (openia_path "${Qt5_LIB_DIR};${VTK_WIN_DIR}\\bin\\$(Configuration);${ITK_WIN_DIR}\\bin\\$(Configuration)${ADDITIONAL_LIB_PATHS}")
	set_target_properties(open_iA PROPERTIES VS_DEBUGGER_COMMAND "${openia_cmd}")
	set_target_properties(open_iA PROPERTIES VS_DEBUGGER_ENVIRONMENT "PATH=${openia_path}")
	set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT open_iA)
endif()