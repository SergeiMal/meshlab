# Copyright 2019, 2020, Collabora, Ltd.
# Copyright 2019, 2021, Visual Computing Lab, ISTI - Italian National Research Council
# SPDX-License-Identifier: BSL-1.0

cmake_minimum_required(VERSION 3.10)

cmake_policy(SET CMP0074 NEW)
set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} /bigobj)

if(WIN32)
  message(STATUS "WIN32 detected.!")
endif()

project(MeshLab)

### Build options
option(BUILD_MINI "Build only a minimal set of plugins" ON)
option(BUILD_STRICT "Strictly enforce resolution of all symbols" ON)
option(BUILD_WITH_DOUBLE_SCALAR "Use double type instead of float type for scalars" OFF)

option(BUILD_ONLY_MESHLAB_LIBRARIES "Build only meshlab-common and plugins" ON)
option(USE_DEFAULT_BUILD_AND_INSTALL_DIRS "If set to OFF, it expects that you set manually the binary and install directories" ON)

option(MESHLAB_IS_NIGHTLY_VERSION "Nightly version of meshlab will be used instead of ML_VERSION" OFF)


IF (POLICY CMP0053)
    CMAKE_POLICY(SET CMP0053 NEW) # CMake 3.1 introduced a much faster implementation of evaluation of the Variable References and Escape Sequences
ENDIF ()
IF (POLICY CMP0068)
    CMAKE_POLICY(SET CMP0068 NEW) # In CMake 3.9, it only affects setting of RPATH. However, if one wants INSTALL_NAME_DIR to apply to the target in the build tree, one may set BUILD_WITH_INSTALL_NAME_DIR.
ENDIF ()
IF (POLICY CMP0087)
    CMAKE_POLICY(SET CMP0087 NEW) # In CMake 3.13 and earlier, install(CODE) and install(SCRIPT) did not evaluate generator expressions. CMake 3.14 and later will evaluate generator expressions for install(CODE) and install(SCRIPT).
ENDIF ()

### Dependencies
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/meshlab_global_settings.cmake")
include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/meshlab_tools.cmake")

#message(STATUS "Searching for required components")
find_package(OpenGL REQUIRED)

find_package(
	Qt5
	COMPONENTS OpenGL Xml Network
	REQUIRED)
find_package(OpenMP)

message(STATUS "Searching for required components with bundled fallback")
find_package(GLEW)
find_package(Eigen3)

#IF (NOT SET_UP_CONFIGURATIONS_DONE)
    SET(SET_UP_CONFIGURATIONS_DONE 1)

    MESSAGE("Begin setup config: SET_UP_CONFIGURATIONS_DONE.")

    # No reason to set CMAKE_CONFIGURATION_TYPES if it's not a multiconfig generator
    # Also no reason mess with CMAKE_BUILD_TYPE if it's a multiconfig generator.
    IF (CMAKE_CONFIGURATION_TYPES) # multiconfig generator?
        SET(CMAKE_CONFIGURATION_TYPES "Debug;RelWithDebInfo;Release" CACHE STRING "" FORCE)
        SET(CMAKE_CONFIGURATION_TYPES "Debug;RelWithDebInfo;Release" CACHE STRING "" FORCE)
    ELSE ()
        IF (NOT CMAKE_BUILD_TYPE)
            MESSAGE("Defaulting to RelWithDebInfo build.")
            #SET(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "" FORCE)
            SET(CMAKE_BUILD_TYPE Release CACHE STRING "" FORCE)
        ENDIF ()
        SET_PROPERTY(CACHE CMAKE_BUILD_TYPE PROPERTY HELPSTRING "Choose the type of build")
        # set the valid options for cmake-gui drop-down list
        SET_PROPERTY(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug;RelWithDebInfo;Release")
    ENDIF ()
    # now set up the Profile configuration
    SET(CMAKE_C_FLAGS_PROFILE "...")
    SET(CMAKE_CXX_FLAGS_PROFILE "...")
    SET(CMAKE_EXE_LINKER_FLAGS_PROFILE "...")
#ENDIF ()

## Allow Debug and RelWithDebInfo only
#IF (CMAKE_CONFIGURATION_TYPES)
    SET(CMAKE_CONFIGURATION_TYPES "Debug;RelWithDebInfo;Release" CACHE STRING "My multi config types" FORCE)
#ENDIF ()

#IF (CMAKE_CONFIGURATION_TYPES)
    SET(MeshLab_BIN_DIRECTORY $<$<CONFIG:RelWithDebInfo>:${CMAKE_CURRENT_BINARY_DIR}/bin/RelWithDebInfo>$<$<CONFIG:Release>:${CMAKE_CURRENT_BINARY_DIR}/bin/Release>$<$<CONFIG:Debug>:${CMAKE_CURRENT_BINARY_DIR}/bin/Debug>)
#ELSE (CMAKE_CONFIGURATION_TYPES)
#    SET(MeshLab_BIN_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
#ENDIF (CMAKE_CONFIGURATION_TYPES)

# Definition of preprocessor macros
add_definitions(-DWEDGNORMAL_MODE)

message( "MeshLab_BIN_DIRECTORY = ${MeshLab_BIN_DIRECTORY}" )
message( "CMAKE_CURRENT_BINARY_DIR = ${CMAKE_CURRENT_BINARY_DIR}")

### Build directories
if (USE_DEFAULT_BUILD_AND_INSTALL_DIRS) # otherwise, we assume that all these dirs are defined in parent dir
	set(MESHLAB_BUILD_DISTRIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/distrib)
	set(MESHLAB_LIB_OUTPUT_DIR ${MESHLAB_BUILD_DISTRIB_DIR})
	set(MESHLAB_PLUGIN_OUTPUT_DIR ${MESHLAB_BUILD_DISTRIB_DIR}/plugins)
	set(MESHLAB_SHADER_OUTPUT_DIR ${MESHLAB_BUILD_DISTRIB_DIR}/shaders)
	set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${MESHLAB_BUILD_DISTRIB_DIR})

	### Install directories
	if(WIN32 OR APPLE)
		set(INSTALL_TO_UNIX_LAYOUT OFF)
	else()
		set(INSTALL_TO_UNIX_LAYOUT ON)
	endif()
	include(GNUInstallDirs)
	if(INSTALL_TO_UNIX_LAYOUT)
		set(MESHLAB_BIN_INSTALL_DIR ${CMAKE_INSTALL_BINDIR})
		set(MESHLAB_LIB_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR}/meshlab)
		set(MESHLAB_PLUGIN_INSTALL_DIR ${MESHLAB_LIB_INSTALL_DIR}/plugins)
		set(MESHLAB_SHADER_INSTALL_DIR ${CMAKE_INSTALL_DATAROOTDIR}/meshlab/shaders)
	elseif(APPLE)
		set(MESHLAB_BIN_INSTALL_DIR .)
		set(MESHLAB_LIB_INSTALL_DIR meshlab.app/Contents/Frameworks)
		set(MESHLAB_PLUGIN_INSTALL_DIR meshlab.app/Contents/PlugIns)
		set(MESHLAB_SHADER_INSTALL_DIR meshlab.app/Contents/shaders)
	else()
		set(MESHLAB_BIN_INSTALL_DIR .)
		set(MESHLAB_LIB_INSTALL_DIR .)
		set(MESHLAB_PLUGIN_INSTALL_DIR plugins)
		set(MESHLAB_SHADER_INSTALL_DIR shaders)
	endif()

	### Install Settings
	if (NOT APPLE)
		set(CMAKE_INSTALL_RPATH $ORIGIN/../${MESHLAB_LIB_INSTALL_DIR};$ORIGIN/../${CMAKE_INSTALL_LIBDIR})
	else()
		SET(CMAKE_INSTALL_RPATH $ORIGIN/../Frameworks)
	endif()

	get_directory_property(hasParent PARENT_DIRECTORY)
	if(hasParent)
		set(MESHLAB_PLUGIN_OUTPUT_DIR ${MESHLAB_PLUGIN_OUTPUT_DIR} PARENT_SCOPE)
		set(MESHLAB_PLUGIN_INSTALL_DIR ${MESHLAB_PLUGIN_INSTALL_DIR} PARENT_SCOPE)
	endif()
endif()

### Enter subdirectories

# VCGLib -- required
if (VCGDIR) # VCGDIR exists - using custom user vcglib path
	if(EXISTS ${VCGDIR})
		add_subdirectory(${VCGDIR} {CMAKE_CURRENT_BINARY_DIR}/vcglib)
		message(STATUS "- VCGLib - using custom VCGDIR path library")
	else()
		set(VCGDIR NOTFOUND)
	endif()
else()
	get_filename_component(VCGDIR "${CMAKE_CURRENT_LIST_DIR}/vcglib" ABSOLUTE)
	if(EXISTS ${VCGDIR})
		add_subdirectory(${VCGDIR})
		message(STATUS "- VCGLib - using bundled source")
	else()
		set(VCGDIR NOTFOUND)
	endif()
endif()
set(VCGDIR "${VCGDIR}")

if(NOT VCGDIR)
	message(FATAL_ERROR "VCGLib not found. Please clone recursively the MeshLab repo.")
endif()

# External
set(EXTERNAL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external)
add_subdirectory(${EXTERNAL_DIR})

add_subdirectory(common)

if (NOT BUILD_ONLY_MESHLAB_LIBRARIES)
	add_subdirectory(meshlab)
	if(WIN32 AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/use_cpu_opengl")
		add_subdirectory(use_cpu_opengl)
	endif()

	# Clear RPATH before adding plugins
	set(CMAKE_INSTALL_RPATH)
endif()



### Plugin subdirectories
if(NOT DEFINED MESHLAB_PLUGINS) # it may be already defined in parent directory
	if(BUILD_MINI)
		set(MESHLAB_PLUGINS
			meshlabplugins/io_base
			meshlabplugins/filter_meshing
#			meshlabplugins/decorate_base
			meshlabplugins/filter_measure
		)
	else()
		set(MESHLAB_PLUGINS
			# IO plugins
			meshlabplugins/io_3ds
			meshlabplugins/io_base
			meshlabplugins/io_bre
			meshlabplugins/io_collada
			meshlabplugins/io_ctm
			meshlabplugins/io_e57
			meshlabplugins/io_expe
			meshlabplugins/io_gltf
			meshlabplugins/io_json
			meshlabplugins/io_nxs
			meshlabplugins/io_pdb
			meshlabplugins/io_tri
			meshlabplugins/io_txt
			meshlabplugins/io_u3d
			meshlabplugins/io_x3d

			# Filter plugins
			meshlabplugins/filter_sample
			meshlabplugins/filter_sample_dyn
			meshlabplugins/filter_createiso
			meshlabplugins/filter_geodesic
			meshlabplugins/filter_sample_gpu
			meshlabplugins/filter_ao
			meshlabplugins/filter_camera
			meshlabplugins/filter_clean
			meshlabplugins/filter_color_projection
			meshlabplugins/filter_colorproc
			meshlabplugins/filter_create
			meshlabplugins/filter_dirt
			meshlabplugins/filter_fractal
			meshlabplugins/filter_func
			meshlabplugins/filter_img_patch_param
			meshlabplugins/filter_isoparametrization
			meshlabplugins/filter_layer
			meshlabplugins/filter_measure
			meshlabplugins/filter_mesh_booleans
			meshlabplugins/filter_meshing
			meshlabplugins/filter_mls
			meshlabplugins/filter_mutualglobal
			meshlabplugins/filter_mutualinfo
			meshlabplugins/filter_plymc
			meshlabplugins/filter_qhull
			meshlabplugins/filter_quality
			meshlabplugins/filter_sampling
			meshlabplugins/filter_screened_poisson
			meshlabplugins/filter_sdfgpu
			meshlabplugins/filter_select
			meshlabplugins/filter_sketchfab
			meshlabplugins/filter_ssynth
			meshlabplugins/filter_texture
			meshlabplugins/filter_texture_defragmentation
			meshlabplugins/filter_trioptimize
			meshlabplugins/filter_unsharp
			meshlabplugins/filter_voronoi

			# Rendering and Decoration Plugins
			meshlabplugins/render_gdp
			meshlabplugins/render_radiance_scaling
			meshlabplugins/decorate_base
			meshlabplugins/decorate_background
			meshlabplugins/decorate_raster_proj
			meshlabplugins/decorate_shadow

			# Edit Plugins
			meshlabplugins/edit_sample
			meshlabplugins/edit_align
			meshlabplugins/edit_manipulators
			meshlabplugins/edit_measure
			meshlabplugins/edit_mutualcorrs
			meshlabplugins/edit_paint
			meshlabplugins/edit_pickpoints
			meshlabplugins/edit_point
			meshlabplugins/edit_referencing
			meshlabplugins/edit_quality
			meshlabplugins/edit_select
		)
	endif()
endif()


message(STATUS "\nConfiguring plugins")
foreach(PLUGIN ${MESHLAB_PLUGINS})
	if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${PLUGIN}/CMakeLists.txt)
		message(STATUS "- ${PLUGIN}")
		add_subdirectory(${PLUGIN})
	else()
		message(STATUS "  - ${PLUGIN} - Skipping, plugin or build system not found.")
	endif()
endforeach()


add_definitions( WIN32 )

### Copy/install other files

# This variable keeps track of the output filenames that need to be copied at build time
set(COPIED_FILES)

if (NOT BUILD_ONLY_MESHLAB_LIBRARIES)
	# shaders
	set(SHADER_BASE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../distrib/shaders")
	file(
		GLOB_RECURSE SHADERS
		LIST_DIRECTORIES false
		"${SHADER_BASE_DIR}/*.vert"
		"${SHADER_BASE_DIR}/*.frag"
		"${SHADER_BASE_DIR}/*.gdp")
	foreach(FN ${SHADERS})
		get_filename_component(SRC_PATH ${FN} DIRECTORY)
		get_filename_component(NAME_ONLY ${FN} NAME)
		file(RELATIVE_PATH REL_DIR "${SHADER_BASE_DIR}" "${SRC_PATH}")
		set(OUTFN "${MESHLAB_SHADER_OUTPUT_DIR}/${REL_DIR}/${NAME_ONLY}")
		if(REL_DIR)
			set(REL_DIR_MESSAGE "${REL_DIR} in ")
		else()
			set(REL_DIR_MESSAGE "")
		endif()
		add_custom_command(
			OUTPUT ${OUTFN}
			COMMAND ${CMAKE_COMMAND} -E make_directory "${MESHLAB_SHADER_OUTPUT_DIR}/${REL_DIR}"
			COMMAND ${CMAKE_COMMAND} -E copy_if_different "${FN}" "${OUTFN}"
			COMMENT "Copying ${NAME_ONLY} to ${REL_DIR_MESSAGE}shader build directory"
			VERBATIM
		)
		install(
			FILES ${FN}
			DESTINATION ${MESHLAB_SHADER_INSTALL_DIR}/${REL_DIR}
			COMPONENT Shaders
		)
		list(APPEND COPIED_FILES "${OUTFN}")
	endforeach()

	# Custom target - to trigger the execution of the custom commands above.
	add_custom_target(copy-distrib-files ALL DEPENDS ${COPIED_FILES})
	set_property(TARGET copy-distrib-files PROPERTY FOLDER Core)

	if(NOT WIN32 AND NOT APPLE)
		install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/../scripts/Linux/resources/meshlab.desktop" DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications)
		install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/../scripts/meshlab.png" DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/512x512/apps)
	endif()

endif()
