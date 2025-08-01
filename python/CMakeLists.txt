# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#
# Includes code assembled from BSD/MIT/Apache-licensed code from some 3rd-party
# projects, including Kudu, Impala, and libdynd. See python/LICENSE.txt

cmake_minimum_required(VERSION 3.25)
project(pyarrow)

# This is needed for 3.13 free-threading. CMake used to add Python
# include directories with `-isystem`, which led to some Python-internal
# includes to resolve to normal 3.13 includes (cause -isystem includes
# are searched after system directories), instead of 3.13-freethreading,
# which in turn meant that Py_GIL_DISABLED was not set.
set(CMAKE_NO_SYSTEM_FROM_IMPORTED ON)

set(PYARROW_VERSION "22.0.0-SNAPSHOT")
string(REGEX MATCH "^[0-9]+\\.[0-9]+\\.[0-9]+" PYARROW_BASE_VERSION "${PYARROW_VERSION}")

# Generate SO version and full SO version
project(pyarrow VERSION "${PYARROW_BASE_VERSION}")
set(PYARROW_VERSION_MAJOR "${pyarrow_VERSION_MAJOR}")
set(PYARROW_VERSION_MINOR "${pyarrow_VERSION_MINOR}")
set(PYARROW_VERSION_PATCH "${pyarrow_VERSION_PATCH}")
# pyarrow 1.x.y => SO version is "10x", full SO version is "10x.y.0"
# Example: for 18.0.0 --> PYARROW_SO_VERSION=1800, PYARROW_FULL_SO_VERSION=1800.0.0
math(EXPR PYARROW_SO_VERSION "${PYARROW_VERSION_MAJOR} * 100 + ${PYARROW_VERSION_MINOR}")
set(PYARROW_FULL_SO_VERSION "${PYARROW_SO_VERSION}.${PYARROW_VERSION_PATCH}.0")

# Running from a Python sdist tarball
set(LOCAL_CMAKE_MODULES "${CMAKE_SOURCE_DIR}/cmake_modules")
if(EXISTS "${LOCAL_CMAKE_MODULES}")
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${LOCAL_CMAKE_MODULES})
endif()

# Running from a git source tree
set(CPP_CMAKE_MODULES "${CMAKE_SOURCE_DIR}/../cpp/cmake_modules")
if(EXISTS "${CPP_CMAKE_MODULES}")
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CPP_CMAKE_MODULES})
endif()

if(PYARROW_CPP_HOME)
  list(INSERT CMAKE_PREFIX_PATH 0 "${PYARROW_CPP_HOME}")
endif()

include(CMakeParseArguments)

# MACOSX_RPATH is enabled by default.
# https://www.cmake.org/cmake/help/latest/policy/CMP0042.html
cmake_policy(SET CMP0042 NEW)

# Only interpret if() arguments as variables or keywords when unquoted.
# https://www.cmake.org/cmake/help/latest/policy/CMP0054.html
cmake_policy(SET CMP0054 NEW)

# RPATH settings on macOS do not affect install_name.
# https://cmake.org/cmake/help/latest/policy/CMP0068.html
if(POLICY CMP0068)
  cmake_policy(SET CMP0068 NEW)
endif()

# find_package() uses <PackageName>_ROOT variables.
# https://cmake.org/cmake/help/latest/policy/CMP0074.html
if(POLICY CMP0074)
  cmake_policy(SET CMP0074 NEW)
endif()

# RPATH entries are properly escaped in the intermediary CMake install script.
# https://cmake.org/cmake/help/latest/policy/CMP0095.html
if(POLICY CMP0095)
  cmake_policy(SET CMP0095 NEW)
endif()

# Use the first Python installation on PATH, not the newest one
set(Python3_FIND_STRATEGY "LOCATION")
# On Windows, use registry last, not first
set(Python3_FIND_REGISTRY "LAST")
# On macOS, use framework last, not first
set(Python3_FIND_FRAMEWORK "LAST")

# Allow "make install" to not depend on all targets.
#
# Must be declared in the top-level CMakeLists.txt.
set(CMAKE_SKIP_INSTALL_ALL_DEPENDENCY true)

set(CMAKE_MACOSX_RPATH 1)
if(DEFINED ENV{MACOSX_DEPLOYMENT_TARGET})
  set(CMAKE_OSX_DEPLOYMENT_TARGET $ENV{MACOSX_DEPLOYMENT_TARGET})
else()
  set(CMAKE_OSX_DEPLOYMENT_TARGET 12.0)
endif()

# Generate a Clang compile_commands.json "compilation database" file for use
# with various development tools, such as Vim's YouCompleteMe plugin.
# See http://clang.llvm.org/docs/JSONCompilationDatabase.html
if("$ENV{CMAKE_EXPORT_COMPILE_COMMANDS}" STREQUAL "1")
  set(CMAKE_EXPORT_COMPILE_COMMANDS 1)
endif()

if(UNIX)
  set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
  # In the event that we are bundling the shared libraries (e.g. in a
  # manylinux1 wheel), we need to set the RPATH of the extensions to the
  # root of the pyarrow/ package so that libarrow is able to be
  # loaded properly
  if(APPLE)
    set(CMAKE_INSTALL_NAME_DIR "@rpath")
    set(CMAKE_INSTALL_RPATH "@loader_path/")
  else()
    set(CMAKE_INSTALL_RPATH "\$ORIGIN")
  endif()
endif()

find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND
   AND NOT CMAKE_C_COMPILER_LAUNCHER
   AND NOT CMAKE_CXX_COMPILER_LAUNCHER)
  message(STATUS "Using ccache: ${CCACHE_FOUND}")
  set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE_FOUND})
  set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_FOUND})
  # ARROW-3985: let ccache preserve C++ comments, because some of them may be
  # meaningful to the compiler
  set(ENV{CCACHE_COMMENTS} "1")
endif()

#
# Compiler flags
#

include(BuildUtils)

# Cython generated code emits way to many warnings at CHECKIN and EVERYTHING
set(BUILD_WARNING_LEVEL "PRODUCTION")

# This must be synchronized with the definition in
# cpp/cmake_modules/DefineOptions.cmake.
if(NOT DEFINED ARROW_SIMD_LEVEL)
  set(ARROW_SIMD_LEVEL
      "DEFAULT"
      CACHE STRING "Compile time SIMD optimization level")
endif()
if(NOT DEFINED ARROW_RUNTIME_SIMD_LEVEL)
  set(ARROW_RUNTIME_SIMD_LEVEL
      "MAX"
      CACHE STRING "Max runtime SIMD optimization level")
endif()
include(SetupCxxFlags)

if($ENV{PYODIDE})
  # These variables are needed for building PyArrow on Emscripten.
  # If they aren't set, CMake cross compiling fails for Python
  # modules (at least under Pyodide it does).
  set(Python3_INCLUDE_DIR $ENV{PYTHONINCLUDE})
  set(Python3_LIBRARY $ENV{CPYTHONLIB})
  set(Python3_EXECUTABLE)
  execute_process(COMMAND ${Python3_EXECUTABLE} -c
                          "import numpy; print(numpy.__version__)"
                  OUTPUT_VARIABLE PYODIDE_NUMPY_VERSION
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
  string(REGEX MATCH "^([0-9]+)" PYODIDE_NUMPY_MAJOR_VERSION ${PYODIDE_NUMPY_VERSION})
  if(PYODIDE_NUMPY_MAJOR_VERSION GREATER_EQUAL 2)
    set(Python3_NumPy_INCLUDE_DIR $ENV{NUMPY_LIB}/_core/include)
  else()
    set(Python3_NumPy_INCLUDE_DIR $ENV{NUMPY_LIB}/core/include)
  endif()
  set(ENV{_PYTHON_SYSCONFIGDATA_NAME} $ENV{SYSCONFIG_NAME})
  # we set the c and cxx compiler manually to bypass pywasmcross
  # which is pyodide's way of messing with C++ build parameters.
  set(CMAKE_C_COMPILER emcc)
  set(CMAKE_CXX_COMPILER em++)
endif()

# Add common flags
set(CMAKE_CXX_FLAGS "${CXX_COMMON_FLAGS} ${CMAKE_CXX_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${PYARROW_CXXFLAGS}")

if(MSVC)
  # MSVC version of -Wno-return-type-c-linkage
  string(APPEND CMAKE_CXX_FLAGS " /wd4190")

  # Cython generates some bitshift expressions that MSVC does not like in
  # __Pyx_PyFloat_DivideObjC
  string(APPEND CMAKE_CXX_FLAGS " /wd4293")

  # Converting to/from C++ bool is pretty wonky in Cython. The C4800 warning
  # seem harmless, and probably not worth the effort of working around it
  string(APPEND CMAKE_CXX_FLAGS " /wd4800")

  # See https://github.com/cython/cython/issues/4445.
  #
  # Cython 3 emits "(void)__Pyx_PyObject_CallMethod0;" to suppress a
  # "unused function" warning but the code emits another "function
  # call missing argument list" warning.
  string(APPEND CMAKE_CXX_FLAGS " /wd4551")
else()
  # Enable perf and other tools to work properly
  string(APPEND CMAKE_CXX_FLAGS " -fno-omit-frame-pointer")

  # Suppress Cython warnings
  string(APPEND CMAKE_CXX_FLAGS " -Wno-unused-variable -Wno-maybe-uninitialized")

  if(CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang" OR CMAKE_CXX_COMPILER_ID STREQUAL
                                                    "Clang")
    # Cython warnings in clang
    string(APPEND CMAKE_CXX_FLAGS " -Wno-parentheses-equality")
    string(APPEND CMAKE_CXX_FLAGS " -Wno-constant-logical-operand")
    string(APPEND CMAKE_CXX_FLAGS " -Wno-missing-declarations")
    string(APPEND CMAKE_CXX_FLAGS " -Wno-sometimes-uninitialized")

    # We have public Cython APIs which return C++ types, which are in an extern
    # "C" blog (no symbol mangling) and clang doesn't like this
    string(APPEND CMAKE_CXX_FLAGS " -Wno-return-type-c-linkage")
  endif()
endif()

# For any C code, use the same flags.
set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS}")

# Add C++-only flags, like -std=c++17
set(CMAKE_CXX_FLAGS "${CXX_ONLY_FLAGS} ${CMAKE_CXX_FLAGS}")

message(STATUS "CMAKE_C_FLAGS: ${CMAKE_C_FLAGS}")
message(STATUS "CMAKE_CXX_FLAGS: ${CMAKE_CXX_FLAGS}")

if(MSVC)
  # MSVC makes its own output directories based on the build configuration
  set(BUILD_SUBDIR_NAME "")
else()
  # Set compile output directory
  string(TOLOWER ${CMAKE_BUILD_TYPE} BUILD_SUBDIR_NAME)
endif()

# If build in-source, create the latest symlink. If build out-of-source, which is
# preferred, simply output the binaries in the build folder
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_CURRENT_BINARY_DIR})
  set(BUILD_OUTPUT_ROOT_DIRECTORY
      "${CMAKE_CURRENT_BINARY_DIR}/build/${BUILD_SUBDIR_NAME}")
  # Link build/latest to the current build directory, to avoid developers
  # accidentally running the latest debug build when in fact they're building
  # release builds.
  file(MAKE_DIRECTORY ${BUILD_OUTPUT_ROOT_DIRECTORY})
  if(NOT APPLE)
    set(MORE_ARGS "-T")
  endif()
  execute_process(COMMAND ln ${MORE_ARGS} -sf ${BUILD_OUTPUT_ROOT_DIRECTORY}
                          ${CMAKE_CURRENT_BINARY_DIR}/build/latest)
else()
  set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/${BUILD_SUBDIR_NAME}")
endif()

message(STATUS "Generator: ${CMAKE_GENERATOR}")
message(STATUS "Build output directory: ${BUILD_OUTPUT_ROOT_DIRECTORY}")

# where to put generated archives (.a files)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")
set(ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")

# where to put generated libraries (.so files)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")
set(LIBRARY_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")

# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}")

# Python and Numpy libraries
find_package(Python3Alt REQUIRED)
message(STATUS "Found NumPy version: ${Python3_NumPy_VERSION}")
message(STATUS "NumPy include dir: ${NUMPY_INCLUDE_DIRS}")

include(UseCython)
message(STATUS "Found Cython version: ${CYTHON_VERSION}")

# Arrow C++ and set default PyArrow build options
include(GNUInstallDirs)
find_package(Arrow REQUIRED)

macro(define_option name description arrow_option)
  set("PYARROW_${name}"
      "AUTO"
      CACHE STRING ${description})

  if("${PYARROW_${name}}" STREQUAL "AUTO")
    # by default, first check if env variable exists, otherwise use Arrow C++ config
    set(env_variable "PYARROW_WITH_${name}")
    if(DEFINED ENV{${env_variable}})
      if($ENV{${env_variable}})
        set("PYARROW_BUILD_${name}" ON)
      else()
        set("PYARROW_BUILD_${name}" OFF)
      endif()
    else()
      if(${arrow_option})
        set("PYARROW_BUILD_${name}" ON)
      else()
        set("PYARROW_BUILD_${name}" OFF)
      endif()
    endif()
  else()
    if("${PYARROW_${name}}")
      set("PYARROW_BUILD_${name}" ON)
    else()
      set("PYARROW_BUILD_${name}" OFF)
    endif()
  endif()
endmacro()

define_option(ACERO "Build the PyArrow Acero integration" ARROW_ACERO)
define_option(CUDA "Build the PyArrow CUDA support" ARROW_CUDA)
define_option(DATASET "Build the PyArrow Dataset integration" ARROW_DATASET)
define_option(FLIGHT "Build the PyArrow Flight integration" ARROW_FLIGHT)
define_option(GANDIVA "Build the PyArrow Gandiva integration" ARROW_GANDIVA)
define_option(ORC "Build the PyArrow ORC integration" ARROW_ORC)
define_option(PARQUET "Build the PyArrow Parquet integration" ARROW_PARQUET)
define_option(PARQUET_ENCRYPTION "Build the PyArrow Parquet encryption integration"
              PARQUET_REQUIRE_ENCRYPTION)
define_option(SUBSTRAIT "Build the PyArrow Substrait integration" ARROW_SUBSTRAIT)
define_option(AZURE "Build the PyArrow Azure integration" ARROW_AZURE)
define_option(GCS "Build the PyArrow GCS integration" ARROW_GCS)
define_option(S3 "Build the PyArrow S3 integration" ARROW_S3)
define_option(HDFS "Build the PyArrow HDFS integration" ARROW_HDFS)
option(PYARROW_BUNDLE_ARROW_CPP "Bundle the Arrow C++ libraries" OFF)
option(PYARROW_BUNDLE_CYTHON_CPP "Bundle the C++ files generated by Cython" OFF)
option(PYARROW_GENERATE_COVERAGE "Build with Cython code coverage enabled" OFF)
set(PYARROW_CXXFLAGS
    ""
    CACHE STRING "Compiler flags to append when compiling PyArrow C++")

# enforce module dependencies
if(PYARROW_BUILD_SUBSTRAIT)
  set(PYARROW_BUILD_DATASET ON)
endif()
if(PYARROW_BUILD_DATASET)
  set(PYARROW_BUILD_ACERO ON)
endif()

# PyArrow C++
set(PYARROW_CPP_ROOT_DIR pyarrow/src)
set(PYARROW_CPP_SOURCE_DIR ${PYARROW_CPP_ROOT_DIR}/arrow/python)

# Write out compile-time configuration constants
string(TOUPPER ${CMAKE_BUILD_TYPE} UPPERCASE_PYBUILD_TYPE)
configure_file("${PYARROW_CPP_SOURCE_DIR}/config_internal.h.cmake"
               "${PYARROW_CPP_SOURCE_DIR}/config_internal.h" ESCAPE_QUOTES)

set(PYARROW_CPP_SRCS
    ${PYARROW_CPP_SOURCE_DIR}/arrow_to_pandas.cc
    ${PYARROW_CPP_SOURCE_DIR}/benchmark.cc
    ${PYARROW_CPP_SOURCE_DIR}/common.cc
    ${PYARROW_CPP_SOURCE_DIR}/config.cc
    ${PYARROW_CPP_SOURCE_DIR}/datetime.cc
    ${PYARROW_CPP_SOURCE_DIR}/decimal.cc
    ${PYARROW_CPP_SOURCE_DIR}/extension_type.cc
    ${PYARROW_CPP_SOURCE_DIR}/gdb.cc
    ${PYARROW_CPP_SOURCE_DIR}/helpers.cc
    ${PYARROW_CPP_SOURCE_DIR}/inference.cc
    ${PYARROW_CPP_SOURCE_DIR}/io.cc
    ${PYARROW_CPP_SOURCE_DIR}/ipc.cc
    ${PYARROW_CPP_SOURCE_DIR}/numpy_convert.cc
    ${PYARROW_CPP_SOURCE_DIR}/numpy_init.cc
    ${PYARROW_CPP_SOURCE_DIR}/numpy_to_arrow.cc
    ${PYARROW_CPP_SOURCE_DIR}/python_test.cc
    ${PYARROW_CPP_SOURCE_DIR}/python_to_arrow.cc
    ${PYARROW_CPP_SOURCE_DIR}/pyarrow.cc
    ${PYARROW_CPP_SOURCE_DIR}/udf.cc
    ${PYARROW_CPP_SOURCE_DIR}/util.cc)
set_source_files_properties(${PYARROW_CPP_SOURCE_DIR}/numpy_init.cc
                            PROPERTIES SKIP_UNITY_BUILD_INCLUSION ON)

set(PYARROW_CPP_LINK_LIBS "")

#
# Arrow vs PyArrow C++ options
#

# Check all the options from Arrow and PyArrow C++ to be in line
#
# Order is important for "NOT ARROW_BUILD_SHARED". We must use
# depending modules -> depended modules order. For example,
# ArrowSubstrait depends on ArrowDataset. So PYARROW_CPP_LINK_LIBS
# must use
# "ArrowSubstrait::arrow_substrait_static;ArrowDataset::arrow_dataset_static"
# order.

if(PYARROW_BUILD_SUBSTRAIT)
  message(STATUS "Building PyArrow with Substrait")
  if(NOT ARROW_SUBSTRAIT)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_SUBSTRAIT=ON")
  endif()
  find_package(ArrowSubstrait REQUIRED)
  if(ARROW_BUILD_SHARED)
    list(APPEND PYARROW_CPP_LINK_LIBS ArrowSubstrait::arrow_substrait_shared)
  else()
    list(APPEND PYARROW_CPP_LINK_LIBS ArrowSubstrait::arrow_substrait_static)
  endif()
endif()

if(PYARROW_BUILD_DATASET)
  message(STATUS "Building PyArrow with Dataset")
  if(NOT ARROW_DATASET)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_DATASET=ON")
  endif()
  find_package(ArrowDataset REQUIRED)
  if(ARROW_BUILD_SHARED)
    list(APPEND PYARROW_CPP_LINK_LIBS ArrowDataset::arrow_dataset_shared)
  else()
    list(APPEND PYARROW_CPP_LINK_LIBS ArrowDataset::arrow_dataset_static)
  endif()
endif()

if(PYARROW_BUILD_ACERO)
  message(STATUS "Building PyArrow with Acero")
  if(NOT ARROW_ACERO)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_ACERO=ON")
  endif()
  find_package(ArrowAcero REQUIRED)
  if(ARROW_BUILD_SHARED)
    list(APPEND PYARROW_CPP_LINK_LIBS ArrowAcero::arrow_acero_shared)
  else()
    list(APPEND PYARROW_CPP_LINK_LIBS ArrowAcero::arrow_acero_static)
  endif()
endif()

# Currently PyArrow cannot be built without ARROW_COMPUTE
if(NOT ARROW_COMPUTE)
  message(FATAL_ERROR "You must build Arrow C++ with ARROW_COMPUTE=ON")
else()
  message(STATUS "Building PyArrow with Compute")
  find_package(ArrowCompute REQUIRED)
  if(ARROW_BUILD_SHARED)
    list(APPEND PYARROW_CPP_LINK_LIBS ArrowCompute::arrow_compute_shared)
  else()
    list(APPEND PYARROW_CPP_LINK_LIBS ArrowCompute::arrow_compute_static)
  endif()
endif()

if(PYARROW_BUILD_PARQUET)
  message(STATUS "Building PyArrow with Parquet")
  if(NOT ARROW_PARQUET)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_PARQUET=ON")
  endif()
  find_package(Parquet REQUIRED)
else()
  if(PYARROW_BUILD_PARQUET_ENCRYPTION)
    message(WARNING "Building PyArrow with Parquet Encryption is requested, but Parquet itself is not enabled. Ignoring the Parquet Encryption setting."
    )
    set(PYARROW_BUILD_PARQUET_ENCRYPTION OFF)
  endif()
endif()

# Check for only Arrow C++ options
if(ARROW_CSV)
  list(APPEND PYARROW_CPP_SRCS ${PYARROW_CPP_SOURCE_DIR}/csv.cc)
else()
  message(FATAL_ERROR "You must build Arrow C++ with ARROW_CSV=ON")
endif()

if(ARROW_FILESYSTEM)
  list(APPEND PYARROW_CPP_SRCS ${PYARROW_CPP_SOURCE_DIR}/filesystem.cc)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  set_property(SOURCE ${PYARROW_CPP_SOURCE_DIR}/pyarrow.cc
               APPEND_STRING
               PROPERTY COMPILE_FLAGS " -Wno-cast-qual ")
endif()

if(NOT PYARROW_CPP_LINK_LIBS)
  if(ARROW_BUILD_SHARED)
    list(APPEND PYARROW_CPP_LINK_LIBS Arrow::arrow_shared)
  else()
    list(APPEND PYARROW_CPP_LINK_LIBS Arrow::arrow_static)
  endif()
endif()

add_library(arrow_python SHARED ${PYARROW_CPP_SRCS})
target_include_directories(arrow_python PUBLIC ${PYARROW_CPP_ROOT_DIR}
                                               ${CMAKE_CURRENT_BINARY_DIR}/pyarrow/src)

# on static builds we need to be careful not to link PYARROW_CPP_LINK_LIBS
# into everything depending on arrow_python, or else we get duplicate
# libraries. Whereas conversely on shared builds, we need everything
# to depend on everything, as python loads modules separately
if(ARROW_BUILD_SHARED)
  target_link_libraries(arrow_python PUBLIC ${PYARROW_CPP_LINK_LIBS})
else()
  target_link_libraries(arrow_python PRIVATE ${PYARROW_CPP_LINK_LIBS})
endif()
target_link_libraries(arrow_python PUBLIC Python3::NumPy)
target_compile_definitions(arrow_python PRIVATE ARROW_PYTHON_EXPORTING)
set_target_properties(arrow_python PROPERTIES VERSION "${PYARROW_FULL_SO_VERSION}"
                                              SOVERSION "${PYARROW_SO_VERSION}")
install(TARGETS arrow_python
        ARCHIVE DESTINATION .
        LIBRARY DESTINATION .
        RUNTIME DESTINATION .)

set(PYARROW_CPP_ENCRYPTION_SRCS ${PYARROW_CPP_SOURCE_DIR}/parquet_encryption.cc)
if(NOT PYARROW_BUILD_PARQUET_ENCRYPTION)
  message(STATUS "Parquet Encryption is NOT Enabled")
else()
  if(PARQUET_REQUIRE_ENCRYPTION)
    add_library(arrow_python_parquet_encryption SHARED ${PYARROW_CPP_ENCRYPTION_SRCS})
    target_link_libraries(arrow_python_parquet_encryption PUBLIC arrow_python
                                                                 ${PARQUET_LINK_LIBS})
    target_compile_definitions(arrow_python_parquet_encryption
                               PRIVATE ARROW_PYTHON_PARQUET_ENCRYPTION_EXPORTING)
    set_target_properties(arrow_python_parquet_encryption
                          PROPERTIES VERSION "${PYARROW_FULL_SO_VERSION}"
                                     SOVERSION "${PYARROW_SO_VERSION}")
    install(TARGETS arrow_python_parquet_encryption
            ARCHIVE DESTINATION .
            LIBRARY DESTINATION .
            RUNTIME DESTINATION .)
    message(STATUS "Parquet Encryption Enabled")
  else()
    message(FATAL_ERROR "You must build Arrow C++ with PARQUET_REQUIRE_ENCRYPTION=ON")
  endif()
endif()

set(PYARROW_CPP_FLIGHT_SRCS ${PYARROW_CPP_SOURCE_DIR}/flight.cc)
if(PYARROW_BUILD_FLIGHT)
  message(STATUS "Building PyArrow with Flight")
  if(NOT ARROW_FLIGHT)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_FLIGHT=ON")
  endif()
  # Must link to shared libarrow_flight: we don't want to link more than one
  # copy of gRPC into the eventual Cython shared object, otherwise gRPC calls
  # fail with weird errors due to multiple copies of global static state (The
  # other solution is to link gRPC shared everywhere instead of statically only
  # in Flight)
  if(NOT ARROW_BUILD_SHARED)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_BUILD_SHARED=ON")
  endif()
  find_package(ArrowFlight REQUIRED)

  add_library(arrow_python_flight SHARED ${PYARROW_CPP_FLIGHT_SRCS})
  target_link_libraries(arrow_python_flight PUBLIC arrow_python
                                                   ArrowFlight::arrow_flight_shared)
  target_compile_definitions(arrow_python_flight PRIVATE ARROW_PYFLIGHT_EXPORTING)
  set_target_properties(arrow_python_flight
                        PROPERTIES VERSION "${PYARROW_FULL_SO_VERSION}"
                                   SOVERSION "${PYARROW_SO_VERSION}")
  install(TARGETS arrow_python_flight
          ARCHIVE DESTINATION .
          LIBRARY DESTINATION .
          RUNTIME DESTINATION .)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  # Clang, be quiet. Python C API has lots of macros
  set_property(SOURCE ${PYARROW_CPP_SRCS} ${PYARROW_CPP_FLIGHT_SRCS}
               APPEND_STRING
               PROPERTY COMPILE_FLAGS -Wno-parentheses-equality)
endif()

install(DIRECTORY ${PYARROW_CPP_SOURCE_DIR}/
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/arrow/python
        FILES_MATCHING
        PATTERN "*internal.h" EXCLUDE
        PATTERN "*.h")

function(bundle_arrow_lib library_path)
  set(options)
  set(one_value_args SO_VERSION)
  set(multi_value_args)
  cmake_parse_arguments(ARG
                        "${options}"
                        "${one_value_args}"
                        "${multi_value_args}"
                        ${ARGN})
  if(ARG_UNPARSED_ARGUMENTS)
    message(SEND_ERROR "Error: unrecognized arguments: ${ARG_UNPARSED_ARGUMENTS}")
  endif()

  get_filename_component(LIBRARY_PATH_REAL ${library_path} REALPATH)
  get_filename_component(LIBRARY_NAME ${library_path} NAME_WE)

  # Only copy the shared library with ABI version on Linux and macOS

  if(MSVC)
    install(FILES ${LIBRARY_PATH_REAL}
            DESTINATION "."
            RENAME ${LIBRARY_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX})
  elseif(APPLE)
    install(FILES ${LIBRARY_PATH_REAL}
            DESTINATION "."
            RENAME ${LIBRARY_NAME}.${ARG_SO_VERSION}${CMAKE_SHARED_LIBRARY_SUFFIX})
  else()
    install(FILES ${LIBRARY_PATH_REAL}
            DESTINATION "."
            RENAME ${LIBRARY_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}.${ARG_SO_VERSION})
  endif()
endfunction()

function(bundle_arrow_import_lib library_path)
  get_filename_component(LIBRARY_NAME ${library_path} NAME_WE)
  install(FILES ${library_path}
          DESTINATION "."
          RENAME ${LIBRARY_NAME}.lib)
endfunction()

function(bundle_arrow_dependency library_name)
  if(MSVC)
    if(DEFINED ENV{CONDA_PREFIX})
      file(TO_CMAKE_PATH "$ENV{CONDA_PREFIX}\\Library" SHARED_LIB_HOME)
    endif()
  else()
    if(DEFINED ENV{CONDA_PREFIX})
      file(TO_CMAKE_PATH "$ENV{CONDA_PREFIX}" SHARED_LIB_HOME)
    endif()
  endif()
  if(DEFINED ENV{${library_name}_HOME})
    file(TO_CMAKE_PATH "$ENV{${library_name}_HOME}" SHARED_LIB_HOME)
  endif()
  arrow_build_shared_library_name(shared_lib_name "${library_name}")
  unset(SHARED_LIB_PATH CACHE)
  if(MSVC)
    set(CMAKE_SHARED_LIBRARY_SUFFIXES_ORIGINAL ${CMAKE_FIND_LIBRARY_SUFFIXES})
    # .dll isn't found by find_library with MSVC because .dll isn't included in
    # CMAKE_FIND_LIBRARY_SUFFIXES.
    list(APPEND CMAKE_FIND_LIBRARY_SUFFIXES "${CMAKE_SHARED_LIBRARY_SUFFIX}")
  endif()
  if(SHARED_LIB_HOME)
    find_library(SHARED_LIB_PATH
                 NAMES "${shared_lib_name}"
                 PATHS "${SHARED_LIB_HOME}"
                 PATH_SUFFIXES ${ARROW_SEARCH_LIB_PATH_SUFFIXES}
                 NO_DEFAULT_PATH)
  else()
    find_library(SHARED_LIB_PATH
                 NAMES "${shared_lib_name}"
                 PATH_SUFFIXES ${ARROW_SEARCH_LIB_PATH_SUFFIXES})
  endif()
  if(MSVC)
    set(CMAKE_SHARED_LIBRARY_SUFFIXES ${CMAKE_FIND_LIBRARY_SUFFIXES_ORIGINAL})
  endif()
  if(SHARED_LIB_PATH)
    get_filename_component(SHARED_LIB_REALPATH ${SHARED_LIB_PATH} REALPATH)
    get_filename_component(SHARED_LIB_NAME ${SHARED_LIB_PATH} NAME)
    message(STATUS "Bundle dependency ${library_name}: ${SHARED_LIB_REALPATH} as ${SHARED_LIB_NAME}"
    )
    install(FILES ${SHARED_LIB_REALPATH}
            DESTINATION "."
            RENAME ${SHARED_LIB_NAME})
  else()
    message(FATAL_ERROR "Unable to bundle dependency: ${library_name}")
  endif()
endfunction()

# Always bundle includes
get_filename_component(ARROW_INCLUDE_ARROW_DIR_REAL ${ARROW_INCLUDE_DIR}/arrow REALPATH)
install(DIRECTORY ${ARROW_INCLUDE_ARROW_DIR_REAL} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

if(PYARROW_BUNDLE_ARROW_CPP)
  # Arrow and Compute
  bundle_arrow_lib(${ARROW_SHARED_LIB} SO_VERSION ${ARROW_SO_VERSION})
  bundle_arrow_lib(${ARROW_COMPUTE_SHARED_LIB} SO_VERSION ${ARROW_SO_VERSION})

  if(MSVC)
    bundle_arrow_import_lib(${ARROW_IMPORT_LIB})
    bundle_arrow_import_lib(${ARROW_COMPUTE_IMPORT_LIB})
  endif()
endif()

#
# Cython modules
#

set(CYTHON_EXTENSIONS
    lib
    _compute
    _csv
    _feather
    _fs
    _json
    _pyarrow_cpp_tests)
set_source_files_properties(pyarrow/lib.pyx PROPERTIES CYTHON_API TRUE)

set(LINK_LIBS arrow_python)

if(PYARROW_BUILD_AZURE)
  message(STATUS "Building PyArrow with Azure")
  if(NOT ARROW_AZURE)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_AZURE=ON")
  endif()
  list(APPEND CYTHON_EXTENSIONS _azurefs)
endif()

if(PYARROW_BUILD_GCS)
  message(STATUS "Building PyArrow with GCS")
  if(NOT ARROW_GCS)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_GCS=ON")
  endif()
  list(APPEND CYTHON_EXTENSIONS _gcsfs)
endif()

if(PYARROW_BUILD_S3)
  message(STATUS "Building PyArrow with S3")
  if(NOT ARROW_S3)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_S3=ON")
  endif()
  list(APPEND CYTHON_EXTENSIONS _s3fs)
endif()

if(PYARROW_BUILD_HDFS)
  message(STATUS "Building PyArrow with HDFS")
  if(NOT ARROW_HDFS)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_HDFS=ON")
  endif()
  list(APPEND CYTHON_EXTENSIONS _hdfs)
endif()

if(PYARROW_BUILD_CUDA)
  message(STATUS "Building PyArrow with CUDA")
  if(NOT ARROW_CUDA)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_CUDA=ON")
  endif()
  find_package(ArrowCUDA REQUIRED)

  if(PYARROW_BUNDLE_ARROW_CPP)
    bundle_arrow_lib(${ARROW_CUDA_SHARED_LIB} SO_VERSION ${ARROW_SO_VERSION})
    if(MSVC)
      bundle_arrow_import_lib(${ARROW_CUDA_IMPORT_LIB})
    endif()
  endif()
  set(CUDA_LINK_LIBS ArrowCUDA::arrow_cuda_shared)
  list(APPEND CYTHON_EXTENSIONS _cuda)
  set_source_files_properties(pyarrow/_cuda.pyx PROPERTIES CYTHON_API TRUE)
endif()

# Acero
if(PYARROW_BUILD_ACERO)
  if(ARROW_BUILD_SHARED)
    if(PYARROW_BUNDLE_ARROW_CPP)
      bundle_arrow_lib(${ARROW_ACERO_SHARED_LIB} SO_VERSION ${ARROW_SO_VERSION})
      if(MSVC)
        bundle_arrow_import_lib(${ARROW_ACERO_IMPORT_LIB})
      endif()
    endif()

    set(ACERO_LINK_LIBS ArrowAcero::arrow_acero_shared)
  else()
    # Acero is statically linked into libarrow_python already
    set(ACERO_LINK_LIBS)
  endif()
  list(APPEND CYTHON_EXTENSIONS _acero)
endif()

# Dataset
if(PYARROW_BUILD_DATASET)
  if(ARROW_BUILD_SHARED)
    if(PYARROW_BUNDLE_ARROW_CPP)
      bundle_arrow_lib(${ARROW_DATASET_SHARED_LIB} SO_VERSION ${ARROW_SO_VERSION})
      if(MSVC)
        bundle_arrow_import_lib(${ARROW_DATASET_IMPORT_LIB})
      endif()
    endif()

    set(DATASET_LINK_LIBS ArrowDataset::arrow_dataset_shared)
  else()
    # dataset is statically linked into libarrow_python already
    set(DATASET_LINK_LIBS)
  endif()
  list(APPEND CYTHON_EXTENSIONS _dataset)
endif()

# Parquet
if(PYARROW_BUILD_PARQUET)
  if(PYARROW_BUNDLE_ARROW_CPP)
    get_filename_component(PARQUET_INCLUDE_PARQUET_DIR_REAL
                           ${PARQUET_INCLUDE_DIR}/parquet REALPATH)
    install(DIRECTORY ${PARQUET_INCLUDE_PARQUET_DIR_REAL}
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
  endif()

  if(ARROW_BUILD_SHARED)
    if(PYARROW_BUNDLE_ARROW_CPP)
      bundle_arrow_lib(${PARQUET_SHARED_LIB} SO_VERSION ${PARQUET_SO_VERSION})
      if(MSVC)
        bundle_arrow_import_lib(${PARQUET_IMPORT_LIB})
      endif()
    endif()
    set(PARQUET_LINK_LIBS Parquet::parquet_shared)
  else()
    # don't link the static lib as it is
    # already in arrow_python
    set(PARQUET_LINK_LIBS)
  endif()
  list(APPEND CYTHON_EXTENSIONS _parquet)
  if(PYARROW_BUILD_PARQUET_ENCRYPTION)
    list(APPEND CYTHON_EXTENSIONS _parquet_encryption)
  endif()
  if(PYARROW_BUILD_DATASET)
    list(APPEND CYTHON_EXTENSIONS _dataset_parquet)
    if(PYARROW_BUILD_PARQUET_ENCRYPTION)
      list(APPEND CYTHON_EXTENSIONS _dataset_parquet_encryption)
    endif()
  endif()
endif()

# ORC
if(PYARROW_BUILD_ORC)
  message(STATUS "Building PyArrow with ORC")
  if(NOT ARROW_ORC)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_ORC=ON")
  endif()
  list(APPEND CYTHON_EXTENSIONS _orc)
  if(PYARROW_BUILD_DATASET)
    list(APPEND CYTHON_EXTENSIONS _dataset_orc)
  endif()
endif()

# Flight
if(PYARROW_BUILD_FLIGHT)
  if(PYARROW_BUNDLE_ARROW_CPP)
    bundle_arrow_lib(${ARROW_FLIGHT_SHARED_LIB} SO_VERSION ${ARROW_SO_VERSION})
    if(MSVC)
      bundle_arrow_import_lib(${ARROW_FLIGHT_IMPORT_LIB})
      # XXX Hardcoded library names because CMake is too stupid to give us
      # the shared library paths.
      # https://gitlab.kitware.com/cmake/cmake/issues/16210
      # bundle_arrow_dependency(libcrypto-1_1-x64)
      # bundle_arrow_dependency(libssl-1_1-x64)
    endif()
  endif()

  set(FLIGHT_LINK_LIBS arrow_python_flight)
  list(APPEND CYTHON_EXTENSIONS _flight)
else()
  set(FLIGHT_LINK_LIBS "")
endif()

# Substrait
if(PYARROW_BUILD_SUBSTRAIT)
  message(STATUS "Building PyArrow with Substrait")

  if(ARROW_BUILD_SHARED)
    if(PYARROW_BUNDLE_ARROW_CPP)
      bundle_arrow_lib(${ARROW_SUBSTRAIT_SHARED_LIB} SO_VERSION ${ARROW_SO_VERSION})
      if(MSVC)
        bundle_arrow_import_lib(${ARROW_SUBSTRAIT_IMPORT_LIB})
      endif()
    endif()
    set(SUBSTRAIT_LINK_LIBS ArrowSubstrait::arrow_substrait_shared)
  else()
    # Arrow Substrait is statically linked into libarrow_python already
    set(SUBSTRAIT_LINK_LIBS)
  endif()

  list(APPEND CYTHON_EXTENSIONS _substrait)
endif()

# Gandiva
if(PYARROW_BUILD_GANDIVA)
  message(STATUS "Building PyArrow with Gandiva")
  if(NOT ARROW_GANDIVA)
    message(FATAL_ERROR "You must build Arrow C++ with ARROW_GANDIVA=ON")
  endif()
  find_package(Gandiva REQUIRED)

  if(PYARROW_BUNDLE_ARROW_CPP)
    get_filename_component(GANDIVA_INCLUDE_GANDIVA_DIR_REAL
                           ${GANDIVA_INCLUDE_DIR}/gandiva REALPATH)
    install(DIRECTORY ${GANDIVA_INCLUDE_GANDIVA_DIR_REAL}
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

    bundle_arrow_lib(${GANDIVA_SHARED_LIB} SO_VERSION ${ARROW_SO_VERSION})

    if(MSVC)
      bundle_arrow_import_lib(${GANDIVA_IMPORT_LIB})
    endif()
  endif()

  set(GANDIVA_LINK_LIBS Gandiva::gandiva_shared)
  list(APPEND CYTHON_EXTENSIONS gandiva)
endif()

#
# Setup and build Cython modules
#

if(PYARROW_GENERATE_COVERAGE)
  set(CYTHON_FLAGS "${CYTHON_FLAGS}" "-Xlinetrace=True")
endif()

# Error on any warnings not already explicitly ignored.
set(CYTHON_FLAGS "${CYTHON_FLAGS}" "--warning-errors")
# GH-40236: make generated C++ code easier to compile by disabling an
# undocumented Cython feature.
set(CYTHON_FLAGS "${CYTHON_FLAGS}" "--no-c-in-traceback")

if(CYTHON_VERSION VERSION_GREATER_EQUAL "3.1.0a0")
  list(APPEND CYTHON_FLAGS "-Xfreethreading_compatible=True")
endif()

foreach(module ${CYTHON_EXTENSIONS})
  string(REPLACE "." ";" directories ${module})
  list(GET directories -1 module_name)
  list(REMOVE_AT directories -1)

  string(REPLACE "." "/" module_root "${module}")
  set(module_SRC pyarrow/${module_root}.pyx)
  set_source_files_properties(${module_SRC} PROPERTIES CYTHON_IS_CXX TRUE)

  cython_add_module(${module_name} ${module_name}_pyx ${module_name}_output ${module_SRC})

  if(directories)
    string(REPLACE ";" "/" module_output_directory ${directories})
    set_target_properties(${module_name} PROPERTIES LIBRARY_OUTPUT_DIRECTORY
                                                    ${module_output_directory})
  endif()

  # XXX(wesm): ARROW-2326 this logic is only needed when we have Cython
  # modules in interior directories. Since all of our C extensions and
  # bundled libraries are in the same place, we can skip this part

  # list(LENGTH directories i)
  # while(${i} GREATER 0)
  #   set(module_install_rpath "${module_install_rpath}/..")
  #   math(EXPR i "${i} - 1" )
  # endwhile(${i} GREATER 0)

  if(PYARROW_GENERATE_COVERAGE)
    set_target_properties(${module_name} PROPERTIES COMPILE_DEFINITIONS
                                                    "CYTHON_TRACE=1;CYTHON_TRACE_NOGIL=1")
  endif()

  target_link_libraries(${module_name} PRIVATE ${LINK_LIBS})

  install(TARGETS ${module_name} LIBRARY DESTINATION ".")
  foreach(output ${${module_name}_output})
    if(output MATCHES "\\.${CYTHON_CXX_EXTENSION}$")
      if(NOT PYARROW_BUNDLE_CYTHON_CPP)
        continue()
      endif()
    endif()
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${output} DESTINATION ".")
  endforeach()
endforeach()

set(ARROW_PYTHON_BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/pyarrow/src/arrow/python")
file(MAKE_DIRECTORY ${ARROW_PYTHON_BINARY_DIR})
add_custom_command(OUTPUT "${ARROW_PYTHON_BINARY_DIR}/lib_api.h"
                          "${ARROW_PYTHON_BINARY_DIR}/lib.h"
                   COMMAND ${CMAKE_COMMAND} -E copy
                           "${CMAKE_CURRENT_BINARY_DIR}/lib_api.h"
                           "${CMAKE_CURRENT_BINARY_DIR}/lib.h"
                           "${ARROW_PYTHON_BINARY_DIR}/"
                   DEPENDS lib_pyx)
add_custom_target(cython_api_headers DEPENDS "${ARROW_PYTHON_BINARY_DIR}/lib_api.h"
                                             "${ARROW_PYTHON_BINARY_DIR}/lib.h")
add_dependencies(arrow_python cython_api_headers)
install(FILES "${ARROW_PYTHON_BINARY_DIR}/lib_api.h" "${ARROW_PYTHON_BINARY_DIR}/lib.h"
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/arrow/python)

# Additional link libraries

if(PYARROW_BUILD_CUDA)
  target_link_libraries(_cuda PRIVATE ${CUDA_LINK_LIBS})
endif()

if(PYARROW_BUILD_FLIGHT)
  target_link_libraries(_flight PRIVATE ${FLIGHT_LINK_LIBS})
endif()

if(PYARROW_BUILD_SUBSTRAIT)
  target_link_libraries(_substrait PRIVATE ${SUBSTRAIT_LINK_LIBS})
endif()

if(PYARROW_BUILD_ACERO)
  target_link_libraries(_acero PRIVATE ${ACERO_LINK_LIBS})
endif()

if(PYARROW_BUILD_DATASET)
  target_link_libraries(_dataset PRIVATE ${DATASET_LINK_LIBS})
  if(PYARROW_BUILD_ORC)
    target_link_libraries(_dataset_orc PRIVATE ${DATASET_LINK_LIBS})
  endif()
  if(PYARROW_BUILD_PARQUET)
    target_link_libraries(_dataset_parquet PRIVATE ${DATASET_LINK_LIBS})
  endif()
endif()

if(PYARROW_BUILD_GANDIVA)
  target_link_libraries(gandiva PRIVATE ${GANDIVA_LINK_LIBS})
endif()

if(PYARROW_BUILD_PARQUET)
  target_link_libraries(_parquet PRIVATE ${PARQUET_LINK_LIBS})
  if(PYARROW_BUILD_PARQUET_ENCRYPTION)
    target_link_libraries(_parquet_encryption PRIVATE arrow_python_parquet_encryption)
  endif()
endif()
