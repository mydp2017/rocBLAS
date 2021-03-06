# ########################################################################
# Copyright 2016 Advanced Micro Devices, Inc.
# ########################################################################

# We require 2.8.12 for linking libraries we specify PRIVATE, INTERFACE and
# PUBLIC keywords
cmake_minimum_required( VERSION 2.8.12 )

# MACOSX_RPATH is enabled by default policy
if( POLICY CMP0042 )
    cmake_policy( SET CMP0042 NEW )
endif( )

# CMP0063: Honor visibility properties for all target types [cmake 3.3]
if( POLICY CMP0063 )
  cmake_policy( SET CMP0063 NEW )
endif( )

if( CMAKE_GENERATOR MATCHES "NMake" )
    option( NMAKE_COMPILE_VERBOSE "Print VERBOSE compile/link msgs to the console" OFF )
    if( NMAKE_COMPILE_VERBOSE )
        set( CMAKE_START_TEMP_FILE "" )
        set( CMAKE_END_TEMP_FILE "" )
        set( CMAKE_VERBOSE_MAKEFILE 1 )
    endif( )
endif( )

# This has to be initialized before the project() command appears
# Set the default of CMAKE_BUILD_TYPE to be release, unless user specifies with -D.  MSVC_IDE does not use CMAKE_BUILD_TYPE
if( NOT DEFINED CMAKE_CONFIGURATION_TYPES AND NOT DEFINED CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." )
endif()

include( build-version )
project_version( NAME rocblas LANGUAGES CXX )

message( STATUS "rocblas_VERSION= ${rocblas_VERSION}" )
message( STATUS "CMAKE_BUILD_TYPE= ${CMAKE_BUILD_TYPE}" )

# This is incremented when the ABI to the library changes
set( rocblas_SOVERSION 1 )

list( APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake )

# This option only works for make/nmake and the ninja generators, but no reason it shouldn't be on all the time
# This tells cmake to create a compile_commands.json file that can be used with clang tooling or vim
set( CMAKE_EXPORT_COMPILE_COMMANDS ON )

# On windows, it's convenient to change the default install prefix such that it does NOT point to 'program files'
# Need to check out CMAKE_RUNTIME_OUTPUT_DIRECTORY variable, and see if that eliminates the need to modify install path
if( WIN32 AND CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT )
    set( CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}/package" CACHE PATH "Install path prefix, prepended onto install directories" FORCE )
endif( )

# Set common compile and link options
if( MSVC )
    # Following options for nMake
    message( STATUS "Detected MSVS Ver: " ${MSVC_VERSION} )

    # CMake uses huge stack frames for windows, we would like to remove.
    string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}" )
    string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS}" )
    string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS}" )
endif( )

set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-Bsymbolic")

include( build-bitness )

# Print out compiler flags for viewing/debug
message( STATUS "CMAKE_CXX_COMPILER flags: " ${CMAKE_CXX_FLAGS} )
message( STATUS "CMAKE_CXX_COMPILER debug flags: " ${CMAKE_CXX_FLAGS_DEBUG} )
message( STATUS "CMAKE_CXX_COMPILER release flags: " ${CMAKE_CXX_FLAGS_RELEASE} )
message( STATUS "CMAKE_CXX_COMPILER relwithdebinfo flags: " ${CMAKE_CXX_FLAGS_RELWITHDEBINFO} )
message( STATUS "CMAKE_EXE_LINKER link flags: " ${CMAKE_EXE_LINKER_FLAGS} )

# configure a header file to pass the CMake version settings to the source, and package the header files in the output archive
configure_file( "${PROJECT_SOURCE_DIR}/include/rocblas-version.h.in" "${PROJECT_BINARY_DIR}/include/rocblas-version.h" )

set( rocblas_headers_public
  include/rocblas.h
  #  include/rocblas.hpp
  include/rocblas-types.h
  include/rocblas-auxiliary.h
  include/rocblas-functions.h
  ${PROJECT_BINARY_DIR}/include/rocblas-version.h
)

source_group( "Header Files\\Public" FILES ${rocblas_headers_public} )

include( GNUInstallDirs )

set( BIN_INSTALL_DIR ${CMAKE_INSTALL_BINDIR} )
set( LIB_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR} )
set( INCLUDE_INSTALL_DIR ${CMAKE_INSTALL_INCLUDEDIR} )

# Build into subdirectories
add_subdirectory( src )

# The following code is setting variables to control the behavior of CPack to generate our
if( WIN32 )
    set( CPACK_SOURCE_GENERATOR "ZIP" )
    set( CPACK_GENERATOR "ZIP" )
else( )
    set( CPACK_SOURCE_GENERATOR "TGZ" )
    set( CPACK_GENERATOR "DEB;RPM" CACHE STRING "cpack list: 7Z, DEB, IFW, NSIS, NSIS64, RPM, STGZ, TBZ2, TGZ, TXZ, TZ, ZIP" )
    # set( CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON )
endif( )

# Package agnostic CPACK vars
set( CPACK_PACKAGE_CONTACT "Kent Knox <kent.knox@amd.com>" )
set( CPACK_PACKAGE_DESCRIPTION_SUMMARY "Radeon Open Compute BLAS library")
set( CPACK_PACKAGE_VERSION_MAJOR ${rocblas_VERSION_MAJOR} )
set( CPACK_PACKAGE_VERSION_MINOR ${rocblas_VERSION_MINOR} )
set( CPACK_PACKAGE_VERSION_PATCH ${rocblas_VERSION_PATCH} )
set( CPACK_PACKAGE_VERSION_TWEAK ${rocblas_VERSION_TWEAK} )
set( CPACK_PACKAGE_VERSION ${rocblas_VERSION} )
set( CPACK_PACKAGE_VENDOR "AMD")
set( CPACK_RESOURCE_FILE_LICENSE "${CMAKE_SOURCE_DIR}/../LICENSE.md" )
set( CPACK_SOURCE_IGNORE_FILES "/\\\\.git/" )

# Package specific CPACK vars
set( CPACK_DEBIAN_PACKAGE_DEPENDS "hip_hcc (>= 1.0.17174)" )
set( CPACK_RPM_PACKAGE_REQUIRES "hip_hcc >= 1.0.17174" )

if( "${CPACK_PACKAGING_INSTALL_PREFIX}" MATCHES "^/opt/rocm.*$")
    include( package-functions )
    write_rocm_package_script_files( ${PROJECT_BINARY_DIR} "rocblas" "librocblas-hcc.so" )

    set( CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${PROJECT_BINARY_DIR}/postinst;${PROJECT_BINARY_DIR}/prerm")
    set( CPACK_RPM_POST_INSTALL_SCRIPT_FILE "${PROJECT_BINARY_DIR}/postinst" )
    set( CPACK_RPM_PRE_UNINSTALL_SCRIPT_FILE "${PROJECT_BINARY_DIR}/prerm" )
endif( )

# Define all variables that influence CPack before including CPack, such as install targets
include( CPack )
