cmake_minimum_required(VERSION 3.14)
# Required CMake version: as high as open_iA!
if (CMAKE_MAJOR_VERSION GREATER 3 OR (CMAKE_MAJOR_VERSION EQUAL 3 AND CMAKE_MINOR_VERSION GREATER 15))
	# prevent all submodules to be fetched in ExternalProject_Add if GIT_SUBMODULES is ""
	CMAKE_POLICY(SET CMP0097 NEW)
endif()
project(open_iA_superbuild LANGUAGES CXX)

include(ExternalProject)

option(VTK_USE_GIT_REPO "Whether to use git repository for VTK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly." OFF)
option(ITK_USE_GIT_REPO "Whether to use git repository for ITK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly." OFF)

option(ENABLE_ALL "Enables all optional modules and filters (see also separate ENABLE_xyz options; all except for ENABLE_PRECOMPILE are enabled if this is set to on. Note that unchecking this box again does NOT have any direct effect; it will not automatically set these options to unchecked or their state before. But you will have to uncheck the option if you want to disable any of the single ENABLE_xyz options affected by this setting, otherwise they will be re-enabled on next 'Configure' run) (default: disabled)" OFF)

option(ENABLE_AI "Whether to enable AI module; requires onnx runtime, which will be fetched automatically; on Windows, you can change whether CUDA or DirectML backend is chosen with the option AI_ONNX_USE_CUDA (default: disabled)" OFF)
option(ENABLE_ASTRA "Whether to build ASTRA reconstruction library and open_iA ASTRA module. This will also fetch and build boost, so enabling it will considerably increase the build time! (default: disabled)" OFF)
option(ENABLE_EIGEN "Whether to fetch and use eigen (default: disabled)" OFF)
option(ENABLE_FILTERS "Whether to build image processing filters (smoothing, segmentation, intensity transformations, geometric transformations, ...) (default: enabled)" ON)
option(ENABLE_HDF5 "Whether to fetch and build HDF5 library and use it in open_iA (default: disabled)" OFF)
option(ENABLE_OPENCL "Enables OpenCL; the DreamCaster tool depends is only enabled if this setting is enabled; enabling this option also enables some GPU-optimized ITK filters (default: disabled)" OFF)
option(ENABLE_PRECOMPILE "Whether to build open_iA with precompiled headers enabled (default: disabled; NOT included in \"ENABLE_ALL\")" OFF)
option(ENABLE_TOOLS "Whether to build common tool modules, e.g. FeatureScout, 4DCT, GEMSe, Dynamic Volume Lines, FIAKER, ... (default: enabled)" ON)
option(ENABLE_TEST "Whether to enable build of tests runners and the capability to submit CDash test runs (default: disabled)" OFF)
option(ENABLE_VR "Enable VR module; requires OpenVR SDK, which will be fetched automatically; also boost (includes) are required (default: disabled)" OFF)

option(BUILD_VTK "Build VTK in the superbuild. If disabled, you need to set VTK_DIR to an existing VTK build (default: enabled)" ON)
option(BUILD_ITK "Build ITK in the superbuild. If disabled, you need to set ITK_DIR to an existing ITK build (default: enabled)" ON)
#option(FETCH_OPENVR "Fetch release of OpenVR SDK. If disabled, you need to set OPENVR_DIR to an existing OpenVR SDK directory" ON)
#option(FETCH_EIGEN "Fetch release of eigen. If disabled, you need to set EIGEN3_DIR to an existing Eigen 3.x directory" ON)
#option(BUILD_HDF5 "Fetch and build HDF5. If disabled, you need to set HDF5_DIR to an existing HDF5 build" ON)
#option(FETCH_ONNX "Fetch required ONNX runtime. If disabled, you need to set ONNX_RUNTIME_DIR to an existing ONNX runtime directory" ON) # could be a bit complicated to implement, since we would have to check the type of runtime (CUDA/DirectML/CPU) and set according flags

option(DISABLE_LIB_WARNINGS "Disables all compilation warnings on the built libraries" OFF)

# Find proper way to append flags here...
#if(MSVC)
#	set(DISABLE_WARNINGS_OPT "\"-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} /W0\"")
#else()
#	set(DISABLE_WARNINGS_OPT "\"-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} -w\"")
#endif()
#
if (NOT ARCHIVE_DIR)
	set(ARCHIVE_DIR ${CMAKE_BINARY_DIR}/archives CACHE PATH "Directory for storing downloaded archive files." FORCE)
endif()

if (ENABLE_ALL)
	set(ENABLE_AI "ON" CACHE BOOL "" FORCE)
	set(ENABLE_ASTRA "ON" CACHE BOOL "" FORCE)
	set(ENABLE_EIGEN "ON" CACHE BOOL "" FORCE)
	set(ENABLE_FILTERS "ON" CACHE BOOL "" FORCE)
	set(ENABLE_HDF5 "ON" CACHE BOOL "" FORCE)
	set(ENABLE_OPENCL "ON" CACHE BOOL "" FORCE)
	set(ENABLE_TOOLS "ON" CACHE BOOL "" FORCE)
	set(ENABLE_TEST "ON" CACHE BOOL "" FORCE)
	set(ENABLE_VR "ON" CACHE BOOL "" FORCE)
endif()

if (ENABLE_ASTRA)
	option(BUILD_ASTRA "Build Astra Toolbox in the superbuild. If disabled, you need to set ASTRA_DIR to an existing Astra Toolkit build" ON)
endif()

set(BUILD_BOOST_OPTION OFF)
if (ENABLE_ASTRA OR ENABLE_VR)
	set(BUILD_BOOST_OPTION ON)
endif()
if (BUILD_BOOST_OPTION)
	if (WIN32)
		set(BUILD_BOOST_DEFAULT_VAL ON)
	else()
		set(BUILD_BOOST_DEFAULT_VAL OFF)
	endif()
	option(BUILD_BOOST "Fetch and build Boost in the superbuild, if astra is enabled. If disabled, you need to set BOOST_DIR to an existing boost build (default: enabled)" ${BUILD_BOOST_DEFAULT_VAL})
endif()

message(STATUS "Compilers: C++: ${CMAKE_CXX_COMPILER}, C: ${CMAKE_C_COMPILER}")

if (WIN32 AND NOT MSVC)
	message(WARNING "Building under windows with non-VS generator/compiler is currently not tested, expect errors! Please report any findings under https://github.com/3dct/open_iA-superbuild/issues!")
endif()

find_package(QT NAMES Qt6 Qt5 COMPONENTS Widgets OpenGLWidgets REQUIRED)
if (QT_DIR AND NOT Qt${QT_VERSION_MAJOR}_DIR)
	set(Qt${QT_VERSION_MAJOR}_DIR ${QT_DIR} CACHE PATH "" FORCE)
	if (${QT_VERSION_MAJOR} GREATER_EQUAL 6)
		set(Qt${QT_VERSION_MAJOR}_DIR ${QT_DIR})
		set(Qt${QT_VERSION_MAJOR}CoreTools_DIR ${QT_DIR}CoreTools CACHE PATH "" FORCE)
		set(Qt${QT_VERSION_MAJOR}GuiTools_DIR ${QT_DIR}GuiTools CACHE PATH "" FORCE)
		set(Qt${QT_VERSION_MAJOR}WidgetsTools_DIR ${QT_DIR}WidgetsTools CACHE PATH "" FORCE)
	endif()
endif()
find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Concurrent Gui OpenGL Svg Widgets Xml REQUIRED)
set(QT_PATH ${Qt${QT_VERSION_MAJOR}_DIR})
message(STATUS "Using Qt version ${QT_VERSION} in ${QT_DIR}")

find_package(Git)
if(NOT GIT_FOUND)
	message(SEND_ERROR "Git not found, but it is required for some patches!")
endif()
set(GIT_APPLY_OPTS --ignore-whitespace)
#--whitespace=nowarn --inaccurate-eof) # potential additional options in case of whitespace errors....
set(GIT_APPLY_CMD "${GIT_EXECUTABLE}" apply ${GIT_APPLY_OPTS})


function(selector varname options default desc)
	list(FIND options "${${varname}}" options_index)
	if (${options_index} EQUAL -1)
		if (DEFINED ${varname})
			message(WARNING "Invalid ${varname}, resetting to default ${default}!")
		endif()
		set(${varname} "${default}" CACHE STRING "${desc}" FORCE)
		set_property(CACHE ${varname} PROPERTY STRINGS ${options})
	endif()
endfunction()

# build types
if (NOT CMAKE_CONFIGURATION_TYPES)
	set(BUILD_TYPE_OPTIONS "Release" "Debug" "RelWithDebInfo" "MinSizeRel")
	selector(CMAKE_BUILD_TYPE "${BUILD_TYPE_OPTIONS}" "Release" "Choose the build configuration.")
endif()

## VTK

set(VTK_VERSION_OPTIONS "8.1.2" "8.2.0" "9.0.0" "9.0.1" "9.0.2" "9.0.3")
set(VTK_VERSION_DESC "Choose the VTK version to build (only used if VTK_USE_GIT_REPO is DISABLED; if VTK_USE_GIT_REPO is enabled, use VTK_GIT_TAG instead).")
selector(VTK_VERSION "${VTK_VERSION_OPTIONS}" "9.0.3" ${VTK_VERSION_DESC})

set(VTK_SMP_TYPE_OPTIONS "Sequential" "OpenMP" "TBB")
set(VTK_SMP_TYPE_DEFAULT "OpenMP")
set(VTK_SMP_TYPE_DESC "Choose the SMP implementation to use in the VTK build to speed up filters with parallel implementations - available are sequential (no parallelization), OpenMP, and TBB (Intel Thread Building Blocks - will require installation of Intel OneAPI SDK)  (default: ${VTK_SMP_TYPE_DEFAULT}).")
selector(VTK_SMP_TYPE "${VTK_SMP_TYPE_OPTIONS}" ${VTK_SMP_TYPE_DEFAULT} ${VTK_SMP_TYPE_DESC})

if (VTK_USE_GIT_REPO)
	set(VTK_GIT_TAG_OPTIONS "")
	foreach(version ${VTK_VERSION_OPTIONS})
		list(APPEND VTK_GIT_TAG_OPTIONS "v${version}")
	endforeach()
	set(VTK_GIT_TAG_DEFAULT "master")
	list(APPEND VTK_GIT_TAG_OPTIONS ${VTK_GIT_TAG_DEFAULT})
	selector(VTK_GIT_TAG "${VTK_GIT_TAG_OPTIONS}" ${VTK_GIT_TAG_DEFAULT} "Choose the VTK git tag to use in build.")
	if (VTK_GIT_TAG STREQUAL "master")
		# determine VTK_VERSION from source...
		set(VTK_VERSION "9.99.99")
		message(WARNING "Note that when building from VTK git master branch, there is no reliable way "
			"for me to determine the actual version number this branch currently refers to; therefore, "
			"no compatibility checks are possible, these checks are therefore disabled, you "
			"have to ensure compatibilities with versions of other libraries such as Qt or ITK "
			"yourself; in case of any incompatibilities you might get build errors.")
	endif()
	set(VTK_REPOSITORY "https://github.com/Kitware/VTK.git")
	# set(VTK_REPOSITORY "https://gitlab.kitware.com/vtk/vtk.git")
endif()

if (${QT_VERSION_MAJOR} GREATER_EQUAL 6 AND NOT VTK_USE_GIT_REPO)
# VTK_VERSION VERSION_LESS "9.1.0")
	message(SEND_ERROR "No released VTK version supports Qt >= 6.x yet! Please chose the VTK git master (VTK_USE_GIT_REPO ON, VTK_GIT_TAG \"master\"), or an older Qt version!")
endif()

if (NOT VTK_USE_GIT_REPO)
	if ("${VTK_VERSION}" STREQUAL "8.2.0")
		set(VTK_ZIP_SHA512  "FD6582EA20A0B339EC847FEFA18956F9919535C6CBE1C5D443E2B3E302D6F6045C4250B0A89719EC4D4309E25F05449DACF37973615FC59BED33714EC6081BF9")
		set(VTK_GZIP_SHA512 "521BD4DABEDBC24B0E80A314A34ECD7554B04AF28A7973245E3A9CF99A09B995D1B8AC42305C8E53369F226A0A6DA3CDB29105BA2C90B46492736EF717760286")
	elseif ("${VTK_VERSION}" STREQUAL "8.1.2")
		set(VTK_ZIP_SHA512  "A6782D29C3E3998A9DAFCE317DD885718E55EC20E368E00C585DE910BA98B1625C6F5F589CAE55CDBC30CE98FEF22C094DA58D4E827485F6E2DBCEE0B96B76DE")
		set(VTK_GZIP_SHA512 "C9FC498804FF9BB433BF79F4B14D9BB6134B026AEA46ED12AC40434BFDD8063C42E75D3E0CFA88D171FA271001C9444D66A471EDC9B22DF83A9109733ABCDFA0")
	elseif ("${VTK_VERSION}" STREQUAL "9.0.0")
		unset(VTK_ZIP_SHA512) # no .zip available for 9.x
		set(VTK_GZIP_SHA512 "6931BAD7D78689999E57378AB555166D1059D5621234B0E0C781AE110E3A02DEFD03DB2D1B5635101102F969FFC28547F38DA587DAB75494FB273C5F38A996A9")
	elseif ("${VTK_VERSION}" STREQUAL "9.0.1")
		unset(VTK_ZIP_SHA512) # no .zip available for 9.x
		set(VTK_GZIP_SHA512 "EB2508E2409AD302CC5DB65282A9C1FF4477AC8A93E0FEE898AD8F0974D72DECC7D5D444DC825A3C2F7952EBF59853616097EBE0D134649CAB0D49A9619065BF")
	elseif ("${VTK_VERSION}" STREQUAL "9.0.2")
		unset(VTK_ZIP_SHA512) # no .zip available for 9.x
		set(VTK_GZIP_SHA512 "65C6BCE3023B45F2D145023AE6BF224257A6D89CDF7BBD4ACDA3D6A25CB6B8C947F0E9A44129BD1882D6D8A8D46FD4CAEF0463BFB94AFC9F4EDCCA8566BC2393")
	elseif ("${VTK_VERSION}" STREQUAL "9.0.3")
		unset(VTK_ZIP_SHA512) # no .zip available for 9.x
		set(VTK_GZIP_SHA512 "00528011F9206444D09FC6CEA05C46930745BB70EA02BE6244AB3EB510AE82AF772157C025157B1761DC4C6A9AB538B57F814C03A708F30AA3598A421FDC6AE2")
	endif()
	string(SUBSTRING "${VTK_VERSION}" 0 3 VTK_SHORT_VERSION)
	set(VTK_URL "https://www.vtk.org/files/release/${VTK_SHORT_VERSION}/VTK-${VTK_VERSION}")
	if (WIN32 AND VTK_ZIP_SHA512)
		set(VTK_URL "${VTK_URL}.zip")
		set(VTK_ARCHIVE_SHA512 ${VTK_ZIP_SHA512})
	else()
		set(VTK_URL "${VTK_URL}.tar.gz")
		set(VTK_ARCHIVE_SHA512 ${VTK_GZIP_SHA512})
	endif()
endif()

## ITK

set(ITK_VERSION_OPTIONS "4.13.2" "5.0.1" "5.1.0" "5.1.1" "5.1.2" "5.2.0" "5.2.1")
set(ITK_VERSION_DESC "Choose the ITK version to build (only used if ITK_USE_GIT_REPO is DISABLED; if ITK_USE_GIT_REPO is enabled, use ITK_GIT_TAG instead).")
selector(ITK_VERSION "${ITK_VERSION_OPTIONS}" "5.2.1" "${ITK_VERSION_DESC}")
if (ITK_USE_GIT_REPO)
	set(ITK_GIT_TAG_OPTIONS "")
	foreach(version ${ITK_VERSION_OPTIONS})
		list(APPEND ITK_GIT_TAG_OPTIONS "v${version}")
	endforeach()
	set(ITK_GIT_TAG_DEFAULT "master")
	list(APPEND ITK_GIT_TAG_OPTIONS ${ITK_GIT_TAG_DEFAULT})
	selector(ITK_GIT_TAG "${ITK_GIT_TAG_OPTIONS}" ${ITK_GIT_TAG_DEFAULT} "Choose the ITK git tag to use in build.")
	set(ITK_REPOSITORY "https://github.com/InsightSoftwareConsortium/ITK.git")
else()
	if ("${ITK_VERSION}" STREQUAL "5.2.1")
		set(ITK_ZIP_SHA512 "E07132A0BE208B53D0A44918D291C47F9FCEDB78C41C6F1EA7A9179FA38A532099F06CE3E1A3993783D984734F9180F428372B5DFC7E17D05733A259B9FBBC1B")
		set(ITK_GZIP_SHA512 "6786E39CDF3D0C3A31ABD1E23481E30F6DC9DAC189FFE372DDE3DB688F2F57686A8BEB321778327E1FF683ED844D41F1DEE937B0BA542B2365E2195DFCA398C7")
	elseif ("${ITK_VERSION}" STREQUAL "5.2.0")
		set(ITK_ZIP_SHA512 "0BB28E3CB8CDB5509A54044F7F05624C5D5E14AD941668DA774ADAB5C6F55ED4E94FC35E2A94D75BE0B2A3EB8E9187968E0BFB27A626C0954CA487CD91BED309")
		set(ITK_GZIP_SHA512 "57706EFF872CD80D8B3EDFB59EE99A9C4B56F35009DCEF63D91E6BD61817A6BE432E27ED29A8EAB12748A4942ED12A8129E7A95AC98FE1BEEFEA2ED3A19083BF")
	elseif ("${ITK_VERSION}" STREQUAL "5.1.2")
		set(ITK_ZIP_SHA512 "9EC1B3BED6C2F0DDE5103976ABE3EA7BD1EBA943D28BEF3EB380E65A8728AEA94DA39B322ED3AD4C77F6F15E7BCCA5CEDEE6429513A86CC01778051F6FD26D12")
		set(ITK_GZIP_SHA512 "B7702B4933A0491110E22E98FAF8534F5DB49B6218830962EA067CCDB2FD5666ED41EE61A12CEE66484F72B1E2225235D0762F0863EAC4568D2C782761E5B03C")
	elseif ("${ITK_VERSION}" STREQUAL "5.1.1")
		set(ITK_ZIP_SHA512  "6763b64e1e6bbfae2b492aed72ab1f4f9d73e96b3e84fce55d8ef0beecaaf0b5401b58891246f11479bb3af38207a75dd04410ccf07f8553b232b179e989d98f")
		set(ITK_GZIP_SHA512 "7b595f52dc2fd19485300412941883abbd19a3c6ef96a90edf89252d9992ca109f2f7de8d9b547ef1fc0cb30d9d4c34ec17c10e8c0d07e90bd0468ffb391a12d")
	elseif ("${ITK_VERSION}" STREQUAL "5.1.0")
		set(ITK_ZIP_SHA512  "e59a3f392850f83372694f95625abcaf6554ce31b7fac8037d6a502e8dbabb96ea356e09f73cb0e7ae8514e6ebcc40b7e524b352e36a11f346dda3b70bb07bb1")
		set(ITK_GZIP_SHA512 "413b6ef549b59d8f507038130446ccb704794c6d0980d8ab631d1b1051cb91f399e77c26b69fab6a1e9b9bb4cc233d49996c3ee735cd4de065993d4379b4ed4c")
	elseif ("${ITK_VERSION}" STREQUAL "5.0.1")
		set(ITK_ZIP_SHA512  "f1a871742247e9596b39126ee70b9ab5e6e3dc6fabfc54fc2f84d1f174b51f4f9086eae5769f6bc1a28eaf81183ebe77d66ca0199d65e65ef689ab9344bd81f2")
		set(ITK_GZIP_SHA512 "f36613ff72c513ded3d32504f71308a94fe75555cf9fd22b77485d1375601f6e1f1539cc5ac82a9e1e229bcf514a88ccb55122a7dfc74a6ae1b6604aa70bd814")
	elseif ("${ITK_VERSION}" STREQUAL "4.13.2")
		set(ITK_ZIP_SHA512  "AB4934953E662FE7F039910FE3DA79C83784026970A0779A8148847401308077B672B5107861FBFBCC8F852249F76BB9A8C1615D28813EE375AC1B3371BC27DB")
		set(ITK_GZIP_SHA512 "053E7109D3BBB0FF9191AF4A3650D58B35BC486D7629E3351D09A67A52379681E9FEC960C8235B56915E2E039503B31439E789A1A2EE306BA7EF67413DAAEA78")
	endif()
	# set(ITK_REPOSITORY "https://github.com/Kitware/ITK.git")
	# set(ITK_REPOSITORY "http://itk.org/ITK.git")
	set(ITK_URL "https://github.com/InsightSoftwareConsortium/ITK/releases/download/v${ITK_VERSION}/InsightToolkit-${ITK_VERSION}")
	if (WIN32)
		set(ITK_URL "${ITK_URL}.zip")
		set(ITK_ARCHIVE_SHA512 ${ITK_ZIP_SHA512})
	else()
		set(ITK_URL "${ITK_URL}.tar.gz")
		set(ITK_ARCHIVE_SHA512 ${ITK_GZIP_SHA512})
	endif()
endif()


# TODO:
#   - Also fetch Qt: Build from source? Download & install probably not an option(only GUI installer available)
#   - check behavior when there are changes in git (and tag not the one referenced in ExternalProject_Add) / offline

## OpenVR SDK

if (ENABLE_VR)
	set(OPENVR_VERSION_OPTIONS "1.9.15" "1.11.11" "1.12.5" "1.14.15" "1.16.8")
	selector(OPENVR_VERSION "${OPENVR_VERSION_OPTIONS}" "1.16.8" "Choose the OpenVR version to build.")
	if ("${OPENVR_VERSION}" STREQUAL "1.9.15")
		set(OPENVR_ARCHIVE_SHA512 "CB047D92C383F5CF761FD5EE262DDA8755BF592AC827A8C7314294DDBA489E4709EB3AA756207382E1C16BB02DAED77EAA6CEBE48411BCE5A4C2957855FA75D9")
	elseif ("${OPENVR_VERSION}" STREQUAL "1.11.11")
		set(OPENVR_ARCHIVE_SHA512 "5DF70083F3203F58BB87D7D43ADDBB2B9F0C193EC04DF9C659CF7B51DBE6D0F151E990294ECE7962BE1266212770D67EF023D97F3194898977D6C5DE3CF1516B")
	elseif ("${OPENVR_VERSION}" STREQUAL "1.12.5")
		set(OPENVR_ARCHIVE_SHA512 "F54B275140D8E91723ECF0A0402B6E9E09B34DE0F2A59C72877CAC1F3AB0CD532D93877782A842D97B4D4AF603FB7DBB111CCC358450E23CA8C644814C82A7AD")
	elseif ("${OPENVR_VERSION}" STREQUAL "1.14.15")
		set(OPENVR_ARCHIVE_SHA512 "F076F0453C99601A4AEC9B5C8DAC25F16C294D0E902879D44D85EF28F693E746758694428DA7F172723CEB1422070F0114750FEFA98D50713AF103A3FA120603")
	elseif ("${OPENVR_VERSION}" STREQUAL "1.16.8")
		set(OPENVR_ARCHIVE_SHA512 "A64287039447648ABB30F25936A73155B77F8E8C7BD618BD64CB327D9108F69A3BCB6BC8BB8672A28D2EB69A7A02AB4D44CB3513460872E5A7AA4AD22EC9A0B3")
	endif()
	set(OPENVR_URL "https://github.com/ValveSoftware/openvr/archive/v${OPENVR_VERSION}.zip")
endif()

## Astra Toolbox

if (ENABLE_ASTRA)
	set(ASTRA_REPOSITORY "https://github.com/astra-toolbox/astra-toolbox.git")
	set(ASTRA_TAG        "v2.0.0")
endif()

if (BUILD_BOOST)
	set(BOOST_VERSION_OPTIONS "1.72.0" "1.73.0" "1.75.0" "1.76.0" "1.77.0")
	selector(BOOST_VERSION "${BOOST_VERSION_OPTIONS}" "1.77.0" "Choose the boost library version to build.")
	if ("${BOOST_VERSION}" STREQUAL "1.72.0")
		set(BOOST_SHORTVER "1_72")
		set(BOOST_ARCHIVE_SHA512 "BEB4D099E9427394D6AD0CE0D526EE2C6E96CC4CAF5DE4747D009B01CD3E517A988E79C0D4522E49217C6A7EFD551075FB18BFAC51F344CEA9978CDCFC57CC56")
	elseif ("${BOOST_VERSION}" STREQUAL "1.73.0")
		set(BOOST_SHORTVER "1_73")
		set(BOOST_ARCHIVE_SHA512 "6D80E3E0A9D1B3114B3660A099306A88CE911300EB29F160D6283AAE547AAC9AF84803DB723D9C603DA16061DF409538FFF43A66AC4A58418F2421EF63AA0CE1")
	elseif ("${BOOST_VERSION}" STREQUAL "1.75.0")
		set(BOOST_SHORTVER "1_75")
		set(BOOST_ARCHIVE_SHA512 "5DB629D67B7AF4337A39B77B1C3DC41193238A88BE6319BC3CC069FF44850C440F477814B3D63022801E5C34B8F5A79A278FB01FEFD669750A1C52C4EF3A16B3")
	elseif ("${BOOST_VERSION}" STREQUAL "1.76.0")
		set(BOOST_SHORTVER "1_76")
		set(BOOST_ARCHIVE_SHA512 "FFDD828F4837F0C7D4E3E9932C6C2E75EB3142ECF0DA924879DCDB89CE606D920B1AF1A5D2C93B3885AD95D7FC98B777305D8DAE78FD8F511251BC591B052CF8")
	elseif ("${BOOST_VERSION}" STREQUAL "1.77.0")
		set(BOOST_SHORTVER "1_77")
		set(BOOST_ARCHIVE_SHA512 "34CAA0F45BBB20CD162E7B9B6BDC39382B8E9FA036F2B872CC45C3EBFE03CB056097507A5DCA4A0D7B7DA4CB0FAB30484FFAE4E8108DE48C7F2ECC846938AFBB")
	endif()
	if (MSVC)
		if (MSVC_VERSION LESS 1900)
			message(SEND_ERROR "Visual Studio < 2015 is not supported, please use a newer version!")
		elseif (MSVC_VERSION LESS 1910)
			set(MSVC_TOOLSET "v140")
			set(BOOST_MSVC_TOOLSET "msvc-14.0")
			set(BOOST_LIB_SUFFIX "vc140")
		elseif (MSVC_VERSION LESS 1920)
			set(MSVC_TOOLSET "v141")
			set(BOOST_MSVC_TOOLSET "msvc-14.1")
			set(BOOST_LIB_SUFFIX "vc141")
		elseif (MSVC_VERSION LESS 1930)
			set(MSVC_TOOLSET "v142")
			set(BOOST_MSVC_TOOLSET "msvc-14.2")
			set(BOOST_LIB_SUFFIX "vc142")
		elseif (MSVC_VERSION LESS 1940)
			set(MSVC_TOOLSET "v143")
			set(BOOST_MSVC_TOOLSET "msvc-14.3")
			set(BOOST_LIB_SUFFIX "vc143")
		else()
			message(SEND_ERROR "Visual Studio > 2022 (more precisely, MSVC_VERSION >= 1940) is not supported (yet), please use an older version!")
		endif()
	endif()
endif()

if (ENABLE_EIGEN)
	set(EIGEN_VERSION_OPTIONS "3.2.10" "3.3.7" "3.3.9")
	selector(EIGEN_VERSION "${EIGEN_VERSION_OPTIONS}" "3.3.9" "Choose the eigen library version to use.")
	if ("${EIGEN_VERSION}" STREQUAL "3.2.10")
		set(EIGEN_ARCHIVE_SHA512 "290AD5B4029A09E64B9337C0A4E266D7F4726A91438E91F8011FC99A01E62E70E3B24CFC1A782D12F121A9620A13E0EC91CA8696DF202BC8D73559322E416477")
		# for .tar.bz2: "E130F8CA426070663AADB462C600DF88BB4EADF4F56785188B4D07B5B3F0E028603B0C9FB5C89E47B8692FD134D56913FEE7DFE632E062DD9E903B38A3211816"
	elseif ("${EIGEN_VERSION}" STREQUAL "3.3.7")
		set(EIGEN_ARCHIVE_SHA512 "6C19DF0271503DDCE61724729031A3B885D86F35D1F24363A76F9E300234D3257FF56020ACE624B2C011E17CBA6D69A3014508C5E6DC3436BBE331A2129E2175")
	elseif ("${EIGEN_VERSION}" STREQUAL "3.3.9")
		set(EIGEN_ARCHIVE_SHA512 "3B06DA583037649AD42190BD018DDF40B85179AD0877EBD7C1CA7B8498A1453EED25919DE27D3BB737754086E1513408E7DE4A2110D51912F2E5503E1AB7D838")
	endif()
	set(EIGEN_URL "https://gitlab.com/libeigen/eigen/-/archive/${EIGEN_VERSION}/eigen-${EIGEN_VERSION}.zip")
endif()

# TODO: configurable HDF5 version (as for other libraries)
set(HDF5_VERSION "1.12.0")
string(SUBSTRING "${HDF5_VERSION}" 0 4 HDF5_SHORT_VERSION)
set(HDF5_URL "https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-${HDF5_SHORT_VERSION}/hdf5-${HDF5_VERSION}/src/CMake-hdf5-${HDF5_VERSION}")
set(HDF5_ZIP_SHA512   "82C37F19E9907DF3722C44F8423DD9408EB0646A600AD59A727E91D2B1AA9DADFD56EDC5E2A8AA272F9AC84B188B9EE7FD2B0145C5380C511E3B132A638366BA")
set(HDF5_TARGZ_SHA512 "67B415D2125010D587003C76FFDBD8B9B27EA79BF3309C1A916F57F9FE008B83B08067325961307D16ADC03B7D8B90FBFCD6CD30EB2B49619112614AFC8AE6EF")
if (WIN32)
	set(HDF5_URL "${HDF5_URL}.zip")
	set(HDF5_ARCHIVE_SHA512 ${HDF5_ZIP_SHA512})
else()
	set(HDF5_URL "${HDF5_URL}.tar.gz")
	set(HDF5_ARCHIVE_SHA512 ${HDF5_TARGZ_SHA512})
endif()

if (MSVC)
	set(ADDITIONAL_LIB_PATHS "") # for additional lib paths to be added to MSVC debugger environment path
endif()

set(ITK_DEPENDS "")

if (BUILD_VTK)
	set(VTK_CMAKE_ARGS
		"-DBUILD_SHARED_LIBS:BOOL=On"
		"-DVTK_LEGACY_REMOVE:BOOL=On"
		"-DQt${QT_VERSION_MAJOR}_DIR:PATH=${QT_PATH}"
		"-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
		"-DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}"
		"-DVTK_SMP_IMPLEMENTATION_TYPE=${VTK_SMP_TYPE}"
	)
	if (QT_VERSION_MAJOR GREATER_EQUAL 6)
		list(APPEND VTK_CMAKE_ARGS
			"-DQt6WidgetsTools_DIR=${Qt6_DIR}WidgetsTools"
			"-DQt6CoreTools_DIR=${Qt6_DIR}CoreTools"
			"-DQt6GuiTools_DIR=${Qt6_DIR}GuiTools")
	endif()
	#if (DISABLE_LIB_WARNINGS)
	#	list(APPEND VTK_CMAKE_ARGS ${DISABLE_WARNINGS_OPT})
	#endif()
	if (NOT CMAKE_CONFIGURATION_TYPES)
		list(APPEND VTK_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
	endif()
	if (VTK_VERSION VERSION_LESS "9.0.0")
		list(APPEND VTK_CMAKE_ARGS
			"-DBUILD_TESTING:BOOL=Off"
			"-DModule_vtkGUISupportQt:BOOL=On"
			"-DModule_vtkGUISupportQtOpenGL:BOOL=On"
			"-DModule_vtkRenderingQt:BOOL=On"
			"-DModule_vtkViewsQt:BOOL=On"
			"-DVTK_RENDER_BACKEND:STRING=OpenGL2")
	else()
		list(APPEND VTK_CMAKE_ARGS
			"-DVTK_BUILD_TESTING=OFF"
			"-DVTK_MODULE_ENABLE_VTK_GUISupportQt=YES"
			"-DVTK_MODULE_ENABLE_VTK_GUISupportQtSQL=NO"
			"-DVTK_MODULE_ENABLE_VTK_RenderingQt=YES"
			"-DVTK_MODULE_ENABLE_VTK_RenderingContextOpenGL2=YES"
			"-DVTK_MODULE_ENABLE_VTK_ViewsQt=YES")
	endif()
	set(VTK_DEPENDS "")
	set(VTK_PATCH_COMMAND ${GIT_APPLY_CMD})
	if (ENABLE_VR)
		set(OPENVR_ROOT_DIR "${CMAKE_BINARY_DIR}/openvr-${OPENVR_VERSION}")
		set(OPENVR_INCLUDE_DIR "${OPENVR_ROOT_DIR}/headers")
		if (WIN32)
			set(OPENVR_LIB_FILES "${OPENVR_ROOT_DIR}/lib/win64/openvr_api.lib")
		elseif(UNIX)
			set(OPENVR_LIB_FILES "${OPENVR_ROOT_DIR}/lib/linux64/libopenvr_api.so")
		elseif(APPLE)
			set(OPENVR_LIB_FILES "${OPENVR_ROOT_DIR}/lib/osx64/OpenVR.framework")
		else()
			message(WARNING "Unknown operating system, probably no VR support!")
		endif()
		set(OPENVR_LIB_AND_INCLUDE
			"-DOpenVR_INCLUDE_DIR:PATH=${OPENVR_INCLUDE_DIR}"
			"-DOpenVR_LIBRARY:FILEPATH=${OPENVR_LIB_FILES}")
		ExternalProject_Add (OpenVR
			URL               ${OPENVR_URL}
			URL_HASH          "SHA512=${OPENVR_ARCHIVE_SHA512}"
			DOWNLOAD_DIR      ${ARCHIVE_DIR}
			SOURCE_DIR        "${OPENVR_ROOT_DIR}"
			BUILD_IN_SOURCE   1
			CONFIGURE_COMMAND ""
			BUILD_COMMAND     ""
			INSTALL_COMMAND   "")
		set(VTK_DEPENDS OpenVR)
		if (VTK_VERSION VERSION_LESS "9.0.0")
			list(APPEND VTK_CMAKE_ARGS
				"-DModule_vtkRenderingOpenVR:BOOL=ON"
				"-DOPENVR_ROOT_DIR:PATH=${OPENVR_ROOT_DIR}")
		else()
			list(APPEND VTK_CMAKE_ARGS
				"-DVTK_MODULE_ENABLE_VTK_RenderingOpenVR=YES"
				${OPENVR_LIB_AND_INCLUDE})
			# In VTK 9, some code was changed in Rendering/OpenVR module that produces an error on clang 10, let's patch it:
			if (NOT VTK_USE_GIT_REPO OR NOT VTK_GIT_TAG STREQUAL "master")		# but it's fixed in git master
				list(APPEND VTK_PATCH_COMMAND "${CMAKE_SOURCE_DIR}/patches/vtk-9.0.0-openvr.patch")
			endif()
		endif()
		if (MSVC)
			string(REPLACE "/" "\\" OPENVR_WIN_DIR "${OPENVR_ROOT_DIR}")
			set(ADDITIONAL_LIB_PATHS "${ADDITIONAL_LIB_PATHS};${OPENVR_WIN_DIR}\\bin\\win64")
		endif()
	endif()
	# let's apply the fix for parallel coordinate charts potentially causing a crash
	# on mouse move (see https://gitlab.kitware.com/vtk/vtk/-/issues/17441):
	if (VTK_VERSION VERSION_LESS "9.0.0")
		set(VTK_PARALLEL_COORDINATE_PATCH_LINE 510)
	else()
		set(VTK_PARALLEL_COORDINATE_PATCH_LINE 492)
	endif()
	CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/patches/vtk-parallelCoordinates.patch.template" "${CMAKE_BINARY_DIR}/vtk-parallelCoordinates.patch")
	list(APPEND VTK_PATCH_COMMAND "${CMAKE_BINARY_DIR}/vtk-parallelCoordinates.patch")

	if (QT_VERSION_MAJOR EQUAL 5 AND Qt5_VERSION VERSION_GREATER "5.14.99" AND VTK_VERSION VERSION_LESS "9.0.1") # when using >= Qt 5.15, VTK up to (and including) 9.0.0 requires a minor patch:
		list(APPEND VTK_PATCH_COMMAND "${CMAKE_SOURCE_DIR}/patches/vtk-${VTK_VERSION}-qt-5.15.patch")
	endif()
	if (VTK_PATCH_COMMAND STREQUAL "${GIT_APPLY_CMD}")
		set(VTK_PATCH_COMMAND "")	# if no patch to apply, set empty
	endif()
	if (VTK_USE_GIT_REPO)
		set(VTK_DIR "${CMAKE_BINARY_DIR}/vtk/bin-${VTK_GIT_TAG}" CACHE PATH
			"VTK build folder." FORCE)
		ExternalProject_Add (VTK
			GIT_REPOSITORY    "${VTK_REPOSITORY}"
			GIT_TAG           "${VTK_GIT_TAG}"
			UPDATE_DISCONNECTED 1
			PATCH_COMMAND     ${VTK_PATCH_COMMAND}
			#UPDATE_COMMAND    ""  # to force cmake to stick to the tag specified above, otherwise it will always go to latest 'release' branch commit
			SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src.git"
			BINARY_DIR        "${VTK_DIR}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
			INSTALL_COMMAND   ""
			DEPENDS           ${VTK_DEPENDS})
	else()
		set(VTK_DIR "${CMAKE_BINARY_DIR}/vtk/bin-${VTK_VERSION}" CACHE PATH
			"VTK build folder." FORCE)
		ExternalProject_Add (VTK
			URL               ${VTK_URL}
			URL_HASH          "SHA512=${VTK_ARCHIVE_SHA512}"
			DOWNLOAD_DIR      ${ARCHIVE_DIR}
			PATCH_COMMAND     ${VTK_PATCH_COMMAND}
			SOURCE_DIR        "${CMAKE_BINARY_DIR}/vtk/src-${VTK_VERSION}"
			BINARY_DIR        "${VTK_DIR}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "${VTK_CMAKE_ARGS}"
			INSTALL_COMMAND   ""
			DEPENDS           ${VTK_DEPENDS})
	endif()
	set(ITK_DEPENDS VTK)
else()
	set(VTK_DIR "${VTK_DIR}" CACHE PATH
		"VTK build folder.")
endif()

if (ENABLE_OPENCL)
	option(BUILD_OPENCL "Build OpenCL ICD loader. If disabled, you need to set OPENCL_DIR to an existing OpenCL ICD loader build" ON)
	if (BUILD_OPENCL)
		set(OPENCL_DIR "${CMAKE_BINARY_DIR}/OpenCL" CACHE PATH
			"OpenCL ICD loader directory" FORCE)
		set(OPENCL_C_HEADERS_REPO "https://github.com/KhronosGroup/OpenCL-Headers.git")
		set(OPENCL_CPP_HEADERS_REPO "https://github.com/KhronosGroup/OpenCL-CLHPP.git")
		set(OPENCL_ICD_REPO "https://github.com/KhronosGroup/OpenCL-ICD-Loader.git")
		set(OPENCL_TAG "v2020.03.13")
		# newer versions of OpenCL CPP don't include cl.hpp, which we still need
		set(OPENCL_CPP_TAG "v2.0.11")
		ExternalProject_Add (OpenCL_C
			GIT_REPOSITORY    ${OPENCL_C_HEADERS_REPO}
			GIT_TAG           "${OPENCL_TAG}"
			UPDATE_DISCONNECTED 1
			SOURCE_DIR        "${OPENCL_DIR}/cheaders"
			BUILD_IN_SOURCE   1
			CONFIGURE_COMMAND ""
			BUILD_COMMAND     ""
			INSTALL_COMMAND   "")
		ExternalProject_Add (OpenCL_CPP
			GIT_REPOSITORY    ${OPENCL_CPP_HEADERS_REPO}
			GIT_TAG           "${OPENCL_CPP_TAG}"
			GIT_SUBMODULES    ""
			UPDATE_DISCONNECTED 1
			SOURCE_DIR        "${OPENCL_DIR}/cppheaders"
			BUILD_IN_SOURCE   1
			CONFIGURE_COMMAND ""
			BUILD_COMMAND     ""
			INSTALL_COMMAND   "")
		# Find way to skip re-build of OpenCL every time!

		set(CMD_COPY_OPENCL_C_HEADERS ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/cheaders/CL" "${OPENCL_DIR}/icdloader/inc/CL")
		set(CMD_COPY_OPENCL_CPP_HEADERS ${CMAKE_COMMAND} -E copy "${OPENCL_DIR}/cppheaders/include/CL/cl.hpp" "${OPENCL_DIR}/cppheaders/include/CL/cl2.hpp" "${OPENCL_DIR}/icdloader/inc/CL/")
		set(OPENCL_ICD_INSTALL_HEADERS_COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/icdloader/inc/CL" "${OPENCL_DIR}/include/CL")
		# Find way to use different configurations properly...?
		if (CMAKE_CONFIGURATION_TYPES)
			# multi-configuration generator type
			set(OPENCL_ICD_INSTALL_LIBS_COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/icd-bin/$(Configuration)" "${OPENCL_DIR}/lib")
		else()
			set(OPENCL_ICD_INSTALL_LIBS_COMMAND ${CMAKE_COMMAND} -E copy_directory "${OPENCL_DIR}/icd-bin" "${OPENCL_DIR}/lib")
		endif()
		set(OPENCL_ICD_CMAKE_ARGS
			"-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
			"-DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}"
		)
		#if (DISABLE_LIB_WARNINGS)
		#	list(APPEND OPENCL_ICD_CMAKE_ARGS ${DISABLE_WARNINGS_OPT})
		#endif()
		if (NOT CMAKE_CONFIGURATION_TYPES)
			LIST(APPEND OPENCL_ICD_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
		endif()
		if (WIN32)
			LIST(APPEND OPENCL_ICD_CMAKE_ARGS "-DOPENCL_ICD_LOADER_REQUIRE_WDK:BOOL=OFF")
		endif()
		ExternalProject_Add (OpenCL_ICD
			GIT_REPOSITORY    ${OPENCL_ICD_REPO}
			GIT_TAG           "${OPENCL_TAG}"
			GIT_SUBMODULES    ""
			UPDATE_DISCONNECTED 1
			PATCH_COMMAND     ${CMD_COPY_OPENCL_C_HEADERS}
			      COMMAND     ${CMD_COPY_OPENCL_CPP_HEADERS}
			SOURCE_DIR        "${OPENCL_DIR}/icdloader"
			BINARY_DIR        "${OPENCL_DIR}/icd-bin"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        ${OPENCL_ICD_CMAKE_ARGS}
			INSTALL_COMMAND   ${OPENCL_ICD_INSTALL_HEADERS_COMMAND}
			        COMMAND   ${OPENCL_ICD_INSTALL_LIBS_COMMAND}
			DEPENDS           OpenCL_CPP OpenCL_C)

		list(APPEND ITK_DEPENDS OpenCL_ICD)
		set(OPENCL_INCLUDE_DIRS "${OPENCL_DIR}/include")
		if (WIN32)
			set(OPENCL_LIB_SUFFIX ".lib") # only Windows has the peculiarity of requiring to link against the stub .lib file
		else()
			set(OPENCL_LIB_SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX})
		endif()
		set(OPENCL_LIB_NAME "${CMAKE_SHARED_LIBRARY_PREFIX}OpenCL${OPENCL_LIB_SUFFIX}")
		set(OPENCL_LIBRARIES "${OPENCL_DIR}/lib/${OPENCL_LIB_NAME}")
	else()
		find_package(OpenCL REQUIRED)
		if (OPENCL_FOUND)
			message(STATUS "OpenCL: ${OpenCL_INCLUDE_DIRS}, ${OpenCL_LIBRARIES}")
			set(OPENCL_INCLUDE_DIRS ${OpenCL_INCLUDE_DIRS})
			set(OPENCL_LIBRARIES ${OpenCL_LIBRARIES})
		endif()
	endif()
endif()

set(openiA_DEPENDS "")
if (BUILD_ITK)
	set(ENABLE_ITK_GPU ${ENABLE_OPENCL})
	if (ENABLE_ITK_GPU AND ITK_VERSION STREQUAL "5.1.0")
		set(ENABLE_ITK_GPU OFF)
		message(WARNING "Disabling ITK's GPU accelerated filters, as they don't work in ITK 5.1.0. If you want to use these, you need to choose version 5.0.1 or version 4.13.2 of ITK! Technical details: In ITK 5.1.0, enabling ITK_USE_GPU flag results in compilation errors when enabled together with shared libraries, see https://github.com/InsightSoftwareConsortium/ITK/issues/1381.")
	endif()
	set(ITK_CMAKE_ARGS
		"-DBUILD_SHARED_LIBS:BOOL=On"
		"-DBUILD_TESTING:BOOL=Off"
		"-DITK_USE_GPU:BOOL=${ENABLE_ITK_GPU}"
		"-DITK_USE_64BITS_IDS:BOOL=On"
		"-DModule_ITKReview:BOOL=On"
		"-DModule_ITKVtkGlue:BOOL=On"
		"-DModule_HigherOrderAccurateGradient:BOOL=On"
		"-DModule_SplitComponents:BOOL=On"
		"-DVTK_DIR:PATH=${VTK_DIR}"
		"-DQt${QT_VERSION_MAJOR}_DIR:PATH=${QT_PATH}"
		"-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
		"-DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}"
	)
#	if (DISABLE_LIB_WARNINGS)
#		list(APPEND ITK_CMAKE_ARGS ${DISABLE_WARNINGS_OPT})
#	endif()
	if (QT_VERSION_MAJOR GREATER_EQUAL 6)
		list(APPEND ITK_CMAKE_ARGS
			"-DQt6WidgetsTools_DIR=${Qt6_DIR}WidgetsTools"
			"-DQt6CoreTools_DIR=${Qt6_DIR}CoreTools"
			"-DQt6GuiTools_DIR=${Qt6_DIR}GuiTools")
	endif()
	if (NOT CMAKE_CONFIGURATION_TYPES)
		LIST(APPEND ITK_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
	endif()
	if (ENABLE_ITK_GPU)
		list(APPEND ITK_CMAKE_ARGS
			"-DOPENCL_ROOT_DIR=${OPENCL_DIR}"
			"-DOPENCL_INCLUDE_DIRS=${OPENCL_INCLUDE_DIRS}"
			"-DOPENCL_LIBRARIES=${OPENCL_LIBRARIES}")
	endif()
	if (ENABLE_VR AND VTK_VERSION VERSION_GREATER_EQUAL "9.0.0")
		list(APPEND ITK_CMAKE_ARGS
			${OPENVR_LIB_AND_INCLUDE})
	endif()
	set(ITK_PATCH_COMMAND ${GIT_APPLY_CMD})
	if (ITK_VERSION VERSION_LESS "5.1.0")
		set(ITK_VTKGLUE_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-vtkGlue.patch")
		LIST(APPEND ITK_PATCH_COMMAND ${ITK_VTKGLUE_PATCH})
	endif()
	if (ITK_VERSION VERSION_LESS "5.2.0" AND MSVC_VERSION GREATER_EQUAL 1928)
		set(ITK_LRINTF_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-lrintf.patch")
		LIST(APPEND ITK_PATCH_COMMAND ${ITK_LRINTF_PATCH})
	endif()
	if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL "11.0.0" AND ITK_VERSION VERSION_LESS "5.2.1")
		if (NOT ITK_VERSION VERSION_EQUAL "5.2.0")
			message(WARNING "You are using g++ version >= 11 with ITK < 5.2.0. Using g++ version >= 11 requires a patch for ITK 5.2.0, and most likely also for versions below, but the patch included in this superbuild is only tested with ITK version 5.2.0. For other ITK versions the patch might fail! If it fails, your options are to choose a newer ITK version (>= 5.2.0), use a different/older compiler, or fix any compilation errors that you get yourself!")
		endif()
		set(ITK_GCC11_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-g++11.patch")
		LIST(APPEND ITK_PATCH_COMMAND ${ITK_GCC11_PATCH})
	endif()
	if (ITK_VERSION VERSION_GREATER_EQUAL "5.2.0")
		set(ITK_GREATER2GB_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-5.2-metaIO-decompressgreater2GBfix.patch")
		LIST(APPEND ITK_PATCH_COMMAND ${ITK_GREATER2GB_PATCH})
	endif()
	if (ITK_PATCH_COMMAND STREQUAL "${GIT_APPLY_CMD}")
		set(ITK_PATCH_COMMAND "")	# if no patch to apply, set empty
	endif()
	if (ITK_USE_GIT_REPO)
		set(ITK_DIR "${CMAKE_BINARY_DIR}/itk/bin-${ITK_GIT_TAG}" CACHE PATH "ITK build folder." FORCE)
		ExternalProject_Add (ITK
			GIT_REPOSITORY    "${ITK_REPOSITORY}"
			GIT_TAG           "${ITK_GIT_TAG}"
			UPDATE_DISCONNECTED 1
			PATCH_COMMAND     ${ITK_PATCH_COMMAND}
			SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src.git"
			BINARY_DIR        "${ITK_DIR}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
			INSTALL_COMMAND   ""
			DEPENDS           ${ITK_DEPENDS})
	else()
		set(ITK_DIR "${CMAKE_BINARY_DIR}/itk/bin-${ITK_VERSION}" CACHE PATH "ITK build folder." FORCE)
		ExternalProject_Add (ITK
			URL               "${ITK_URL}"
			URL_HASH          "SHA512=${ITK_ARCHIVE_SHA512}"
			DOWNLOAD_DIR      ${ARCHIVE_DIR}
			PATCH_COMMAND     ${ITK_PATCH_COMMAND}
			SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src-${ITK_VERSION}"
			BINARY_DIR        "${ITK_DIR}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
			INSTALL_COMMAND   ""
			DEPENDS           ${ITK_DEPENDS})
	endif()
	set(openiA_DEPENDS ITK)
else()
	set(ITK_DIR "${ITK_DIR}" CACHE PATH
		"ITK build folder.")
endif()

set(CUDA_FIND_ENABLED OFF)
if ((ENABLE_ASTRA AND BUILD_ASTRA) OR ENABLE_AI)
	set(CUDA_FIND_ENABLED ON)
endif()
if (CUDA_FIND_ENABLED)
	FIND_PACKAGE(CUDA)
	if (CUDA_FOUND)
		message(STATUS "CUDA: ${CUDA_VERSION} in ${CUDA_TOOLKIT_ROOT_DIR}.")
	else()
		message(SEND_ERROR "CUDA, required for ASTRA / AI, not found! Specify CUDA_TOOLKIT_ROOT_DIR, or uncheck ENABLE_ASTRA!")
	endif()
endif()

if (BUILD_BOOST)
	# fetch and build boost...
	# inspired by https://github.com/apache/geode-native/blob/develop/dependencies/boost/CMakeLists.txt
	string(REPLACE "." "_" BOOST_VERSION_UNDERSCORE ${BOOST_VERSION})
	set(BOOST_URL "https://boostorg.jfrog.io/artifactory/main/release/${BOOST_VERSION}/source/boost_${BOOST_VERSION_UNDERSCORE}.zip")
	set(BOOST_BOOTSTRAP_OPTS "")
	if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
		set(BOOST_BOOTSTRAP_OPTS "--with-toolset=clang")
	endif()
	set(B2_FLAGS "-j8 address-model=64 architecture=x86 link=static threading=multi runtime-link=shared --build-type minimal stage")
	if (WIN32)
		set( BOOST_BOOTSTRAP_COMMAND ${CMAKE_BINARY_DIR}\\boost\\bootstrap.bat )
		set( B2_COMMAND ${CMAKE_BINARY_DIR}\\boost\\b2 )
		set( B2_FLAGS ${B2_FLAGS} toolset=${BOOST_MSVC_TOOLSET} )
	else()
		set( BOOST_BOOTSTRAP_COMMAND ${CMAKE_BINARY_DIR}/boost/bootstrap.sh )
		set( B2_COMMAND ${CMAKE_BINARY_DIR}/boost/b2 )
		set( B2_FLAGS ${B2_FLAGS} )
	endif()
	set(BOOST_DIR "${CMAKE_BINARY_DIR}/boost")
	ExternalProject_Add (boost
		URL               ${BOOST_URL}
		URL_HASH          SHA512=${BOOST_ARCHIVE_SHA512}
		DOWNLOAD_DIR      ${ARCHIVE_DIR}
		BUILD_IN_SOURCE   1
		SOURCE_DIR        "${BOOST_DIR}"
		CONFIGURE_COMMAND ${BOOST_BOOTSTRAP_COMMAND} ${BOOST_BOOTSTRAP_OPTS}
		BUILD_COMMAND     ${B2_COMMAND} ${B2_FLAGS}
		INSTALL_COMMAND   ""  # ${B2_COMMAND} ${B2_FLAGS} install
	)
	# set boost include dir
	set(BOOST_INCLUDE_DIR ${BOOST_DIR})
endif()

if (ENABLE_ASTRA)
	if (BUILD_ASTRA)
		set(ASTRA_DIR "${CMAKE_BINARY_DIR}/astra" CACHE PATH
			"Astra toolbox directory" FORCE)
		if (WIN32)
			option(ASTRA_PATCH "Whether to patch Astra Toolbox project files. Only required to be set first time sources are fetched, disable later." ON)
		endif()

		set(ASTRA_DEPENDS "")
		set(CMD_COPY_BOOST_HEADERS "")
		set(CMD_CREATE_LIBX64_DIR "")
		set(CMD_COPY_BOOST_LIBS "")

		if (BUILD_BOOST)
			# prepare the commands required to copy over boost stuff into astra build tree:
			set(CMD_COPY_BOOST_HEADERS ${CMAKE_COMMAND} -E copy_directory "${BOOST_DIR}/boost" "${ASTRA_DIR}/lib/include/boost")
			set(CMD_CREATE_LIBX64_DIR ${CMAKE_COMMAND} -E make_directory "${ASTRA_DIR}/lib/x64")
			set(BOOST_LIB_BASE "${BOOST_DIR}/stage/lib/libboost_")
			set(BOOST_LIB_LIST "chrono" "date_time" "system" "thread")
			if (WIN32)
				set(BOOST_LIB_LIST1 ${BOOST_LIB_LIST})
				set(BOOST_LIB_LIST2 ${BOOST_LIB_LIST})
				list(TRANSFORM BOOST_LIB_LIST1 PREPEND ${BOOST_LIB_BASE})
				list(TRANSFORM BOOST_LIB_LIST2 PREPEND ${BOOST_LIB_BASE})
				set(BOOST_LIB_SUFF1 "-${BOOST_LIB_SUFFIX}-mt-gd-x64-${BOOST_SHORTVER}.lib")
				set(BOOST_LIB_SUFF2 "-${BOOST_LIB_SUFFIX}-mt-x64-${BOOST_SHORTVER}.lib")
				list(TRANSFORM BOOST_LIB_LIST1 APPEND ${BOOST_LIB_SUFF1})
				list(TRANSFORM BOOST_LIB_LIST2 APPEND ${BOOST_LIB_SUFF2})
				set(BOOST_COPY_LIBS ${BOOST_LIB_LIST1} ${BOOST_LIB_LIST2})
			else()
				set(BOOST_COPY_LIBS ${BOOST_LIB_LIST})
				list(TRANSFORM BOOST_COPY_LIBS PREPEND ${BOOST_LIB_BASE})
				list(TRANSFORM BOOST_COPY_LIBS APPEND ${CMAKE_STATIC_LIBRARY_SUFFIX})
			endif()
			# message(STATUS "BOOST Copy libs: ${BOOST_COPY_LIBS}")
			set(CMD_COPY_BOOST_LIBS ${CMAKE_COMMAND} -E copy ${BOOST_COPY_LIBS} "${ASTRA_DIR}/lib/x64")
			set(ASTRA_DEPENDS boost)
		else()
			message(STATUS "Note: Building Astra Toolbox is enabled but building BOOST is disabled")
			message(STATUS "	Make sure to install boost via package manager, e.g. on Ubuntu via `sudo apt install libboost-dev`")
		endif()
		set(ASTRA_PATCH_COMMAND ${GIT_APPLY_CMD})
		if (WIN32)
			CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/patches/astra-vs.patch.template" "${CMAKE_BINARY_DIR}/astra-vs.patch")
			# find way to only do this once:
			if (ASTRA_PATCH)
				list(APPEND ASTRA_PATCH_COMMAND "${CMAKE_BINARY_DIR}/astra-vs.patch")
			endif()
			set(ASTRA_CONFIGURE_COMMAND "")
			set(ASTRA_BUILD_COMMAND "${CMAKE_MAKE_PROGRAM}" "${ASTRA_DIR}/astra_vc14.sln" -t:astra_vc14 -p:Configuration=Release_CUDA -p:Platform=x64)
		else()
			set(ASTRA_BUILD_DIR "${ASTRA_DIR}/build/linux")
			set(ASTRA_CONFIGURE_COMMAND1 "${CMAKE_COMMAND}" -E chdir "${ASTRA_BUILD_DIR}" "${ASTRA_BUILD_DIR}/autogen.sh")
			set(ASTRA_CONFIGURE_COMMAND2 "${CMAKE_COMMAND}" -E chdir "${ASTRA_BUILD_DIR}" "${ASTRA_BUILD_DIR}/configure" "--with-cuda=${CUDA_TOOLKIT_ROOT_DIR}")
			set(ASTRA_BUILD_COMMAND "${CMAKE_COMMAND}" -E chdir "${ASTRA_BUILD_DIR}" "make" -j 8)
		endif()

		if (UNIX)
			message(STATUS "Note: Building Astra Toolbox is enabled, and you're building under *nix.")
			message(STATUS "	Make sure to install autoconf/automake/libtool, e.g. on Ubuntu via `sudo apt install automake libtool`")
		endif()

		if (DISABLE_LIB_WARNINGS)
			list(APPEND ASTRA_PATCH_COMMAND "${CMAKE_SOURCE_DIR}/patches/astra-vs-warnings.patch")
		endif()

		if (ASTRA_PATCH_COMMAND STREQUAL "${GIT_APPLY_CMD}")
			set(ASTRA_PATCH_COMMAND "")	# if no patch to apply, set empty
		endif()

		# fetch and build astra:
		ExternalProject_Add (astra
			GIT_REPOSITORY    "${ASTRA_REPOSITORY}"
			GIT_TAG           "${ASTRA_TAG}"
			UPDATE_DISCONNECTED 1
			SOURCE_DIR        "${ASTRA_DIR}"
			BUILD_IN_SOURCE   1
			PATCH_COMMAND     ${ASTRA_PATCH_COMMAND}
			      COMMAND     ${CMD_COPY_BOOST_HEADERS}
			      COMMAND     ${CMD_CREATE_LIBX64_DIR}
			      COMMAND     ${CMD_COPY_BOOST_LIBS}
			CONFIGURE_COMMAND ${ASTRA_CONFIGURE_COMMAND1}
			      COMMAND     ${ASTRA_CONFIGURE_COMMAND2}
			BUILD_COMMAND     ${ASTRA_BUILD_COMMAND}
			INSTALL_COMMAND   ""
			DEPENDS           ${ASTRA_DEPENDS}
		)
	else()
		set(ASTRA_DIR "${ASTRA_DIR}" CACHE PATH
			"Astra toolbox directory")
	endif()
endif()


if (ENABLE_HDF5)
	set(HDF5_SRC_DIR "${CMAKE_BINARY_DIR}/hdf5-${HDF5_VERSION}-src")
	set(HDF5_PATCH_COMMAND ${GIT_APPLY_CMD} "${CMAKE_SOURCE_DIR}/patches/hdf5-options.patch")
	if (WIN32)
		if (MSVC_VERSION LESS 1900)
			message(SEND_ERROR "Visual Studio < 2015 is not supported, please use a newer version!")
		elseif (MSVC_VERSION LESS 1910)
			set(HDF5_GENERATOR "VS201564")
		elseif (MSVC_VERSION LESS 1920)
			set(HDF5_GENERATOR "VS201764")
		elseif (MSVC_VERSION LESS 1930)
			set(HDF5_GENERATOR "VS201964")
		#elseif (MSVC_VERSION LESS 1940)
		#	set(HDF5_GENERATOR "VS202264")
		else()
			message(SEND_ERROR "Visual Studio >= 2022 (more precisely, MSVC_VERSION >= 1930) is not supported (yet) in HDF5 build, please use an older MSVC version if you require HDF5!")
		endif()
		set(HDF5_BUILD_COMMAND "${CMAKE_CTEST_COMMAND}" -S HDF5config.cmake,BUILD_GENERATOR=${HDF5_GENERATOR} -C Release -V -O hdf5.log)
		set(HDF5_PLATFORM_SUFFIX "win64.zip")
	else()
		set(HDF5_BUILD_COMMAND "${HDF5_SRC_DIR}/build-unix.sh")
		set(HDF5_PLATFORM_SUFFIX "Linux.tar.gz")
	# Apple?
	endif()
	set(HDF5_FULL_NAME "HDF5-${HDF5_VERSION}-${HDF5_PLATFORM_SUFFIX}")
	set(HDF5_INSTALL_COMMAND "${CMAKE_COMMAND}" -E chdir "${CMAKE_BINARY_DIR}" "${CMAKE_COMMAND}" -E tar x "${HDF5_SRC_DIR}/${HDF5_FULL_NAME}")
	set(HDF5_INSTALL_DIR "${CMAKE_BINARY_DIR}/${HDF5_FULL_NAME}")
	ExternalProject_Add (hdf5
		URL               ${HDF5_URL}
		URL_HASH          "SHA512=${HDF5_ARCHIVE_SHA512}"
		DOWNLOAD_DIR      ${ARCHIVE_DIR}
		PATCH_COMMAND     ${HDF5_PATCH_COMMAND}
		SOURCE_DIR        "${HDF5_SRC_DIR}"
		BUILD_IN_SOURCE   1
		CONFIGURE_COMMAND ""
		BUILD_COMMAND     ${HDF5_BUILD_COMMAND}
		INSTALL_COMMAND   "${HDF5_INSTALL_COMMAND}")
endif()

if (ENABLE_AI)
	# CUDA or DirectML versions of onnx runtime are available
	# (there is a CPU version as well, but open_iA currently does not support it)
	set(ONNX_VERSION_OPTIONS "1.4.0" "1.6.0" "1.8.1")
	selector(ONNX_VERSION "${ONNX_VERSION_OPTIONS}" "1.6.0" "Choose the onnx runtime version to use.")
	set(ONNX_SRC_DIR "${CMAKE_BINARY_DIR}/onnx-${ONNX_VERSION}")
	set(ONNX_INCLUDE_SUBDIR "include")
	if (WIN32)
		option(AI_ONNX_USE_CUDA "Whether to use CUDA for running ONNX. If disabled, DirectML will be used (which is still experimental, and according to our experiments, will not work on non-insider builds)." ON)
		if (AI_ONNX_USE_CUDA)
			if (NOT CUDA_FOUND)	# CUDA might have been found for Astra already
				FIND_PACKAGE(CUDA)
			endif()
			if (NOT CUDA_FOUND)
				message(SEND_ERROR "CUDA, required for ONNX, not found! Specify CUDA_TOOLKIT_ROOT_DIR, or uncheck ENABLE_AI (or AI_ONNX_USE_CUDA, if you want to use DirectML, see note there)!")
			endif()
			if ("${ONNX_VERSION}" STREQUAL "1.4.0")
				set(ONNX_HASH "BF892FCEF02AE2F060C93BF20F6DBE868B5282C907403A4820F5D037AB2AF9973BA406F6F25D1E35BE125BC02ABDDFDCEEF2C58B5ö48469E632B962B4AAB88EF2")
			elseif ("${ONNX_VERSION}" STREQUAL "1.6.0")
				set(ONNX_HASH "ADED9F8529DEDC4380D70D7755452DD99AFC18ACB1B8E6AAE4DE88E105C874287CC9406FA1A158D12E487504D574DB7B851B105A1CD47C02ACAC784496C499DA")
			elseif ("${ONNX_VERSION}" STREQUAL "1.8.1")
				set(ONNX_HASH "1657EB1111BB7FF936BE009B6BD5D104CEFAAD77E74CB166216C0DD4FB85548C9895B08028405C498D73C97E28EC2B4DF134B73919AA3E56235ABE5528215020")
			endif()

			set(ONNX_ARCHIVE "https://github.com/microsoft/onnxruntime/releases/download/v${ONNX_VERSION}/onnxruntime-win-x64-gpu-${ONNX_VERSION}.zip")
			set(ONNX_LIB_DIR "${ONNX_SRC_DIR}/lib")
		else () # for DirectML:
			set(ONNX_ARCHIVE "https://github.com/microsoft/onnxruntime/releases/download/v${ONNX_VERSION}/Microsoft.ML.OnnxRuntime.DirectML.${ONNX_VERSION}.zip")
			if ("${ONNX_VERSION}" STREQUAL "1.4.0")
				set(ONNX_HASH "DDD6961B9A6F9CC044F10BD45AEF64CFC79E091D45957AFFF68BA372A7C4D7EF2ACEC9E2679C786F72FAEF3986390B0D3F3051060994E3D6439AF828DC53BA7C")
			elseif ("${ONNX_VERSION}" STREQUAL "1.6.0")
				set(ONNX_HASH "A754E7BD62845825617A9DE5DB3F9B34451DFAADBBF46333E83A4D316F9DF4B510322642BF95989735235FDEC20036EF4611E10C37A11FB31F8DC3DEF3E774D9")
			elseif ("${ONNX_VERSION}" STREQUAL "1.8.1")
				set(ONNX_HASH "A67B596DB12CC03FBFA3B5DB57063C1928D229AD4C8271D8E4AC612EB11955BB129B27AB36EE0912E7D11A26C2DBCFE2B83F20C11EECAE37931ADBC9954DD46E")
			endif()
			set(ONNX_INCLUDE_SUBDIR "build/native/include")
			set(ONNX_LIB_DIR "${ONNX_SRC_DIR}/runtimes/win-x64/native")
		endif()
		set(ONNX_LIBRARIES "${ONNX_LIB_DIR}/onnxruntime.lib")
		if (MSVC)
			string(REPLACE "/" "\\" ONNX_WIN_LIB_DIR "${ONNX_LIB_DIR}")
			set(ADDITIONAL_LIB_PATHS "${ADDITIONAL_LIB_PATHS};${ONNX_WIN_LIB_DIR}")
		endif()
	elseif (UNIX)
		# only CUDA option avalailable for linux
		set(AI_ONNX_USE_CUDA ON)
		set(ONNX_ARCHIVE "https://github.com/microsoft/onnxruntime/releases/download/v${ONNX_VERSION}/onnxruntime-linux-x64-gpu-${ONNX_VERSION}.tgz")
		if ("${ONNX_VERSION}" STREQUAL "1.4.0")
			set(ONNX_HASH "e9803f467e2d9dc9cd2c708b2ef58ba0844f039ee54544fc5a127ae9812fae1426dbcc47b0531ca41a75d02f4cda462055e2c9d27086b7af14da3a3643d3c43c")
		elseif ("${ONNX_VERSION}" STREQUAL "1.6.0")
			set(ONNX_HASH "CCBF5065834FB1C9FCB38BE09D12C68A13BA7A5789474E9784A3908B0180E61600A5DCA0D29162E9070103D4FC2BF476190102548DC6E5A3B88386BD012B3921")
		elseif ("${ONNX_VERSION}" STREQUAL "1.8.1")
			set(ONNX_HASH "EDB03E82E40E2022B0AFAF1E60F4830465BABF95BFF98B506BE7A2F6C5AFD015F4678A57CBB83188DD6465C24A8615F00ECDF485F275A8887319581D1CBE5603")
		endif()
		set(ONNX_LIBRARIES "${ONNX_SRC_DIR}/lib/libonnxruntime.so.${ONNX_VERSION}")
	elseif(APPLE)
		message(SEND_ERROR "No supported ONNX runtime for OS X - there is only a CPU-based library, which is not supported by open_iA. Please disable ENABLE_AI option!")
		#set(ONNX_ARCHIVE "https://github.com/microsoft/onnxruntime/releases/download/v${ONNX_VERSION}/onnxruntime-osx-x64-${ONNX_VERSION}.tgz")
		#set(ONNX_HASH "91EC06DA8EA381ED6FA9089D6570A8DE9C38178323B4FF6AB0BF014A4A314FDECD617DB32864364F81DC5A9665FC66A060E17349FC058686CD451DA35943909D")
		#set(ONNX_LIBRARIES "${ONNX_SRC_DIR}/lib/libonnxruntime.${ONNX_VERSION}.dylib")
	else()
		message(SEND_ERROR "Unknown operating system, no supported ONNX runtime available. Please disable ENABLE_AI option!")
	endif()

	if (AI_ONNX_USE_CUDA)
		# see https://onnxruntime.ai/docs/reference/execution-providers/CUDA-ExecutionProvider.html#requirements
		if ("${ONNX_VERSION}" STREQUAL "1.4.0")
			set(ONNX_REQUIRED_CUDA_VERSION "10.1")
			set(ONNX_REQUIRED_CUDNN_VERSION "7.6.5")
		elseif ("${ONNX_VERSION}" STREQUAL "1.6.0")
			set(ONNX_REQUIRED_CUDA_VERSION "10.2")
			set(ONNX_REQUIRED_CUDNN_VERSION "8.0.3")
		elseif ("${ONNX_VERSION}" STREQUAL "1.8.1")
			set(ONNX_REQUIRED_CUDA_VERSION "11.0")	# though supposedly it _also_ supports CUDA 11.1 (see https://github.com/microsoft/onnxruntime/releases/tag/v1.8.1)
			if (WIN32)
				set(ONNX_REQUIRED_CUDNN_VERSION "8.0.2.39")
			else()
				set(ONNX_REQUIRED_CUDNN_VERSION "8.0.4")
			endif()
		endif()
		if (NOT "${CUDA_VERSION}" STREQUAL "${ONNX_REQUIRED_CUDA_VERSION}")
			message(WARNING "Wrong CUDA version - the CUDA variant of the ONNX runtime version ${ONNX_VERSION} requires CUDA version ${ONNX_REQUIRED_CUDA_VERSION}. Please either install this CUDA version, or choose a different ONNX runtime version!")
		endif()
		message(WARNING "onnx CUDA runtime v${ONNX_VERSION} requires cuDNN version ${ONNX_REQUIRED_CUDNN_VERSION} (I cannot check for cuDNN at the moment; so you need to make sure yourself you have installed it)!")
	endif()

	set(ONNX_PATCH_COMMAND2 "")
	# Patch onnx runtime api to have unscoped "OrtInvalid" instead of "Invalid" enum entry
	# (which might clash with an enum value in open_iA, in older versions)
	if ("${ONNX_VERSION}" STREQUAL "1.4.0")
		set(ONNX_ENUM_PATCH_LINE 212)
	elseif ("${ONNX_VERSION}" STREQUAL "1.6.0")
		set(ONNX_ENUM_PATCH_LINE 235)
	elseif ("${ONNX_VERSION}" STREQUAL "1.8.1")
		set(ONNX_ENUM_PATCH_LINE 241)
		# fix include folder bug in 1.8.1:
		set(ONNX_PATCH_COMMAND2 ${GIT_APPLY_CMD} "${CMAKE_SOURCE_DIR}/patches/onnx-1.8.1-include.patch")
	endif()
	CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/patches/onnx-runtime_c_api.h.patch.template" "${CMAKE_BINARY_DIR}/onnx-runtime_c_api.h.patch")
	set(ONNX_PATCH_COMMAND ${GIT_APPLY_CMD} "${CMAKE_BINARY_DIR}/onnx-runtime_c_api.h.patch")
	set(ONNX_INCLUDE_DIR "${ONNX_SRC_DIR}/${ONNX_INCLUDE_SUBDIR}")
	ExternalProject_Add (onnx
		URL               ${ONNX_ARCHIVE}
		URL_HASH          "SHA512=${ONNX_HASH}"
		DOWNLOAD_DIR      ${ARCHIVE_DIR}
		SOURCE_DIR        "${ONNX_SRC_DIR}"
		PATCH_COMMAND     ${ONNX_PATCH_COMMAND}
		      COMMAND     ${ONNX_PATCH_COMMAND2}
		BUILD_IN_SOURCE   1
		CONFIGURE_COMMAND ""
		BUILD_COMMAND     ""
		INSTALL_COMMAND   "")
endif()

set(OPEN_IA_CMAKE_ARGS
	"-DQT_DIR:PATH=${QT_PATH}"
	"-DITK_DIR:PATH=${ITK_DIR}"
	"-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
	"-DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}"
)
if (QT_VERSION_MAJOR GREATER_EQUAL 6)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DQt6WidgetsTools_DIR=${Qt6_DIR}WidgetsTools"
		"-DQt6CoreTools_DIR=${Qt6_DIR}CoreTools"
		"-DQt6GuiTools_DIR=${Qt6_DIR}GuiTools")
endif()
if (NOT CMAKE_CONFIGURATION_TYPES)
	list(APPEND OPEN_IA_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
endif()
if (ENABLE_OPENCL)
	list(APPEND OPEN_IA_CMAKE_ARGS
	"-DOPENCL_ROOT_DIR=${OPENCL_DIR}"
	"-DOPENCL_INCLUDE_DIRS=${OPENCL_INCLUDE_DIRS}"
	"-DOPENCL_LIBRARIES=${OPENCL_LIBRARIES}")
endif()
if (ENABLE_FILTERS)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_CommonImageFilters:BOOL=ON"
		"-DModule_ExtractSurface:BOOL=ON"
		"-DModule_FeatureCharacteristics:BOOL=ON"
		"-DModule_MetaFilters:BOOL=ON"
		"-DModule_Metrics:BOOL=ON"
		"-DModule_Segmentation:BOOL=ON")
endif()
if (ENABLE_TOOLS)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_4DCT:BOOL=ON"
		"-DModule_BoneThickness:BOOL=ON"
		"-DModule_DreamCaster:BOOL=${ENABLE_OPENCL}"
		"-DModule_DynamicVolumeLines:BOOL=ON"
		"-DModule_ElastixRegistration:BOOL=ON"
		"-DModule_FeatureAnalyzer:BOOL=ON"
		"-DModule_FeatureAnalyzerComputation:BOOL=ON"
		"-DModule_FeatureScout:BOOL=ON"
		"-DModule_FiAKEr:BOOL=ON"
		"-DModule_FoamCharacterization:BOOL=ON"
		"-DModule_FuzzyFeatureTracking:BOOL=ON"
		"-DModule_GEMSe:BOOL=ON"
		"-DModule_InSpectr:BOOL=ON"
		"-DModule_ModalityExplorer:BOOL=ON"
		"-DModule_ParameterExplorer:BOOL=ON"
		"-DModule_PolygonPrimitives:BOOL=ON"
		"-DModule_TripleHistogramTF:BOOL=ON"
		"-DModule_Uncertainty:BOOL=ON")
	# Qt 6.0 does not come with Qt Charts; any Qt 5 version ships it though, and Qt >= 6.1 does too:
	if (QT_VERSION_MAJOR LESS 6 OR QT_VERSION VERSION_GREATER_EQUAL "6.1.0")
		list(APPEND OPEN_IA_CMAKE_ARGS
			"-DModule_AdaptiveThreshold:BOOL=ON"
			"-DQt${QT_VERSION_MAJOR}Charts_DIR:PATH=${Qt${QT_VERSION_MAJOR}_DIR}Charts")
	else()
		message(WARNING "Qt 6.0 does NOT support Qt charts, thus AdaptiveThresholding module depending on that library cannot be enabled!")
	endif()
endif()
if (ENABLE_TEST)
	list(APPEND OPEN_IA_CMAKE_ARGS "-DopeniA_TESTING_ENABLED:BOOL=ON")
endif()
if (ENABLE_PRECOMPILE)
	list(APPEND OPEN_IA_CMAKE_ARGS "-DopeniA_PRECOMPILE:BOOL=ON")
endif()
if (ENABLE_ASTRA)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_AstraReconstruction:BOOL=ON"
		"-DASTRA_TOOLBOX_DIR:PATH=${ASTRA_DIR}")
	if (BUILD_ASTRA)
		list(APPEND openiA_DEPENDS astra)
	endif()
endif()
if (ENABLE_VR)
	list(APPEND OPEN_IA_CMAKE_ARGS "-DModule_VR:BOOL=ON")
	if (VTK_VERSION VERSION_LESS "9.0.0")
		list(APPEND OPEN_IA_CMAKE_ARGS
			"-DOPENVR_ROOT_DIR:PATH=${OPENVR_ROOT_DIR}")
	else()
		list(APPEND OPEN_IA_CMAKE_ARGS
			${OPENVR_LIB_AND_INCLUDE})
	endif()
	if (BUILD_BOOST)
		list(APPEND openiA_DEPENDS boost)
	endif()
	if (NOT ENABLE_ASTRA)	# if astra module is enabled, it will care for setting up boost include directory; otherwise, we need to set it manually:
		list(APPEND OPEN_IA_CMAKE_ARGS "-DBOOST_INCLUDE_DIR:PATH=${BOOST_INCLUDE_DIR}")
	endif()
endif()
if (ENABLE_AI)
	LIST(APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_AI:BOOL=ON"
		"-DONNX_RUNTIME_DIR:PATH=${ONNX_SRC_DIR}"
		"-DONNX_RUNTIME_INCLUDE_DIR:PATH=${ONNX_INCLUDE_DIR}"
		"-DONNX_RUNTIME_LIBRARIES:FILEPATH=${ONNX_LIBRARIES}"
		"-DopeniA_ONNX_CUDA:BOOL=${AI_ONNX_USE_CUDA}")
	list(APPEND openiA_DEPENDS onnx)
endif()
if (ENABLE_EIGEN)
	set(EIGEN_BIN_DIR "${CMAKE_BINARY_DIR}/eigen-${EIGEN_VERSION}")
	ExternalProject_Add (eigen
		URL               ${EIGEN_URL}
		URL_HASH          "SHA512=${EIGEN_ARCHIVE_SHA512}"
		DOWNLOAD_DIR      ${ARCHIVE_DIR}
		SOURCE_DIR        "${EIGEN_BIN_DIR}"
		BUILD_IN_SOURCE   1
        CONFIGURE_COMMAND ""
        BUILD_COMMAND     ""
		INSTALL_COMMAND   "")
	list(APPEND openiA_DEPENDS eigen)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DEIGEN3_INCLUDE_DIR:PATH=${EIGEN_BIN_DIR}")
endif()
if (ENABLE_HDF5)
	list(APPEND openiA_DEPENDS hdf5)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DHDF5_DIR:PATH=${HDF5_INSTALL_DIR}/share/cmake/hdf5")
endif()

set(OPEN_IA_GIT_TAG_OPTIONS "develop" "master" "2020.09" "2020.04" "2020.01" "2019.12" "2019.07")
selector(OPEN_IA_GIT_TAG "${OPEN_IA_GIT_TAG_OPTIONS}" "develop" "Choose the open_iA version to build.")
if ("${OPEN_IA_GIT_TAG}" STREQUAL "2019.07" AND "${ITK_VERSION}" STREQUAL "5.0.1")
	message(SEND_ERROR "open_iA version ${OPEN_IA_GIT_TAG} is not compatible with ITK version ${ITK_VERSION}!")
endif()
set(OPEN_IA_BIN_DIR "${CMAKE_BINARY_DIR}/open_iA/bin")
ExternalProject_Add (open_iA
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
	string(REPLACE "/lib/cmake/Qt${QT_VERSION_MAJOR}" "/bin" QT_LIB_DIR ${Qt${QT_VERSION_MAJOR}_DIR})
	string(REPLACE "/" "\\" QT_LIB_DIR ${QT_LIB_DIR})
	string(REPLACE "/" "\\" VTK_WIN_DIR ${VTK_DIR})
	string(REPLACE "/" "\\" ITK_WIN_DIR ${ITK_DIR})
	set(openia_cmd  "${OPEN_IA_BIN_DIR_WIN}\\x64\\$(Configuration)\\open_iA.exe")
	set(openia_path "${QT_LIB_DIR};${VTK_WIN_DIR}\\bin\\$(Configuration);${ITK_WIN_DIR}\\bin\\$(Configuration)${ADDITIONAL_LIB_PATHS}")
	set_target_properties(open_iA PROPERTIES VS_DEBUGGER_COMMAND "${openia_cmd}")
	set_target_properties(open_iA PROPERTIES VS_DEBUGGER_ENVIRONMENT "PATH=${openia_path}")
	set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT open_iA)
endif()
