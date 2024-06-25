cmake_minimum_required(VERSION 3.17...3.28 FATAL_ERROR)
# Required CMake version: as high as open_iA!
#    CMake 3.17 required for --no-tests=ignore, FindCUDAToolkit
if (CMAKE_MAJOR_VERSION GREATER 3 OR (CMAKE_MAJOR_VERSION EQUAL 3 AND CMAKE_MINOR_VERSION GREATER 15))
	# prevent all submodules to be fetched in ExternalProject_Add if GIT_SUBMODULES is ""
	CMAKE_POLICY(SET CMP0097 NEW)
endif()
project(open_iA_superbuild LANGUAGES C CXX)

include(ExternalProject)

option(VTK_USE_GIT_REPO "Whether to use git repository for VTK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly. Also, you have to take care of any eventually required patches manually (as for example 'master' branch is subject to daily change)." OFF)
option(ITK_USE_GIT_REPO "Whether to use git repository for ITK library. If disabled (default), the release archives will be used instead. Note that enabling this option might increase build times significantly." OFF)
mark_as_advanced(VTK_USE_GIT_REPO)
mark_as_advanced(ITK_USE_GIT_REPO)

option(ENABLE_ALL "Enables all optional modules and filters (see also separate ENABLE_xyz options; all except for ENABLE_PRECOMPILE and ENABLE_VTK_AVIWRITER are enabled if this is set to on. Note that unchecking this box again does NOT have any direct effect; it will not automatically set these options to unchecked or their state before. But you will have to uncheck the option if you want to disable any of the single ENABLE_xyz options affected by this setting, otherwise they will be re-enabled on next 'Configure' run) (default: disabled)" OFF)

option(ENABLE_AI "Whether to enable AI module; requires onnx runtime, which will be fetched automatically; on Windows, you can change whether CUDA or DirectML backend is chosen with the option AI_ONNX_USE_CUDA (default: disabled)" OFF)
option(ENABLE_ASTRA "Whether to build ASTRA reconstruction library and open_iA ASTRA module. This will also fetch and build boost, so enabling it will considerably increase the build time! (default: disabled)" OFF)
option(ENABLE_EIGEN "Whether to fetch and use eigen (default: disabled)" OFF)
option(ENABLE_FILTERS "Whether to build image processing filters (smoothing, segmentation, intensity transformations, geometric transformations, ...) (default: enabled)" ON)
option(ENABLE_HDF5 "Whether to fetch and build HDF5 library and use it in open_iA (default: disabled)" OFF)
option(ENABLE_OPENCL "Enables OpenCL; the DreamCaster tool depends is only enabled if this setting is enabled; enabling this option also enables some GPU-optimized ITK filters (default: disabled)" OFF)
option(ENABLE_PRECOMPILE "Whether to build open_iA with precompiled headers enabled (default: disabled; NOT included in \"ENABLE_ALL\")" OFF)
mark_as_advanced(ENABLE_PRECOMPILE)
option(ENABLE_TOOLS "Whether to build common tool modules, e.g. FeatureScout, 4DCT, GEMSe, Dynamic Volume Lines, FIAKER, ... (default: enabled)" ON)
option(ENABLE_TEST "Whether to enable build of tests runners and the capability to submit CDash test runs (default: disabled)" OFF)
option(ENABLE_VR "Enable VR module; requires OpenVR or OpenXR SDK; these can automatically be fetched, depending on ENABLE_OPENVR / ENABLE_OPENXR; also boost (includes) are required (default: disabled)" OFF)
if (ENABLE_VR)
	option(ENABLE_OPENVR "Enable automatic fetching of OpenVR SDK, if ENABLE_VR is enabled. OpenVR is more thoroughly tested than OpenXR, but OpenVR will probably not receive future maintenance (default: enabled)" ON)
endif()

option(BUILD_VTK "Build VTK in the superbuild. If disabled, you need to set VTK_DIR to an existing VTK build (default: enabled)" ON)
option(BUILD_ITK "Build ITK in the superbuild. If disabled, you need to set ITK_DIR to an existing ITK build (default: enabled)" ON)
#option(FETCH_OPENVR "Fetch release of OpenVR SDK. If disabled, you need to set OPENVR_DIR to an existing OpenVR SDK directory" ON)
#option(FETCH_EIGEN "Fetch release of eigen. If disabled, you need to set EIGEN3_DIR to an existing Eigen 3.x directory" ON)
#option(BUILD_HDF5 "Fetch and build HDF5. If disabled, you need to set HDF5_DIR to an existing HDF5 build" ON)
#option(FETCH_ONNX "Fetch required ONNX runtime. If disabled, you need to set ONNX_RUNTIME_DIR to an existing ONNX runtime directory" ON) # could be a bit complicated to implement, since we would have to check the type of runtime (CUDA/DirectML/CPU) and set according flags

option(DISABLE_LIB_WARNINGS "Disables all compilation warnings on the built libraries" OFF)
mark_as_advanced(DISABLE_LIB_WARNINGS)

if (MSVC AND BUILD_VTK)
	option(ENABLE_VTK_AVIWRITER "Enable usage of *.avi (an old Windows movie file format) writer in open_iA. Note that enabling this might cause linker errors, since we cannot reliably determine whether VTK builds the required parts or not." OFF)
	mark_as_advanced(ENABLE_VTK_AVIWRITER)
endif()

if (MSVC AND MSVC_VERSION LESS 1920)
	message(WARNING "You are using a Visual Studio versions < 2019. Note that such old versions are not supported, you are on your own!")
endif()


# Find proper way to append flags here...
#if(MSVC)
#	set(DISABLE_WARNINGS_OPT "\"-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} /W0\"")
#else()
#	set(DISABLE_WARNINGS_OPT "\"-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} -w\"")
#endif()
#

function(selector varname options default desc)
	list(FIND options "${${varname}}" options_index)
	if (${options_index} EQUAL -1)
		if (DEFINED ${varname})
			message(WARNING "Invalid ${varname}, resetting to default ${default}!")
		endif()
		set(${varname} "${default}" CACHE STRING "${desc}" FORCE)
	endif()
	set_property(CACHE ${varname} PROPERTY STRINGS ${options})
endfunction()

set(AVX_OPTIMIZATION_OPTIONS "off" "AVX" "AVX2")
set(AVX_OPTIMIZATION_DEFAULT "off")
set(AVX_OPTIMIZATION_DESC "Choose whether code should use AVX instructions, and if so which (AVX/AVX2)  (default: ${AVX_OPTIMIZATION_DEFAULT}).")
selector(AVX_OPTIMIZATION "${AVX_OPTIMIZATION_OPTIONS}" ${AVX_OPTIMIZATION_DEFAULT} ${AVX_OPTIMIZATION_DESC})
mark_as_advanced(AVX_OPTIMIZATION)

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
	set(ENABLE_OPENVR "ON" CACHE BOOL "" FORCE)
	set(ENABLE_OPENXR "ON" CACHE BOOL "" FORCE)
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

message(STATUS "Compilers: C++: ${CMAKE_CXX_COMPILER}")

if (WIN32 AND NOT MSVC)
	message(WARNING "Building under windows with non-VS generator/compiler is currently not tested, expect errors! Please report any findings under https://github.com/3dct/open_iA-superbuild/issues!")
endif()

find_package(QT NAMES Qt6 COMPONENTS Widgets OpenGLWidgets REQUIRED)  # for now, keep this way of searching for Qt - avoids having to manually set the ...Tools directories!
#if (QT_VERSION VERSION_LESS "6.0.0")
#	message(FATAL_ERROR "Your Qt version is too old. Please use Qt >= 6.0.0")
#endif()
if (QT_DIR AND NOT Qt6_DIR)
	set(Qt6_DIR ${QT_DIR} CACHE PATH "" FORCE)
	set(Qt6_DIR ${QT_DIR})
	set(Qt6CoreTools_DIR ${QT_DIR}CoreTools CACHE PATH "" FORCE)
	set(Qt6GuiTools_DIR ${QT_DIR}GuiTools CACHE PATH "" FORCE)
	set(Qt6WidgetsTools_DIR ${QT_DIR}WidgetsTools CACHE PATH "" FORCE)
endif()
if (QT_VERSION VERSION_GREATER_EQUAL "6.4.0")
	find_package(Qt${QT_VERSION_MAJOR} REQUIRED Concurrent Gui OpenGL Svg Widgets Xml OPTIONAL_COMPONENTS Charts HttpServer WebSockets)
else()
	find_package(Qt${QT_VERSION_MAJOR} REQUIRED Concurrent Gui OpenGL Svg Widgets Xml OPTIONAL_COMPONENTS Charts WebSockets)
endif()
set(QT_PATH ${Qt${QT_VERSION_MAJOR}_DIR})
set(QTFEATURES "")
if (Qt${QT_VERSION_MAJOR}Charts_FOUND)
    list(APPEND QTFEATURES "Charts")
endif()
if (Qt${QT_VERSION_MAJOR}HttpServer_FOUND)
    list(APPEND QTFEATURES "HttpServer")
endif()
if (Qt${QT_VERSION_MAJOR}WebSockets_FOUND)
    list(APPEND QTFEATURES "WebSockets")
endif()
message(STATUS "Qt: ${QT_VERSION} (features: ${QTFEATURES}) in ${QT_DIR}")

find_package(Git)
if (NOT GIT_FOUND)
	message(SEND_ERROR "Git not found, but it is required for some patches!")
endif()
set(GIT_APPLY_OPTS --ignore-whitespace)
#--whitespace=nowarn --inaccurate-eof) # potential additional options in case of whitespace errors....
set(GIT_APPLY_CMD "${GIT_EXECUTABLE}" apply ${GIT_APPLY_OPTS})

# build types
if (NOT CMAKE_CONFIGURATION_TYPES)
	set(BUILD_TYPE_OPTIONS "Release" "Debug" "RelWithDebInfo" "MinSizeRel")
	selector(CMAKE_BUILD_TYPE "${BUILD_TYPE_OPTIONS}" "Release" "Choose the build configuration.")
endif()

## VTK

set(VTK_VERSION_OPTIONS "9.1.0" "9.2.2" "9.2.6" "9.3.0")
set(VTK_VERSION_DESC "Choose the VTK version to build (only used if VTK_USE_GIT_REPO is DISABLED; if VTK_USE_GIT_REPO is enabled, use VTK_GIT_TAG instead).")
selector(VTK_VERSION "${VTK_VERSION_OPTIONS}" "9.3.0" ${VTK_VERSION_DESC})

set(VTK_SMP_TYPE_OPTIONS "Sequential" "OpenMP" "TBB" "STDThread")
set(VTK_SMP_TYPE_DEFAULT "STDThread")
set(VTK_SMP_TYPE_DESC "Choose the SMP implementation to use in the VTK build to speed up filters with parallel implementations - available are sequential (no parallelization), OpenMP, and TBB (Intel Thread Building Blocks - will require installation of Intel OneAPI SDK)  (default: ${VTK_SMP_TYPE_DEFAULT}).")
selector(VTK_SMP_TYPE "${VTK_SMP_TYPE_OPTIONS}" ${VTK_SMP_TYPE_DEFAULT} ${VTK_SMP_TYPE_DESC})
mark_as_advanced(VTK_SMP_TYPE)

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

if (NOT VTK_USE_GIT_REPO)
	if ("${VTK_VERSION}" STREQUAL "9.1.0")
		set(VTK_TGZ_SHA512 "B2C4BE8795FA082E0776E6FFDB9A3FD88DBB235841A8369A34EBE26CF4C5FCB1610FCCA987BE314510629DA3EDC74EE76C0CE2CC88D5EF4B099AC550AC892B0A")
	elseif ("${VTK_VERSION}" STREQUAL "9.2.2")
		set(VTK_TGZ_SHA512 "11B8203A4EFAAC0CDA9EA46ACF58C0D88C18F55BFD52E284F74EF313646913D194D3885DA9D9FA7222D926502522505BFDF9D719F8C09790DAB1315E2161610A")
	elseif ("${VTK_VERSION}" STREQUAL "9.2.6")
		set(VTK_TGZ_SHA512 "F2328CAAE959D583299B7FD57205F3DD76F87C8C1EE78653E85D44CAB085295BF7BF88B3F6A2B960A57DF96CCB32049337EBCCB067ECDE6D84D25EDA636196BC")
	elseif ("${VTK_VERSION}" STREQUAL "9.3.0")
		set(VTK_TGZ_SHA512 "2722DC8186779DF1AF5C534F26B20186EA33CB3BCA2CEF8CB333542B12E738382616E26E8D4D0341CB98C30403F8A896D77DDB04ADB07E26D0D548C4A34B7D2F")
	endif()
	string(SUBSTRING "${VTK_VERSION}" 0 3 VTK_SHORT_VERSION)
	set(VTK_URL "https://www.vtk.org/files/release/${VTK_SHORT_VERSION}/VTK-${VTK_VERSION}")
	set(VTK_URL "${VTK_URL}.tar.gz")
	set(VTK_ARCHIVE_SHA512 ${VTK_TGZ_SHA512})
endif()

## ITK

set(ITK_VERSION_OPTIONS "5.1.2" "5.2.1" "5.3.0" "5.4.0")

set(ITK_VERSION_DESC "Choose the ITK version to build (only used if ITK_USE_GIT_REPO is DISABLED; if ITK_USE_GIT_REPO is enabled, use ITK_GIT_TAG instead).")
selector(ITK_VERSION "${ITK_VERSION_OPTIONS}" "5.3.0" "${ITK_VERSION_DESC}")
if (ITK_USE_GIT_REPO)
	# only allow git tags >= 5.4.0, so that we can skip patching (which would require more complex logic below...)
	set(ITK_GIT_TAG_OPTIONS "")
	list(APPEND ITK_GIT_TAG_OPTIONS "v5.4.0") # version 5.4.0 is tagged but not officially released yet...
	set(ITK_GIT_TAG_DEFAULT "master")
	list(APPEND ITK_GIT_TAG_OPTIONS ${ITK_GIT_TAG_DEFAULT})
	selector(ITK_GIT_TAG "${ITK_GIT_TAG_OPTIONS}" ${ITK_GIT_TAG_DEFAULT} "Choose the ITK git tag to use in build.")
	set(ITK_REPOSITORY "https://github.com/InsightSoftwareConsortium/ITK.git")
	# set(ITK_REPOSITORY "https://github.com/Kitware/ITK.git")
	# set(ITK_REPOSITORY "http://itk.org/ITK.git")
else()
	if ("${ITK_VERSION}" STREQUAL "5.1.2")
		set(ITK_ZIP_SHA512 "9EC1B3BED6C2F0DDE5103976ABE3EA7BD1EBA943D28BEF3EB380E65A8728AEA94DA39B322ED3AD4C77F6F15E7BCCA5CEDEE6429513A86CC01778051F6FD26D12")
		set(ITK_TGZ_SHA512 "B7702B4933A0491110E22E98FAF8534F5DB49B6218830962EA067CCDB2FD5666ED41EE61A12CEE66484F72B1E2225235D0762F0863EAC4568D2C782761E5B03C")
	elseif ("${ITK_VERSION}" STREQUAL "5.2.1")
		set(ITK_ZIP_SHA512 "E07132A0BE208B53D0A44918D291C47F9FCEDB78C41C6F1EA7A9179FA38A532099F06CE3E1A3993783D984734F9180F428372B5DFC7E17D05733A259B9FBBC1B")
		set(ITK_TGZ_SHA512 "6786E39CDF3D0C3A31ABD1E23481E30F6DC9DAC189FFE372DDE3DB688F2F57686A8BEB321778327E1FF683ED844D41F1DEE937B0BA542B2365E2195DFCA398C7")
	elseif ("${ITK_VERSION}" STREQUAL "5.3.0")
		set(ITK_ZIP_SHA512 "CE778628B297E77DFF5C328BE16409F3596EA2D9675D67106C3360082C66617BF8870C58BBF20D3B92316B26048CA1EB1113AECB5DB1F09455D6FE8F39022F6C")
		set(ITK_TGZ_SHA512 "29359839C0FB13ACD430410E6ADADBECB4D9D8CB4871A0ABA0AC67B539C235354A591655DD654F43DAA5F035A33721671D665EEE8A4A129A9D0D3419E2356E97")
	elseif ("${ITK_VERSION}" STREQUAL "5.4.0")
		set(ITK_ZIP_SHA512 "49376E56B08A59736FF9876FAF04D4A20B295C23FA7A2C715C2830EFDEC0EFC3FB272BCBDEA9894D58902151B539506025CAF73FB670945D8F937CD3544EACEA")
		set(ITK_TGZ_SHA512 "3A98ECECF258AAC545F094DD3E97918C93CC82BC623DDF793C4BF0162AB06C83FBFD4D08130BDEC6E617BDA85DD17225488BC1394BC91B17F1232126A5D990DB")
	endif()
	if ("${ITK_VERSION}" STREQUAL "5.4.0")
		set(ITK_URL "https://github.com/InsightSoftwareConsortium/ITK/archive/refs/tags/v${ITK_VERSION}")
	else()
		set(ITK_URL "https://github.com/InsightSoftwareConsortium/ITK/releases/download/v${ITK_VERSION}/InsightToolkit-${ITK_VERSION}")
	endif()
	if (WIN32)
		set(ITK_URL "${ITK_URL}.zip")
		set(ITK_ARCHIVE_SHA512 ${ITK_ZIP_SHA512})
	else()
		set(ITK_URL "${ITK_URL}.tar.gz")
		set(ITK_ARCHIVE_SHA512 ${ITK_TGZ_SHA512})
	endif()
endif()


# TODO:
#   - check behavior when there are changes in git (and tag not the one referenced in ExternalProject_Add) / offline

if (ENABLE_VR)
	# OpenVR SDK
	if (ENABLE_OPENVR)
		set(OPENVR_VERSION_OPTIONS "1.26.7" "2.0.10" "2.2.3" "2.5.1")
		selector(OPENVR_VERSION "${OPENVR_VERSION_OPTIONS}" "2.5.1" "Choose the OpenVR version to fetch.")
		if ("${OPENVR_VERSION}" STREQUAL "1.26.7")
			set(OPENVR_ARCHIVE_SHA512 "8F8B88F6CE8AD3412C2EE9D76157216AF291C44BF0A898C21D96E3CE8BC76DC39CC48C330DEBDEEE34C0DC1F3943F73BEC92284D1B47AE7A25176A756DA700A2")
		elseif ("${OPENVR_VERSION}" STREQUAL "2.0.10")
			set(OPENVR_ARCHIVE_SHA512 "3E7E4073763D13F045E020672D6DEDAC7CF7E81EA6417F375DEC19A79AD83B0BAD26D03DDC8F2374F2D01DF6419701C1590A3BFBCDC7A943E1A1DFB7B36B9BA9")
		elseif ("${OPENVR_VERSION}" STREQUAL "2.2.3")
			set(OPENVR_ARCHIVE_SHA512 "DC442D9AD338B6FD2DAA9EDB0CAF3A158FB1405426AC71DEE17B413010DB31348B24B42C4263E1AE6600EB823EA7E23AB6FF5270DF977C62995FBDB8FF2F2D60")
		elseif ("${OPENVR_VERSION}" STREQUAL "2.5.1")
			set(OPENVR_ARCHIVE_SHA512 "7639BB313034B85A6A3B32764289B9244EB305650B3E898D0F029B4DED5EDADFCA811C17145D0C6B474FBDC381044651F0DFACAB0B3A95B2E0956D563A92B351")
		endif()
		set(OPENVR_URL "https://github.com/ValveSoftware/openvr/archive/v${OPENVR_VERSION}.zip")
		if (NOT WIN32)
			message(WARNING "OpenVR support under non-Windows operating systems is currently untested. Please report any problems you should encounter under https://github.com/3dct/open_iA/issues!")
		endif()
	endif()
	if (VTK_VERSION VERSION_GREATER_EQUAL "9.2.0")
		option(ENABLE_OPENXR "Enable automatic fetching of OpenXR SDK, if ENABLE_VR is enabled. OpenXR is less tested than OpenVR and does not support ImNDT's built-in AR mode at the moment, but it has broader support for devices and will probably be better supported in future (default: disabled)" OFF)
		if (ENABLE_ALL)
			set(ENABLE_OPENXR "ON" CACHE BOOL "" FORCE)
		endif()
	endif()
	# OpenXR SDK
	if (ENABLE_OPENXR)
		if (VTK_VERSION VERSION_LESS "9.2.0")
			message(WARNING "OpenXR is only supported for VTK >= 9.2.0, but currently, VTK ${VTK_VERSION} is configured to be built! If you need OpenXR, please switch to an appropriate VTK version!")
		else()
			set(OPENXR_VERSION_OPTIONS  "1.0.33" "1.0.34" "1.1.36")
			selector(OPENXR_VERSION "${OPENXR_VERSION_OPTIONS}" "1.0.34" "Choose the OpenXR SDK version to fetch.")
			if ("${OPENXR_VERSION}" STREQUAL "1.0.33")
				set(OPENXR_ARCHIVE_SHA512 "41039EFE52CE23116FA6A4C79150AD342E392995F743975275AB916FA8EE979180A5C0ABF064CD4BA38C4118174E0BC7596956EF7755DD51A5B87C955C38D352")
			elseif ("${OPENXR_VERSION}" STREQUAL "1.0.34")
				set(OPENXR_ARCHIVE_SHA512 "81A03B3B96D80D6A06DAAD8B5ABB5C37ECB9242CE6D6EAD3737FD4570F44B156CFE401249EAA5BA6339B9B8627EF128C509DE0853E3A30D63381B910F6571E58")
			elseif ("${OPENXR_VERSION}" STREQUAL "1.1.36")
				message(WARNING "Note: open_iA (more specifically, the VTK<->OpenXR bindings that open_iA uses) are currently not compatible with 1.1.36!")
				set(OPENXR_ARCHIVE_SHA512 "5FE4B60BC467D39E7E3F459A684848A8850D2CF95A60B0D0169B2A958DF6556F2D888A2CA800B0784766570E4C5AF45CF21FEF7A239321BE133CF339FF436BE9")
			endif()
			if (WIN32)
				set(OPENXR_URL "https://github.com/KhronosGroup/OpenXR-SDK-Source/releases/download/release-${OPENXR_VERSION}/openxr_loader_windows-${OPENXR_VERSION}.zip")
			else()
				message(SEND_ERROR "Fetching OpenXR is not supported currently under any other system than Windows! You might have more luck with OpenVR (set ENABLE_OPENVR to TRUE, ENABLE_OPENXR to FALSE); but OpenVR support under non-Windows operating systems for open_iA is also currently untested!")
			endif()
		endif()
	endif()
endif()

## Astra Toolbox

if (ENABLE_ASTRA)
	set(ASTRA_REPOSITORY "https://github.com/astra-toolbox/astra-toolbox.git")
	set(ASTRA_TAG_OPTIONS "v2.0.0" "v2.1.0" "master")
	set(ASTRA_TAG_DESC "Choose the astra version (tag) to fetch and build.")
	selector(ASTRA_TAG "${ASTRA_TAG_OPTIONS}" "v2.1.0" "${ASTRA_TAG_DESC}")
endif()

if (BUILD_BOOST)
	set(BOOST_VERSION_OPTIONS "1.83.0" "1.84.0" "1.85.0")
	selector(BOOST_VERSION "${BOOST_VERSION_OPTIONS}" "1.85.0" "Choose the boost library version to build.")
	if ("${BOOST_VERSION}" STREQUAL "1.83.0")
		set(BOOST_ARCHIVE_SHA512 "DF99CC2E7B41CC5A5F545A34A37F20562ED8B43C1EE4D0AD6EF0D5C86F8CE83D1B786CCCC9EE6DA3FCB197235F63274797BF43BDA08ACA4ED55BA9864992B0B5")
	elseif ("${BOOST_VERSION}" STREQUAL "1.84.0")
		set(BOOST_ARCHIVE_SHA512 "01604FB5560C5F82C953E82306B70BFC95404DC730EAEB4A13F5A2566D12208EC9CE7046BCA4CF92B1260D6E905F0CC1C1235F37E3574FF5BBB675745A3C4848")
	elseif ("${BOOST_VERSION}" STREQUAL "1.85.0")
		set(BOOST_ARCHIVE_SHA512 "F090B0CD97D0B0037013EAD8928F4773B9009D1ACA57ACE1359CACFB47805DBA8FB52EE410CF6CAC47B9C981DB21457A9D278335D378AE9BE94358392A4CDBFA")
		if (CUDAToolkit_FOUND AND CUDAToolkit_VERSION VERSION_LESS 12)
			message(WARNING "Note that with boost 1.85, CUDA (encountered with 11.7) or the Astra build with it might report errors like `error : identifier \"__builtin_FUNCSIG\" is undefined` during build! Choose an older version (< 1.85) of boost here, or install a newer version of CUDA (tested to work with 12.4)")
		endif()
	endif()
	string(SUBSTRING "${BOOST_VERSION}" 0 4 BOOST_SHORTVERSION)
	string(REPLACE "." "_" BOOST_SHORTVER "${BOOST_SHORTVERSION}")
	if (MSVC)
		if (MSVC_VERSION LESS 1930)    # Visual Studio 2019 / Toolset 14.2
			set(MSVC_TOOLSET "v142")
			set(BOOST_MSVC_TOOLSET "msvc-14.2")
			set(BOOST_LIB_SUFFIX "vc142")
		else()    # Visual Studio 2022 / Toolset 14.3x -> not known yet what VS > 2022 will have; but 1940 is still a VS2022 version (17.10)
			set(MSVC_TOOLSET "v143")
			set(BOOST_MSVC_TOOLSET "msvc-14.3")
			set(BOOST_LIB_SUFFIX "vc143")
		endif()
	endif()
endif()

if (ENABLE_EIGEN)
	option(EIGEN_USE_GIT_REPO "Whether to use git repository for eigen library. If disabled (default), the release archives will be used instead." ON)
	if (NOT EIGEN_USE_GIT_REPO)
		 message(WARNING "EIGEN_USE_GIT_REPO is OFF. Note that for recent builds of open_iA (with C++ standard increased to C++20) this setting currently needs to be ON, since no released version of eigen (3.4.0 at this time) seems compatible with C++20 - expect errors like `error: unknown type name 'EIGEN_EMPTY_STRUCT_CTOR'`")
		set(EIGEN_VERSION_OPTIONS "3.3.7" "3.3.9" "3.4.0")
		selector(EIGEN_VERSION "${EIGEN_VERSION_OPTIONS}" "3.4.0" "Choose the eigen library version to use.")
		if ("${EIGEN_VERSION}" STREQUAL "3.3.7")
			set(EIGEN_ARCHIVE_SHA512 "6C19DF0271503DDCE61724729031A3B885D86F35D1F24363A76F9E300234D3257FF56020ACE624B2C011E17CBA6D69A3014508C5E6DC3436BBE331A2129E2175")
		elseif ("${EIGEN_VERSION}" STREQUAL "3.3.9")
			set(EIGEN_ARCHIVE_SHA512 "3B06DA583037649AD42190BD018DDF40B85179AD0877EBD7C1CA7B8498A1453EED25919DE27D3BB737754086E1513408E7DE4A2110D51912F2E5503E1AB7D838")
		elseif ("${EIGEN_VERSION}" STREQUAL "3.4.0")
			set(EIGEN_ARCHIVE_SHA512 "C8BBB447D1AC18431205C3EAAD07858FEB65FE0BA2CD37CF2AD2ED42B7D68D7DB2A84EA56475464C26D330FCEAFC1012838D39B1D74A7DCFFC0F515AE40959EA")
		endif()
		set(EIGEN_URL "https://gitlab.com/libeigen/eigen/-/archive/${EIGEN_VERSION}/eigen-${EIGEN_VERSION}.zip")
	else()
		set(EIGEN_URL "https://gitlab.com/libeigen/eigen.git")
	endif()
endif()

if (ENABLE_HDF5)
	set(HDF5_VERSION_OPTIONS "1.14.4.3")
	selector(HDF5_VERSION "${HDF5_VERSION_OPTIONS}" "1.14.4.3" "Choose the HDF5 library version to build.")
	string(SUBSTRING "${HDF5_VERSION}" 0 4 HDF5_SHORT_VERSION)
	set(HDF5_URL "https://github.com/HDFGroup/hdf5/archive/refs/tags/hdf5_${HDF5_VERSION}.zip")
	if (HDF5_VERSION STREQUAL "1.14.4.3")
		set(HDF5_ARCHIVE_SHA512   "F933F16AC27B73AE72382AF69A0E2411BD14D694D6C57A52521C7E4BD0B0C5FAE39C12DB58C83226388E5D7BDDFC2E40A674A4DD79F20DC0BD22DEA496587F72")
	else()
		message(SEND_ERROR "Invalid HDF5 version!")
	endif()
endif()

if (MSVC)
	set(ADDITIONAL_LIB_PATHS "") # for additional lib paths to be added to MSVC debugger environment path
endif()

set(ITK_DEPENDS "")

if (BUILD_VTK)
	if (VTK_VERSION VERSION_LESS "9.1.0" AND "${VTK_SMP_TYPE}" STREQUAL "STDThread")
		message(SEND_ERROR "VTK only supports VTK_SMP_TYPE = STDThread with version 9.1.0 or higher; you have selected ${VTK_VERSION}. Please either chose a different VTK_SMP_TYPE or a higher VTK_VERSION!")
	endif()
	set(VTK_CMAKE_ARGS
		"-DBUILD_SHARED_LIBS:BOOL=On"
		"-DVTK_LEGACY_REMOVE:BOOL=On"
		"-DQt6_DIR:PATH=${Qt6_DIR}"
		"-DQt6WidgetsTools_DIR:PATH=${Qt6_DIR}WidgetsTools"
		"-DQt6CoreTools_DIR:PATH=${Qt6_DIR}CoreTools"
		"-DQt6GuiTools_DIR:PATH=${Qt6_DIR}GuiTools"
		"-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
		"-DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}"
		"-DVTK_SMP_IMPLEMENTATION_TYPE=${VTK_SMP_TYPE}"
		"-DVTK_BUILD_TESTING:BOOL=OFF"
		"-DVTK_MODULE_ENABLE_VTK_GUISupportQt=YES"
		"-DVTK_MODULE_ENABLE_VTK_GUISupportQtSQL=NO"
		"-DVTK_MODULE_ENABLE_VTK_RenderingQt=YES"
		"-DVTK_MODULE_ENABLE_VTK_RenderingContextOpenGL2=YES"
		"-DVTK_MODULE_ENABLE_VTK_ViewsQt=YES"
	)
	#if (DISABLE_LIB_WARNINGS)
	#	list(APPEND VTK_CMAKE_ARGS ${DISABLE_WARNINGS_OPT})
	#endif()
	if (NOT CMAKE_CONFIGURATION_TYPES)
		list(APPEND VTK_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
	endif()
	if (WIN32 AND VTK_VERSION VERSION_GREATER_EQUAL "9.1.0")
		list(APPEND VTK_CMAKE_ARGS "-DVTK_USE_VIDEO_FOR_WINDOWS=ON")
	endif()
	set(VTK_DEPENDS "")
	set(VTK_PATCH_COMMAND ${GIT_APPLY_CMD})
	if (ENABLE_VR)
		if (ENABLE_OPENVR)
			set(OPENVR_ROOT_DIR "${CMAKE_BINARY_DIR}/openvr-${OPENVR_VERSION}")
			set(OPENVR_INCLUDE_DIR "${OPENVR_ROOT_DIR}/headers")
			if (WIN32)
				set(OPENVR_LIB_FILES "${OPENVR_ROOT_DIR}/lib/win64/openvr_api.lib")
			elseif (UNIX)
				set(OPENVR_LIB_FILES "${OPENVR_ROOT_DIR}/lib/linux64/libopenvr_api.so")
			elseif (APPLE)
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
			list(APPEND VTK_DEPENDS OpenVR)
			list(APPEND VTK_CMAKE_ARGS
				"-DVTK_MODULE_ENABLE_VTK_RenderingOpenVR:BOOL=YES"
				${OPENVR_LIB_AND_INCLUDE})
			if (MSVC)
				string(REPLACE "/" "\\" OPENVR_WIN_DIR "${OPENVR_ROOT_DIR}")
				set(ADDITIONAL_LIB_PATHS "${ADDITIONAL_LIB_PATHS};${OPENVR_WIN_DIR}\\bin\\win64")
			endif()
			if (VTK_VERSION VERSION_GREATER "9.1.0" AND VTK_VERSION VERSION_LESS "9.3.0" AND (NOT VTK_USE_GIT_REPO OR NOT VTK_GIT_TAG STREQUAL "master"))  # so far, any version
				# apply Open(X/V)R patch (see https://gitlab.kitware.com/vtk/vtk/-/merge_requests/9620):
				list(APPEND VTK_PATCH_COMMAND "${CMAKE_SOURCE_DIR}/patches/vtk-9.2-vr-9620.patch")
			endif()
		endif()
		if (ENABLE_OPENXR AND VTK_VERSION VERSION_GREATER_EQUAL "9.2.0")
			set(OPENXR_ROOT_DIR "${CMAKE_BINARY_DIR}/openxr-${OPENXR_VERSION}")
			set(OPENXR_INCLUDE_DIR "${OPENXR_ROOT_DIR}/include")
			set(VTK_OPENXR_INCLUDE_DIR ${OPENXR_INCLUDE_DIR})
			if ( VTK_VERSION VERSION_LESS_EQUAL "9.3.0")
				set(VTK_OPENXR_INCLUDE_DIR "${OPENXR_ROOT_DIR}/include/openxr")
			endif()
			#if (WIN32)
				set(OPENXR_LIBRARY "${OPENXR_ROOT_DIR}/x64/lib/openxr_loader.lib")
			#endif()
			set(OPENXR_LIB_AND_INCLUDE
				"-DOpenXR_INCLUDE_DIR:PATH=${OPENXR_INCLUDE_DIR}"
				"-DOpenXR_LIBRARY:FILEPATH=${OPENXR_LIBRARY}")
			set(VTK_OPENXR_LIB_AND_INCLUDE
				"-DOpenXR_INCLUDE_DIR:PATH=${VTK_OPENXR_INCLUDE_DIR}"
				"-DOpenXR_LIBRARY:FILEPATH=${OPENXR_LIBRARY}")
			set(OPENXR_DIR "-DOpenXR_DIR:PATH=${OPENXR_ROOT_DIR}/x64/lib/cmake/openxr")
			ExternalProject_Add (OpenXR
				URL               ${OPENXR_URL}
				URL_HASH          "SHA512=${OPENXR_ARCHIVE_SHA512}"
				DOWNLOAD_DIR      ${ARCHIVE_DIR}
				SOURCE_DIR        "${OPENXR_ROOT_DIR}"
				BUILD_IN_SOURCE   1
				CONFIGURE_COMMAND ""
				BUILD_COMMAND     ""
				INSTALL_COMMAND   "")
			list(APPEND VTK_DEPENDS OpenXR)
			list(APPEND VTK_CMAKE_ARGS "-DVTK_MODULE_ENABLE_VTK_RenderingOpenXR:BOOL=YES")
			if (VTK_VERSION VERSION_GREATER "9.3.0" OR (VTK_USE_GIT_REPO AND VTK_GIT_TAG STREQUAL "master"))
				# new: use CMake config provided by newer OpenXR versions:
				list(APPEND VTK_CMAKE_ARGS ${OPENXR_DIR})
			else()
				# old (VTK 9.3.0 or older): use include and lib directory
				list(APPEND VTK_CMAKE_ARGS ${VTK_OPENXR_LIB_AND_INCLUDE})
			endif()
			if (MSVC)
				string(REPLACE "/" "\\" OPENXR_WIN_DIR "${OPENXR_ROOT_DIR}")
				set(ADDITIONAL_LIB_PATHS "${ADDITIONAL_LIB_PATHS};${OPENXR_WIN_DIR}\\x64\\bin")
			endif()
		endif()
	endif()
	if (VTK_VERSION VERSION_LESS "9.3.0")
		# let's apply the fix for parallel coordinate charts potentially causing a crash
		# on mouse move (see https://gitlab.kitware.com/vtk/vtk/-/issues/17441):
		set(VTK_PARALLEL_COORDINATE_PATCH_LINE 492)
		CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/patches/vtk-parallelCoordinates.patch.template" "${CMAKE_BINARY_DIR}/vtk-parallelCoordinates.patch")
		list(APPEND VTK_PATCH_COMMAND "${CMAKE_BINARY_DIR}/vtk-parallelCoordinates.patch")
	endif()
	if (VTK_VERSION VERSION_EQUAL "9.3.0")
		# fix compilation error for VS: https://gitlab.kitware.com/vtk/vtk/-/issues/19166
		list(APPEND VTK_PATCH_COMMAND "${CMAKE_SOURCE_DIR}/patches/vtk-9.3-vs-debug.patch")
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
	list(APPEND ITK_DEPENDS VTK)
else()
	set(VTK_DIR "${VTK_DIR}" CACHE PATH
		"VTK build folder.")
endif()

if (ENABLE_OPENCL)
	option(BUILD_OPENCL "Build OpenCL ICD loader. If disabled, you need to specify existing OpenCL Headers / C++ Headers / ICD loader!" ON)
	if (BUILD_OPENCL)
		set(OPENCL_ROOT_DIR "${CMAKE_BINARY_DIR}/OpenCL" CACHE PATH "OpenCL ICD loader directory" FORCE)
		set(OpenCL_DIR "${OPENCL_ROOT_DIR}/install")

		# C Headers
		set(OPENCL_HEADERS_REPO "https://github.com/KhronosGroup/OpenCL-Headers.git")
		set(OPENCL_VERSION_OPTIONS "v2024.05.08" "v2023.12.14" "v2023.04.17" "v2023.02.06")
		set(OPENCL_VERSION_DESC "Choose the version of OpenCL C/C++ headers and ICD loader to build.")
		selector(OPENCL_VERSION "${OPENCL_VERSION_OPTIONS}" "v2023.12.14" ${OPENCL_VERSION_DESC})
		# other available versions: (for both OpenCL-ICD-Loader and OpenCL-Headers) see github tags; of note:
		#   - v2022.05.18  known to cause error (`OpenCL/icdloader/test/layer/icd_print_layer_generated.c:946:50: error: expected function body after function declarator`) and warning (`CL_TARGET_OPENCL_VERSION is not a valid value (100, 110, 120, 200, 210, 220). Defaulting to 220 (OpenCL 2.2)")`)
		#   - v2020.12.18  not tested; first version to contain CMakeLists.txt file
		#   - v2020.06.16  known to work well; previously used, hard-coded;
		#   - v2020.03.13  known to make problems on linux (`multiple definition of ret_val`)
		set(OPENCL_HEADERS_CMAKE_ARGS
			"-DCMAKE_INSTALL_PREFIX:PATH=${OpenCL_DIR}"
			"-DOPENCL_HEADERS_BUILD_CXX_TESTS:BOOL=OFF"
			"-DBUILD_TESTING:BOOL=OFF"
		)
		ExternalProject_Add (OpenCL_Headers
			GIT_REPOSITORY    ${OPENCL_HEADERS_REPO}
			GIT_TAG           "${OPENCL_VERSION}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        ${OPENCL_HEADERS_CMAKE_ARGS}
			SOURCE_DIR        "${OPENCL_ROOT_DIR}/Headers/src.git"
			BINARY_DIR        "${OPENCL_ROOT_DIR}/Headers/bin"
		)
		set(OpenCLHeaders_DIR "${OpenCL_DIR}/share/cmake/OpenCLHeaders")

		# C++ Headers
		set(OPENCL_CLHPP_HEADERS_REPO "https://github.com/KhronosGroup/OpenCL-CLHPP.git")
		#set(OPENCL_CLHPP_VERSION_OPTIONS "v2023.04.17")  # slightly different than Headers and ICD-Loader versions, but seem to have unified for recent versions
		#set(OPENCL_CLHPP_VERSION_DESC "Choose the version of OpenCL C++ headers (OpenCL-CLHPP) to build.")
		#selector(OPENCL_CLHPP_VERSION "${OPENCL_CLHPP_VERSION_OPTIONS}" "v2.0.11" ${OPENCL_CLHPP_VERSION_DESC})
		set(OPENCL_CLHPP_VERSION ${OPENCL_VERSION})
		# other available OpenCL-CLHPP versions see github tags; of note:
		#   - v2.0.11  last version to include cl.hpp, which open_iA still needed until recently
		#   - v2.0.9   first version listed on github; already contains CMakeLists.txt
		set(OPENCL_CLHPP_CMAKE_ARGS
			"-DCMAKE_INSTALL_PREFIX:PATH=${OpenCL_DIR}"
			"-DOpenCLHeaders_DIR:PATH=\"${OpenCLHeaders_DIR}\""
			"-DBUILD_DOCS:BOOL=OFF"
			"-DBUILD_EXAMPLES:BOOL=OFF"
			"-DBUILD_TESTING:BOOL=OFF"
		)
		ExternalProject_Add (OpenCL_CLHPP
			GIT_REPOSITORY    ${OPENCL_CLHPP_HEADERS_REPO}
			GIT_TAG           "${OPENCL_CLHPP_VERSION}"
			GIT_SUBMODULES    ""          # avoid pulling in CMock/Unity submodules only required for testing
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        ${OPENCL_CLHPP_CMAKE_ARGS}
			SOURCE_DIR        "${OPENCL_ROOT_DIR}/CLHPP/src.git"
			BINARY_DIR        "${OPENCL_ROOT_DIR}/CLHPP/bin"
			DEPENDS           OpenCL_Headers
		)
		set(OpenCLHeadersCpp_DIR "${OpenCL_DIR}/share/cmake/OpenCLHeadersCpp")

		# ICD loader library:
		set(OPENCL_ICD_REPO "https://github.com/KhronosGroup/OpenCL-ICD-Loader.git")
		set(OPENCL_ICD_CMAKE_ARGS
			"-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
			"-DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}"
			"-DCMAKE_INSTALL_PREFIX:PATH=${OpenCL_DIR}"
			"-DOpenCLHeaders_DIR:PATH=\"${OpenCLHeaders_DIR}\""
			"-DENABLE_OPENCL_LAYERINFO:BOOL=OFF"
			"-DOPENCL_ICD_LOADER_BUILD_SHARED_LIBS:BOOL=ON"
			"-DBUILD_TESTING:BOOL=OFF"
		)
		#if (DISABLE_LIB_WARNINGS)
		#	list(APPEND OPENCL_ICD_CMAKE_ARGS ${DISABLE_WARNINGS_OPT})
		#endif()
		if (NOT CMAKE_CONFIGURATION_TYPES)
			list(APPEND OPENCL_ICD_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
		endif()
		ExternalProject_Add (OpenCL_ICD
			GIT_REPOSITORY    ${OPENCL_ICD_REPO}
			GIT_TAG           "${OPENCL_VERSION}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			SOURCE_DIR        "${OPENCL_ROOT_DIR}/ICD-Loader/src.git"
			BINARY_DIR        "${OPENCL_ROOT_DIR}/ICD-Loader/bin"
			CMAKE_ARGS        ${OPENCL_ICD_CMAKE_ARGS}
			DEPENDS           OpenCL_CLHPP OpenCL_Headers
		)
		set(OpenCLICDLoader_DIR "${OpenCL_DIR}/share/cmake/OpenCLICDLoader")
		list(APPEND ITK_DEPENDS OpenCL_ICD)
	else()
		find_package(OpenCLHeaders REQUIRED)
		find_package(OpenCLICDLoader REQUIRED)
		find_package(OpenCLHeadersCpp REQUIRED)
		# assume these packages all are in share/cmake/OpenCL... subdirectory of install location:
		string(REPLACE "/share/cmake/OpenCLHeaders" "" OpenCL_DIR ${OpenCLHeaders_DIR})
		# not working for some reason:
		# get_target_property(OpenCLHeaders_INCLUDE_DIRS OpenCL::Headers INCLUDE_DIRECTORIES)
		# get_target_property(OpenCLICDLoader_LIBRARIES OpenCL::OpenCL LINK_LIBRARIES)
		# assume C and C++ headers are in same directory...
		# set(OpenCL_INCLUDE_DIRS ${OpenCLHeaders_INCLUDE_DIRS})
		# ...and assume that only ICD loader holds a library to link
		# set(OpenCL_LIBRARY ${OpenCLICDLoader_LIBRARIES})
	endif()
	if (DEFINED OpenCLICDLoader_DIR)
		# maybe use get_target_property somehow, as tried above (but not working)?
		# get rid of hard-coded references, this must be a property somewhere!
		set(OpenCL_INCLUDE_DIRS "${OpenCL_DIR}/include")
		set(OPENCL_SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX})
		if (WIN32)
			set(OPENCL_SUFFIX ".lib")
		endif()
		set(OpenCL_LIBRARY "${OpenCL_DIR}/${CMAKE_INSTALL_LIBDIR}/${CMAKE_SHARED_LIBRARY_PREFIX}OpenCL${OPENCL_SUFFIX}")
		message(STATUS "OpenCL: ${OpenCL_DIR}; include: ${OpenCL_INCLUDE_DIRS}; lib: ${OpenCL_LIBRARY}")
	endif()
endif()

set(openiA_DEPENDS "")
if (BUILD_ITK)
	set(ENABLE_ITK_GPU ${ENABLE_OPENCL})
	# on Apple, an ITK build with GPU/OpenCL fails with `undefined symbols: _clBuildProgram...`
	# see also https://github.com/InsightSoftwareConsortium/ITK/issues/3821
	# Since above issue mentions ITK_USE_GPU to be deprecated, simply disable it for now:
	if (APPLE)
		set(ENABLE_ITK_GPU OFF)
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
		"-DQt6_DIR:PATH=${Qt6_DIR}"
		"-DQt6WidgetsTools_DIR:PATH=${Qt6_DIR}WidgetsTools"
		"-DQt6CoreTools_DIR:PATH=${Qt6_DIR}CoreTools"
		"-DQt6GuiTools_DIR:PATH=${Qt6_DIR}GuiTools"
		"-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
		"-DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}"
	)
#	if (DISABLE_LIB_WARNINGS)
#		list(APPEND ITK_CMAKE_ARGS ${DISABLE_WARNINGS_OPT})
#	endif()
	if (NOT CMAKE_CONFIGURATION_TYPES)
		list(APPEND ITK_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
	endif()
	if (ENABLE_ITK_GPU)
		# the way to specify OpenCL includes libraries changed from ITK 5.2.x to 5.3.0:
		if (ITK_VERSION VERSION_GREATER_EQUAL "5.3.0")
			list(APPEND ITK_CMAKE_ARGS
				"-DOpenCL_INCLUDE_DIR:PATH=${OpenCL_INCLUDE_DIRS}"
				"-DOpenCL_LIBRARY:FILEPATH=${OpenCL_LIBRARY}")
		else()
			list(APPEND ITK_CMAKE_ARGS
				"-DOPENCL_ROOT_DIR:PATH=${OpenCL_DIR}"
				"-DOPENCL_INCLUDE_DIRS:PATH=${OpenCL_INCLUDE_DIRS}"
				"-DOPENCL_LIBRARIES=${OpenCL_LIBRARY}")
		endif()
	endif()
	if (ENABLE_VR)
		if (ENABLE_OPENVR)
			list(APPEND ITK_CMAKE_ARGS ${OPENVR_LIB_AND_INCLUDE})
		endif()
		if (ENABLE_OPENXR)
			list(APPEND ITK_CMAKE_ARGS ${VTK_OPENXR_LIB_AND_INCLUDE})
		endif()
	endif()
	if (ITK_USE_GIT_REPO)
		message(STATUS "ITK_USE_GIT_REPO is on, not applying any patches!")
		set(ITK_DIR "${CMAKE_BINARY_DIR}/itk/bin-${ITK_GIT_TAG}" CACHE PATH "ITK build folder." FORCE)
		ExternalProject_Add (ITK
			GIT_REPOSITORY    "${ITK_REPOSITORY}"
			GIT_TAG           "${ITK_GIT_TAG}"
			UPDATE_DISCONNECTED 1
			SOURCE_DIR        "${CMAKE_BINARY_DIR}/itk/src.git"
			BINARY_DIR        "${ITK_DIR}"
			CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
			CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
			CMAKE_ARGS        "${ITK_CMAKE_ARGS}"
			INSTALL_COMMAND   ""
			DEPENDS           ${ITK_DEPENDS})
	else()
		set(ITK_PATCH_COMMAND ${GIT_APPLY_CMD})
		if (ITK_VERSION VERSION_LESS "5.1.0")
			set(ITK_VTKGLUE_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-vtkGlue.patch")
			list(APPEND ITK_PATCH_COMMAND ${ITK_VTKGLUE_PATCH})
		endif()
		if (ITK_VERSION VERSION_LESS "5.2.0" AND MSVC_VERSION GREATER_EQUAL 1928)
			set(ITK_LRINTF_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-lrintf.patch")
			list(APPEND ITK_PATCH_COMMAND ${ITK_LRINTF_PATCH})
		endif()
		if (ITK_VERSION VERSION_EQUAL "5.3.0" AND MSVC)
			set(ITK_ITERATOR_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-5.3.0-iterator.patch")
			list(APPEND ITK_PATCH_COMMAND ${ITK_ITERATOR_PATCH})
			if (MSVC_VERSION LESS 1930)
				set (ITK_TRAILING_RETURN_TYPE_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-5.3.0-vs2019.patch")
				list(APPEND ITK_PATCH_COMMAND ${ITK_TRAILING_RETURN_TYPE_PATCH})
			endif()
		endif()
		if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL "11.0.0" AND ITK_VERSION VERSION_LESS "5.2.1")
			if (NOT ITK_VERSION VERSION_EQUAL "5.2.0")
				message(WARNING "You are using g++ version >= 11 with ITK < 5.2.0. Using g++ version >= 11 requires a patch for ITK 5.2.0, and most likely also for versions below, but the patch included in this superbuild is only tested with ITK version 5.2.0. For other ITK versions the patch might fail! If it fails, your options are to choose a newer ITK version (>= 5.2.0), use a different/older compiler, or fix any compilation errors that you get yourself!")
			endif()
			set(ITK_GCC11_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-g++11.patch")
			list(APPEND ITK_PATCH_COMMAND ${ITK_GCC11_PATCH})
		endif()
		if (ITK_VERSION VERSION_GREATER_EQUAL "5.2.0" AND ITK_VERSION VERSION_LESS "5.3.0")
			# patch should be included in >= 5.3.0, see https://github.com/Kitware/MetaIO/pull/110
			set(ITK_GREATER2GB_PATCH "${CMAKE_SOURCE_DIR}/patches/itk-5.2-metaIO-decompressgreater2GBfix.patch")
			list(APPEND ITK_PATCH_COMMAND ${ITK_GREATER2GB_PATCH})
		endif()
		if (ITK_PATCH_COMMAND STREQUAL "${GIT_APPLY_CMD}")
			set(ITK_PATCH_COMMAND "")	# if no patch to apply, set empty
		endif()
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
if ((ENABLE_ASTRA AND BUILD_ASTRA) OR (ENABLE_AI AND AI_ONNX_USE_CUDA))
	set(CUDA_FIND_ENABLED ON)
endif()
if (CUDA_FIND_ENABLED)
	FIND_PACKAGE(CUDAToolkit)
	if (CUDAToolkit_FOUND)
		set(CUDAToolkit_VERSION_UNDERSCORE "${CUDAToolkit_VERSION_MAJOR}_${CUDAToolkit_VERSION_MINOR}")
		set(CUDAToolkit_VERSION_SHORT "${CUDAToolkit_VERSION_MAJOR}.${CUDAToolkit_VERSION_MINOR}")
		if (NOT DEFINED CUDAToolkit_ROOT_DIR)
			set(CUDAToolkit_ROOT_DIR "${CUDAToolkit_TARGET_DIR}")
		endif()
		message(STATUS "CUDA: ${CUDAToolkit_VERSION} in ${CUDAToolkit_ROOT_DIR}.")
	else()
		message(SEND_ERROR "CUDA, required for ASTRA / AI, not found! Specify where to find CUDAToolkit, or uncheck ENABLE_ASTRA / ENABLE_AI!")
	endif()
endif()

if (BUILD_BOOST)
	if (DEFINED CUDAToolkit_VERSION AND CUDAToolkit_VERSION VERSION_LESS "11.4")
		message(SEND_ERROR "The combination of boost >= 1.79 with CUDA < 11.4 can cause build problems (on Visual Studio, there is an error : identifier \"__builtin_FILE\" is undefined)! Please use a CUDA version >= 11.4!")
	endif()
	# fetch and build boost...
	# inspired by https://github.com/apache/geode-native/blob/develop/dependencies/boost/CMakeLists.txt
	string(REPLACE "." "_" BOOST_VERSION_UNDERSCORE ${BOOST_VERSION})
	set(BOOST_URL "https://boostorg.jfrog.io/artifactory/main/release/${BOOST_VERSION}/source/boost_${BOOST_VERSION_UNDERSCORE}.zip")
	set(BOOST_BOOTSTRAP_OPTS "")
	if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
		set(BOOST_BOOTSTRAP_OPTS "--with-toolset=clang")
	endif()
	set(B2_FLAGS -j8 address-model=64 architecture=x86 link=static threading=multi runtime-link=shared --with-chrono --with-date_time --with-system --with-thread stage)
	if (WIN32)
		set(BOOST_BOOTSTRAP_COMMAND ${CMAKE_BINARY_DIR}\\boost\\bootstrap.bat)
		set(BOOST_BOOTSTRAP_OPTS ${BOOST_LIB_SUFFIX})    # in case of multiple VS installations, make sure the one selected as generator is used!
		set(B2_COMMAND ${CMAKE_BINARY_DIR}\\boost\\b2)
		set(B2_FLAGS ${B2_FLAGS} toolset=${BOOST_MSVC_TOOLSET})
	else()
		set(BOOST_BOOTSTRAP_COMMAND ${CMAKE_BINARY_DIR}/boost/bootstrap.sh)
		set(B2_COMMAND ${CMAKE_BINARY_DIR}/boost/b2)
		set(B2_FLAGS ${B2_FLAGS})
	endif()
	if (BOOST_VERSION VERSION_EQUAL "1.85.0")
		set (BOOST_PATCH_COMMAND ${GIT_APPLY_CMD} "${CMAKE_SOURCE_DIR}/patches/boost-1.85-vs17.10.patch")
	else()
		set (BOOST_PATCH_COMMAND "")
	endif()
	set(BOOST_DIR "${CMAKE_BINARY_DIR}/boost")
	ExternalProject_Add (boost
		URL               ${BOOST_URL}
		URL_HASH          SHA512=${BOOST_ARCHIVE_SHA512}
		DOWNLOAD_DIR      ${ARCHIVE_DIR}
		BUILD_IN_SOURCE   1
		SOURCE_DIR        "${BOOST_DIR}"
		PATCH_COMMAND     ${BOOST_PATCH_COMMAND}
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
		if (MSVC)
			option(ASTRA_PATCH "Whether to patch Astra Toolbox project files. Only required to be set first time sources are fetched, disable later." ON)
			mark_as_advanced(ASTRA_PATCH)
		endif()

		if (BOOST_VERSION VERSION_GREATER_EQUAL "1.85.0" AND CUDAToolkit_VERSION VERSION_LESS "11.8.0")
			message(WARNING "Note that with boost 1.85 and CUDA toolkit probably < 11.8 (encountered with 11.7, but not with 11.8), the Astra build might report errors like `error : identifier \"__builtin_FUNCSIG\" is undefined` during build! Choose an older version (< 1.85) of boost here, or install a newer version of CUDA (tested to work with 12.4).")
		endif()
		if (CUDAToolkit_VERSION VERSION_GREATER_EQUAL "12.0.0" AND NOT ("${ASTRA_TAG}" STREQUAL "master"))
			message(WARNING "NOTE: You have selected a CUDA version >= 12, and a tagged Astra version; these are not compatible - there is no tagged Astra version yet which is compatible with CUDA >= 12; so this configuration will lead to build errors. Please either select a lower CUDA version, or choose ASTRA_TAG 'master'!")
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
		if (MSVC)
			CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/patches/astra-${ASTRA_TAG}-vs.patch.template" "${CMAKE_BINARY_DIR}/astra-vs.patch")
			# find way to only do this once:
			if (ASTRA_PATCH)
				list(APPEND ASTRA_PATCH_COMMAND "${CMAKE_BINARY_DIR}/astra-vs.patch")
			endif()
			if (ASTRA_PATCH AND DISABLE_LIB_WARNINGS)
				list(APPEND ASTRA_PATCH_COMMAND "${CMAKE_SOURCE_DIR}/patches/astra-vs-warnings.patch")
			endif()
			if (ASTRA_PATCH AND CUDAToolkit_VERSION VERSION_GREATER_EQUAL "11.0")
				list(APPEND ASTRA_PATCH_COMMAND "${CMAKE_SOURCE_DIR}/patches/astra-${ASTRA_TAG}-vs-remove_30.patch")
			endif()
			set(ASTRA_BUILD_COMMAND "${CMAKE_MAKE_PROGRAM}" "${ASTRA_DIR}/astra_vc14.sln" -t:astra_vc14 "-p:Configuration=$<IF:$<CONFIG:Debug>,Debug,Release>_CUDA" -p:Platform=x64 -m)
		else()
			set(ASTRA_BUILD_DIR "${ASTRA_DIR}/build/linux")
			set(ASTRA_CONFIGURE_COMMAND1 "${CMAKE_COMMAND}" -E chdir "${ASTRA_BUILD_DIR}" "${ASTRA_BUILD_DIR}/autogen.sh")
			set(ASTRA_CONFIGURE_COMMAND2 "${CMAKE_COMMAND}" -E chdir "${ASTRA_BUILD_DIR}" "${ASTRA_BUILD_DIR}/configure" "--with-cuda=${CUDAToolkit_ROOT_DIR}")
			set(ASTRA_BUILD_COMMAND "${CMAKE_COMMAND}" -E chdir "${ASTRA_BUILD_DIR}" "make" -j 8)
		endif()

		if (UNIX)
			message(STATUS "Note: Building Astra Toolbox is enabled, and you're building under *nix.")
			message(STATUS "	Make sure to install autoconf/automake/libtool, e.g. on Ubuntu via `sudo apt install automake libtool`")
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
	set(HDF5_FULL_NAME "hdf5-${HDF5_VERSION}")
	set(HDF5_DIR "${CMAKE_BINARY_DIR}/${HDF5_FULL_NAME}")
	set(HDF5_SRC_DIR "${HDF5_DIR}/src")
	set(HDF5_BIN_DIR "${HDF5_DIR}/bin")
	set(HDF5_INSTALL_DIR "${HDF5_DIR}/install")
	set(HDF5_COMPRESSION_LIB_DIR "${HDF5_DIR}")
	set(ZLIB_VERSION "1.3.1")
	set(HDF5_ZLIB_URL "https://github.com/madler/zlib/releases/download/v${ZLIB_VERSION}/zlib-${ZLIB_VERSION}.tar.gz")
	set(HDF5_ZLIB_ARCHIVE_HASH "580677AAD97093829090D4B605AC81C50327E74A6C2DE0B85DD2E8525553F3DDDE17556EA46F8F007F89E435493C9A20BC997D1EF1C1C2C23274528E3C46B94F")
	ExternalProject_Add(hdf5_zlib
		URL               ${HDF5_ZLIB_URL}
		URL_HASH          "SHA512=${HDF5_ZLIB_ARCHIVE_HASH}"
		DOWNLOAD_NO_EXTRACT
		DOWNLOAD_DIR      ${HDF5_COMPRESSION_LIB_DIR}
		CONFIGURE_COMMAND ""
		BUILD_COMMAND     ""
		INSTALL_COMMAND   ""
	)
	set(HDF5_CMAKE_ARGS "-C ${HDF5_SRC_DIR}/config/cmake/cacheinit.cmake" "--log-level=VERBOSE" "-DHDF5_ALLOW_EXTERNAL_SUPPORT:STRING=TGZ"  "-DBUILD_SHARED_LIBS:BOOL=OFF" "-DHDF5_BUILD_TOOLS:BOOL=OFF" "-DHDF5_BUILD_UTILS:BOOL=OFF" "-DBUILD_TESTING:BOOL=OFF" "-DHDF5_BUILD_EXAMPLES:BOOL=OFF" "-DHDF5_BUILD_JAVA:BOOL=OFF" "-DHDF5_BUILD_FORTRAN:BOOL=OFF" "-DHDF5_BUILD_CPP_LIB:BOOL=OFF" "-DHDF5_ENABLE_Z_LIB_SUPPORT:BOOL=ON" "-DZLIB_TGZ_NAME:STRING=zlib-${ZLIB_VERSION}.tar.gz" "-DZLIB_USE_LOCALCONTENT:BOOL=ON" "-DZLIB_USE_EXTERNAL:BOOL=ON" "-DHDF5_ENABLE_SZIP_SUPPORT:BOOL=OFF" "-DHDF5_PACKAGE_EXTLIBS:BOOL=ON" "-DTGZPATH:PATH=${HDF5_COMPRESSION_LIB_DIR}" "-DCMAKE_INSTALL_PREFIX:PATH=${HDF5_INSTALL_DIR}")
	ExternalProject_Add (hdf5
		URL               ${HDF5_URL}
		URL_HASH          "SHA512=${HDF5_ARCHIVE_SHA512}"
		DOWNLOAD_DIR      ${ARCHIVE_DIR}
		SOURCE_DIR        "${HDF5_SRC_DIR}"
		BINARY_DIR        "${HDF5_BIN_DIR}"
		CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
		CMAKE_GENERATOR_PLATFORM "${CMAKE_GENERATOR_PLATFORM}"
		CMAKE_ARGS        "${HDF5_CMAKE_ARGS}"
		DEPENDS           hdf5_zlib)
endif()

if (ENABLE_AI)
	# CUDA or DirectML versions of onnx runtime are available
	# (there is a CPU version as well, but open_iA currently does not support it)
	set(ONNX_VERSION_OPTIONS "1.13.1" "1.14.1" "1.16.3" "1.17.1" "1.17.3" "1.18.0")
	selector(ONNX_VERSION "${ONNX_VERSION_OPTIONS}" "1.16.3" "Choose the onnx runtime version to use.")
	set(ONNX_SRC_DIR "${CMAKE_BINARY_DIR}/onnx-${ONNX_VERSION}")
	set(ONNX_INCLUDE_SUBDIR "include")
	if (WIN32)
		option(AI_ONNX_USE_CUDA "Whether to use CUDA for running ONNX. Note that CUDA support is currently BROKEN in open_iA. If disabled (default), DirectML will be used (on Windows)." OFF)
		mark_as_advanced(AI_ONNX_USE_CUDA)
		if (AI_ONNX_USE_CUDA)
			if (NOT CUDAToolkit_FOUND)	# CUDA might have been found for Astra already
				FIND_PACKAGE(CUDAToolkit)
			endif()
			if (NOT CUDAToolkit_FOUND)
				message(SEND_ERROR "CUDA, required for ONNX, not found! Specify where to find CUDAToolkit, or uncheck ENABLE_AI (or AI_ONNX_USE_CUDA, if you want to use DirectML)!")
			endif()
			if ("${ONNX_VERSION}" STREQUAL "1.13.1")
				set(ONNX_HASH "0D0AB564E7F991582750E988C9BC746BA151F866DACDA742BA50621685AAA258828806CFEA2645320B4CD526C21D91D4EBE7C7FA12FB5435C491CBCCFAFFE147")
			else()
				message(SEND_ERROR "Invalid ONNX version!")
			endif()
			set(ONNX_ARCHIVE "https://github.com/microsoft/onnxruntime/releases/download/v${ONNX_VERSION}/onnxruntime-win-x64-gpu-${ONNX_VERSION}.zip")
			set(ONNX_LIB_DIR "${ONNX_SRC_DIR}/lib")
		else () # for DirectML:
			if ("${ONNX_VERSION}" STREQUAL "1.13.1")
				set(ONNX_HASH "C2D85850E143F5CF0057FB85A4FE23937FF37C4BB5060E4C9783417B08033F29B576860FD1691149F6175E09CB3B65C612067C3896C9B741ABDD6711C9198AC6")
			elseif ("${ONNX_VERSION}" STREQUAL "1.14.1")
				set(ONNX_HASH "04693D1524E628C8EBDEEF7272725964AD78E90DF821C88361BF6C1F37DF718040B80120D3C167A36E3BDEDF5A82D8A6035EC9379BE7E3D35CA67F3C4CCB4388")
			elseif ("${ONNX_VERSION}" STREQUAL "1.16.3")
				set(ONNX_HASH "F522CBF76C189C0BEE7869081DBDBF724EFFBF33DD0EBFDD7C077709EC08BAE904BA182EFD47FF40598C8BD287B64BAECDD8BB84E3109E7F4B7649FF8490DDF6")
			elseif ("${ONNX_VERSION}" STREQUAL "1.17.1")
				set(ONNX_HASH "BB58FC1B308BFED59D5481B51EB4E201E037AA45BE4F4200A5436B88C3CC53C3345313F7B475274A6E794C61393B4D8C084CBA55EDF05DD22C07300C6E1386F2")
			elseif ("${ONNX_VERSION}" STREQUAL "1.17.3")
				set(ONNX_HASH "3F1D75BCA1BD42E1AEA2B5BE8AEBC00DF8F9A3881957D385DB4A53CA4AE511BEFB8A305CDC4664A67596CA8AED48BA1EC58BD2341DACEBC83AB19BD94DCA8AFE")
			elseif ("${ONNX_VERSION}" STREQUAL "1.18.0")
				set(ONNX_HASH "38467A89861DE8D1CE0DAFC10B25683491DF220ABB37AEAF6EF80BDED59C450E445AE5997C5DBE853DBDF9A20D468145A5521F3C1712A669C9F7717A87A45492")
			else()
				message(SEND_ERROR "Invalid ONNX version!")
			endif()
			set(ONNX_ARCHIVE "https://github.com/microsoft/onnxruntime/releases/download/v${ONNX_VERSION}/Microsoft.ML.OnnxRuntime.DirectML.${ONNX_VERSION}.zip")
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
		if ("${ONNX_VERSION}" STREQUAL "1.13.1")
			set(ONNX_HASH "9E161B423CC85BB0D10F124D78C7999409B5CB63AB0E8FE97C250A45A91F7D1EDB9536D5E951DA72F57CF07DFBE994CC6F1D23AD72A8F36A69DCADF12A3EA294")
		else()
			message(SEND_ERROR "Invalid ONNX version!")
		endif()
		set(ONNX_LIBRARIES "${ONNX_SRC_DIR}/lib/libonnxruntime.so.${ONNX_VERSION}")
	elseif (APPLE)
		message(SEND_ERROR "No supported ONNX runtime for OS X - there is only a CPU-based library, which is not supported by open_iA. Please disable ENABLE_AI option!")
		#set(ONNX_ARCHIVE "https://github.com/microsoft/onnxruntime/releases/download/v${ONNX_VERSION}/onnxruntime-osx-x64-${ONNX_VERSION}.tgz")
		#set(ONNX_HASH "91EC06DA8EA381ED6FA9089D6570A8DE9C38178323B4FF6AB0BF014A4A314FDECD617DB32864364F81DC5A9665FC66A060E17349FC058686CD451DA35943909D")
		#set(ONNX_LIBRARIES "${ONNX_SRC_DIR}/lib/libonnxruntime.${ONNX_VERSION}.dylib")
	else()
		message(SEND_ERROR "Unknown operating system, no supported ONNX runtime available. Please disable ENABLE_AI option!")
	endif()

	if (AI_ONNX_USE_CUDA)
		message(WARNING "ONNX CUDA support in open_iA is currently BROKEN! Please use DirectML on Windows (by disabling AI_ONNX_USE_CUDA).")
		if ("${ONNX_VERSION}" STREQUAL "1.13.1")
			set(ONNX_REQUIRED_CUDA_MAJOR_VERSION "11")
			if (WIN32)
				set(ONNX_REQUIRED_CUDNN_VERSION "8.2.2.26")
			else()
				set(ONNX_REQUIRED_CUDNN_VERSION "8.2.4")
			endif()
		else()
			set(ONNX_REQUIRED_CUDA_MAJOR_VERSION "unknown")
		endif()
		if (NOT "${CUDAToolkit_VERSION_MAJOR}" STREQUAL "${ONNX_REQUIRED_CUDA_MAJOR_VERSION}")
			message(WARNING "Incompatible CUDA version (${CUDAToolkit_VERSION}) for ONNX - the chosen CUDA variant of the ONNX runtime version ${ONNX_VERSION} requires CUDA version ${ONNX_REQUIRED_CUDA_MAJOR_VERSION}.x! Please either install a compatible CUDA version, choose a different ONNX_VERSION, or disable AI_ONNX_USE_CUDA to use DirectML runtime instead (only available on Windows)! See also https://onnxruntime.ai/docs/reference/execution-providers/CUDA-ExecutionProvider.html#requirements")
		endif()
		message(WARNING "onnx CUDA runtime v${ONNX_VERSION} requires cuDNN version ${ONNX_REQUIRED_CUDNN_VERSION} (I cannot check for cuDNN at the moment; so you need to make sure yourself you have installed it)!")
	endif()

	set(ONNX_INCLUDE_DIR "${ONNX_SRC_DIR}/${ONNX_INCLUDE_SUBDIR}")
	ExternalProject_Add (onnx
		URL               ${ONNX_ARCHIVE}
		URL_HASH          "SHA512=${ONNX_HASH}"
		DOWNLOAD_DIR      ${ARCHIVE_DIR}
		SOURCE_DIR        "${ONNX_SRC_DIR}"
		BUILD_IN_SOURCE   1
		CONFIGURE_COMMAND ""
		BUILD_COMMAND     ""
		INSTALL_COMMAND   "")
endif()

set(OPEN_IA_CMAKE_ARGS
	"-DQt6_DIR:PATH=${Qt6_DIR}"
	"-DQt6WidgetsTools_DIR:PATH=${Qt6_DIR}WidgetsTools"
	"-DQt6CoreTools_DIR:PATH=${Qt6_DIR}CoreTools"
	"-DQt6GuiTools_DIR:PATH=${Qt6_DIR}GuiTools"
	"-DITK_DIR:PATH=${ITK_DIR}"
	"-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
	"-DopeniA_AVX_SUPPORT:STRING=${AVX_OPTIMIZATION}"
)
if (NOT CMAKE_CONFIGURATION_TYPES)
	list(APPEND OPEN_IA_CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
endif()
if (ENABLE_OPENCL)
	list(APPEND OPEN_IA_CMAKE_ARGS
	"-DOpenCLHeaders_DIR:PATH=${OpenCLHeaders_DIR}"
	"-DOpenCLHeadersCpp_DIR:PATH=${OpenCLHeadersCpp_DIR}"
	"-DOpenCLICDLoader_DIR:PATH=${OpenCLICDLoader_DIR}"
	)
endif()
if (ENABLE_FILTERS)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_CommonImageFilters:BOOL=ON"
		"-DModule_Surfaces:BOOL=ON"
		"-DModule_FeatureCharacteristics:BOOL=ON"
		"-DModule_MetaFilters:BOOL=ON"
		"-DModule_Metrics:BOOL=ON"
		"-DModule_Segmentation:BOOL=ON")
endif()
if (ENABLE_TOOLS)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_4DCT:BOOL=ON"
		"-DModule_BoneThickness:BOOL=ON"
		"-DModule_CompVis:BOOL=ON"
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
		"-DModule_Labelling:BOOL=ON"
		"-DModule_NModalTF:BOOL=ON"
		"-DModule_Remote:BOOL=ON"
		"-DModule_TripleHistogramTF:BOOL=ON"
		"-DModule_Uncertainty:BOOL=ON"
		"-DModule_VolumePlayer:BOOL=ON")
	# Qt 6.0 does not come with Qt Charts; any Qt 5 version ships it though, and Qt >= 6.1 does too:
	if (QT_VERSION_MAJOR LESS 6 OR QT_VERSION VERSION_GREATER_EQUAL "6.1.0" AND Qt${QT_VERSION_MAJOR}Charts_FOUND)
		list(APPEND OPEN_IA_CMAKE_ARGS
			"-DModule_AdaptiveThreshold:BOOL=ON"
			"-DQt${QT_VERSION_MAJOR}Charts_DIR:PATH=${Qt${QT_VERSION_MAJOR}_DIR}Charts")
	else()
		message(WARNING "We did not find the Qt Charts component! Either you are using Qt 6.0.x, which does not supply that component, or you did not select the component during Qt installation. The AdaptiveThresholding module depending on that component is therefore disabled!")
	endif()
	if (NOT ENABLE_FILTERS)		# if filters are not enabled,
		list(APPEND OPEN_IA_CMAKE_ARGS	# we need to enable at least those filters required by modules:
			"-DModule_MetaFilters:BOOL=ON"				# required by GEMSe
			"-DModule_Segmentation:BOOL=ON")			# required by AdaptiveThreshold, FiAKEr
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
		"-DASTRA_TOOLBOX_DIR:PATH=${ASTRA_DIR}"
		"-DCUDAToolkit_ROOT:PATH=${CUDAToolkit_ROOT_DIR}")
	if (BUILD_ASTRA)
		list(APPEND openiA_DEPENDS astra)
	endif()
endif()
if (ENABLE_VR)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_ImNDT:BOOL=ON"
		"-DModule_XVRA:BOOL=ON")
	if (ENABLE_OPENVR)
		list(APPEND OPEN_IA_CMAKE_ARGS ${OPENVR_LIB_AND_INCLUDE})
	endif()
	if (ENABLE_OPENXR)
		list(APPEND OPEN_IA_CMAKE_ARGS ${OPENXR_LIB_AND_INCLUDE} ${OPENXR_DIR})
	endif()
	if (BUILD_BOOST)
		list(APPEND openiA_DEPENDS boost)
	endif()
	if (NOT ENABLE_ASTRA)	# if astra module is enabled, it will care for setting up boost include directory; otherwise, we need to set it manually:
		if (NOT BUILD_BOOST)
			find_package(Boost)
			set(BOOST_INCLUDE_DIR ${Boost_INCLUDE_DIR})
		endif ()
		list(APPEND OPEN_IA_CMAKE_ARGS "-DBOOST_INCLUDE_DIR:PATH=${BOOST_INCLUDE_DIR}")
	endif()
endif()
if (ENABLE_AI)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DModule_AI:BOOL=ON"
		"-DONNX_RUNTIME_DIR:PATH=${ONNX_SRC_DIR}"
		"-DONNX_RUNTIME_INCLUDE_DIR:PATH=${ONNX_INCLUDE_DIR}"
		"-DONNX_RUNTIME_LIBRARIES:FILEPATH=${ONNX_LIBRARIES}"
		"-DopeniA_ONNX_CUDA:BOOL=${AI_ONNX_USE_CUDA}")
	list(APPEND openiA_DEPENDS onnx)
endif()
if (ENABLE_EIGEN)
	if (EIGEN_USE_GIT_REPO)
		set(EIGEN_GIT_TAG "master")
		set(EIGEN_BIN_DIR "${CMAKE_BINARY_DIR}/eigen-${EIGEN_GIT_TAG}")
		ExternalProject_Add (eigen
			GIT_REPOSITORY    "${EIGEN_URL}"
			GIT_TAG           "${EIGEN_GIT_TAG}"
			SOURCE_DIR        "${EIGEN_BIN_DIR}"
			BUILD_IN_SOURCE   1
			CONFIGURE_COMMAND ""
			BUILD_COMMAND     ""
			INSTALL_COMMAND   "")
	else()
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
	endif()
	list(APPEND openiA_DEPENDS eigen)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DEIGEN3_INCLUDE_DIR:PATH=${EIGEN_BIN_DIR}")
endif()
if (ENABLE_HDF5)
	list(APPEND openiA_DEPENDS hdf5)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DHDF5_DIR:PATH=${HDF5_INSTALL_DIR}/cmake")
endif()
if (ENABLE_VTK_AVIWRITER)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DVTK_USE_AVIWRITER:BOOL=ON")
endif()
list(APPEND OPEN_IA_CMAKE_ARGS
	"-DopeniA_CUDA_ENABLED:BOOL=${CUDA_FIND_ENABLED}")
if (CUDA_FIND_ENABLED)
	list(APPEND OPEN_IA_CMAKE_ARGS
		"-DCUDAToolkit_NVCC_EXECUTABLE:FILEPATH=${CUDAToolkit_NVCC_EXECUTABLE}")
endif()

set(OPEN_IA_GIT_TAG_OPTIONS "develop" "main" "2023.04" "2023.02" "2022.06")
selector(OPEN_IA_GIT_TAG "${OPEN_IA_GIT_TAG_OPTIONS}" "develop" "Choose the open_iA version to build.")
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
