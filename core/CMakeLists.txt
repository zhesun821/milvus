#-------------------------------------------------------------------------------
# Copyright (C) 2019-2020 Zilliz. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance
# with the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software distributed under the License
# is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
# or implied. See the License for the specific language governing permissions and limitations under the License.
#-------------------------------------------------------------------------------

cmake_minimum_required( VERSION 3.14 )
message( STATUS "Building using CMake version: ${CMAKE_VERSION}" )

set( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake" )
include( Utils )

# **************************** Build time, type and code version ****************************
get_current_time( BUILD_TIME )
message( STATUS "Build time = ${BUILD_TIME}" )

get_build_type( TARGET  BUILD_TYPE 
                        DEFAULT "Release" )
message( STATUS "Build type = ${BUILD_TYPE}" )

get_milvus_version( TARGET  MILVUS_VERSION 
                            DEFAULT "0.10.0" )
message( STATUS "Build version = ${MILVUS_VERSION}" )

get_last_commit_id( LAST_COMMIT_ID )
message( STATUS "LAST_COMMIT_ID = ${LAST_COMMIT_ID}" )

configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/src/version.h.in
                ${CMAKE_CURRENT_SOURCE_DIR}/src/version.h @ONLY )

# unset(CMAKE_EXPORT_COMPILE_COMMANDS CACHE)
set( CMAKE_EXPORT_COMPILE_COMMANDS ON )

# **************************** Project ****************************
project( milvus VERSION "${MILVUS_VERSION}" )

set( CMAKE_CXX_STANDARD 17 )
set( CMAKE_CXX_STANDARD_REQUIRED on )

set( MILVUS_SOURCE_DIR      ${PROJECT_SOURCE_DIR} )
set( MILVUS_BINARY_DIR      ${PROJECT_BINARY_DIR} )
set( MILVUS_ENGINE_SRC      ${PROJECT_SOURCE_DIR}/src )
set( MILVUS_THIRDPARTY_SRC  ${PROJECT_SOURCE_DIR}/thirdparty )

# This will set RPATH to all excutable TARGET
# self-installed dynamic libraries will be correctly linked by excutable
set( CMAKE_INSTALL_RPATH "/usr/lib" "${CMAKE_INSTALL_PREFIX}/lib" )
set( CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE )
# **************************** Dependencies ****************************
include( CTest )
include( BuildUtils )
include( DefineOptions )
import_mysql_inc()
using_ccache_if_defined( MILVUS_USE_CCACHE )

include( ExternalProject )
include( FetchContent )
set( FETCHCONTENT_BASE_DIR ${MILVUS_BINARY_DIR}/3rdparty_download )
set(FETCHCONTENT_QUIET OFF)
include( ThirdPartyPackages )

# **************************** Compiler arguments ****************************

if ( MILVUS_GPU_VERSION )
    message( STATUS "Building Milvus GPU version" )
    add_compile_definitions( "MILVUS_GPU_VERSION" )
    enable_language( CUDA )
    find_package( CUDA 10 REQUIRED )
    include_directories( ${CUDA_INCLUDE_DIRS} )
    link_directories(    ${CUDA_TOOLKIT_ROOT_DIR}/lib64 )
    set( CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -Xcompiler -fPIC -std=c++11 -D_FORCE_INLINES --expt-extended-lambda" )
    if ( CCACHE_FOUND )
        set(CMAKE_CUDA_COMPILER_LAUNCHER "${CCACHE_FOUND}")
    endif()
else ()
    message( STATUS "Building Milvus CPU version" )
endif ()

# TODO: remove after all ut set "FIU_ENABLE"
set(MILVUS_WITH_FIU "ON")

set_milvus_definition( MILVUS_WITH_PROMETHEUS   "MILVUS_WITH_PROMETHEUS" )
set_milvus_definition( ENABLE_CPU_PROFILING     "ENABLE_CPU_PROFILING" )
set_milvus_definition( MILVUS_WITH_FIU          "FIU_ENABLE" )

config_summary()

if ( CMAKE_BUILD_TYPE STREQUAL "Release" )
    append_flags( CMAKE_CXX_FLAGS FLAGS "-O3" )
    append_flags( CUDA_NVCC_FLAGS FLAGS "-O3" )
elseif ( CMAKE_BUILD_TYPE STREQUAL "Debug" )
    append_flags( CMAKE_CXX_FLAGS FLAGS "-O0" "-g" )
    append_flags( CUDA_NVCC_FLAGS FLAGS "-O0" "-g" )
else ()
    message( FATAL_ERROR "Unknown CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}" )
endif ()

append_flags( CMAKE_CXX_FLAGS
              FLAGS
                    "-fPIC"
                    "-DELPP_THREAD_SAFE"
                    "-fopenmp"
                    "-Werror"
                    )

# **************************** Source files ****************************

add_subdirectory( thirdparty )
add_subdirectory( src )

# Unittest lib
if ( BUILD_UNIT_TEST STREQUAL "ON" )
    if ( BUILD_COVERAGE STREQUAL "ON" )
        append_flags(  CMAKE_CXX_FLAGS 
                       FLAGS
                            "-fprofile-arcs"
                            "-ftest-coverage"
                            )
    endif ()
    append_flags( CMAKE_CXX_FLAGS FLAGS "-DELPP_DISABLE_LOGS")

    add_subdirectory( ${CMAKE_CURRENT_SOURCE_DIR}/unittest )
endif ()


add_custom_target( Clean-All COMMAND ${CMAKE_BUILD_TOOL} clean )

# **************************** Install ****************************

if ( NOT MILVUS_DB_PATH )
    set( MILVUS_DB_PATH "${CMAKE_INSTALL_PREFIX}" )
endif ()

if ( MILVUS_GPU_VERSION )
    set( GPU_ENABLE "true" )
else ()
    set( GPU_ENABLE "false" )
endif ()
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/conf/server_config.template
                ${CMAKE_CURRENT_SOURCE_DIR}/conf/server_config.yaml @ONLY)

install(DIRECTORY scripts/
        DESTINATION scripts
        FILE_PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ
        GROUP_EXECUTE GROUP_READ
        WORLD_EXECUTE WORLD_READ
        FILES_MATCHING PATTERN "*.sh")
install(DIRECTORY scripts/migration
        DESTINATION scripts
        FILE_PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ
        GROUP_EXECUTE GROUP_READ
        WORLD_EXECUTE WORLD_READ)
install(FILES
        conf/server_config.yaml
        DESTINATION
        conf)


# **************************** Coding style check tools ****************************
find_package( Python COMPONENTS Interpreter Development )
find_package( ClangTools )
set( BUILD_SUPPORT_DIR "${CMAKE_SOURCE_DIR}/build-support" )

if("$ENV{CMAKE_EXPORT_COMPILE_COMMANDS}" STREQUAL "1" OR CLANG_TIDY_FOUND)
  # Generate a Clang compile_commands.json "compilation database" file for use
  # with various development tools, such as Vim's YouCompleteMe plugin.
  # See http://clang.llvm.org/docs/JSONCompilationDatabase.html
  set(CMAKE_EXPORT_COMPILE_COMMANDS 1)
endif()

#
# "make lint" target
#
if ( NOT MILVUS_VERBOSE_LINT )
    set( MILVUS_LINT_QUIET "--quiet" )
endif ()

if ( NOT LINT_EXCLUSIONS_FILE )
    # source files matching a glob from a line in this file
    # will be excluded from linting (cpplint, clang-tidy, clang-format)
    set( LINT_EXCLUSIONS_FILE ${BUILD_SUPPORT_DIR}/lint_exclusions.txt )
endif ()

find_program( CPPLINT_BIN NAMES cpplint cpplint.py HINTS ${BUILD_SUPPORT_DIR} )
message( STATUS "Found cpplint executable at ${CPPLINT_BIN}" )

#
# "make lint" targets
#
add_custom_target( lint
    ${PYTHON_EXECUTABLE}    ${BUILD_SUPPORT_DIR}/run_cpplint.py
                            --cpplint_binary    ${CPPLINT_BIN}
                            --exclude_globs     ${LINT_EXCLUSIONS_FILE}
                            --source_dir        ${CMAKE_CURRENT_SOURCE_DIR}
                            ${MILVUS_LINT_QUIET}
                            )

#
# "make clang-format" and "make check-clang-format" targets
#
if ( ${CLANG_FORMAT_FOUND} )
    # runs clang format and updates files in place.
    add_custom_target( clang-format
        ${PYTHON_EXECUTABLE}    ${BUILD_SUPPORT_DIR}/run_clang_format.py
                                --clang_format_binary   ${CLANG_FORMAT_BIN}
                                --exclude_globs         ${LINT_EXCLUSIONS_FILE}
                                --source_dir            ${CMAKE_CURRENT_SOURCE_DIR}/src
                                --fix
                                ${MILVUS_LINT_QUIET} )

    # runs clang format and exits with a non-zero exit code if any files need to be reformatted
    add_custom_target( check-clang-format
        ${PYTHON_EXECUTABLE}    ${BUILD_SUPPORT_DIR}/run_clang_format.py
                                --clang_format_binary   ${CLANG_FORMAT_BIN}
                                --exclude_globs         ${LINT_EXCLUSIONS_FILE}
                                --source_dir            ${CMAKE_CURRENT_SOURCE_DIR}/src
                                ${MILVUS_LINT_QUIET} )
endif ()

#
# "make clang-tidy" and "make check-clang-tidy" targets
#
if ( ${CLANG_TIDY_FOUND} )
    # runs clang-tidy and attempts to fix any warning automatically
    add_custom_target( clang-tidy
        ${PYTHON_EXECUTABLE}    ${BUILD_SUPPORT_DIR}/run_clang_tidy.py
                                --clang_tidy_binary ${CLANG_TIDY_BIN}
                                --exclude_globs     ${LINT_EXCLUSIONS_FILE}
                                --compile_commands  ${CMAKE_BINARY_DIR}/compile_commands.json
                                --source_dir        ${CMAKE_CURRENT_SOURCE_DIR}/src
                                --fix
                                ${MILVUS_LINT_QUIET} )

    # runs clang-tidy and exits with a non-zero exit code if any errors are found.
    add_custom_target( check-clang-tidy
        ${PYTHON_EXECUTABLE}    ${BUILD_SUPPORT_DIR}/run_clang_tidy.py
                                --clang_tidy_binary ${CLANG_TIDY_BIN}
                                --exclude_globs     ${LINT_EXCLUSIONS_FILE}
                                --compile_commands  ${CMAKE_BINARY_DIR}/compile_commands.json
                                --source_dir        ${CMAKE_CURRENT_SOURCE_DIR}/src
                                ${MILVUS_LINT_QUIET} )
endif ()
