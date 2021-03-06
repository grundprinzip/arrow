# Copyright 2014 Cloudera, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 2.7)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake_modules")
include(CMakeParseArguments)

set(BUILD_SUPPORT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/build-support)

# Allow "make install" to not depend on all targets.
#
# Must be declared in the top-level CMakeLists.txt.
set(CMAKE_SKIP_INSTALL_ALL_DEPENDENCY true)


# Bootstrap thridparty dependencies
if ("$ENV{DISABLE_NATIVE_TOOLCHAIN}" STREQUAL "")
  # Enable toolchain variable if the environment is setup
  set(NATIVE_TOOLCHAIN ON)
  message(STATUS "Toolchain build.")

  # If toolchain is not set, pick a directory
  if ("$ENV{NATIVE_TOOLCHAIN}" STREQUAL "")
    set(ENV{NATIVE_TOOLCHAIN} "${CMAKE_CURRENT_SOURCE_DIR}/toolchain")
  endif()

  # Set the environment variables for dependent versions
  set(ENV{GCC_VERSION} "4.9.2")
  set(ENV{GFLAGS_VERSION} "2.0")
  set(ENV{GLOG_VERSION} "0.3.3-p1")
  set(ENV{GPERFTOOLS_VERSION} "2.3")
  set(ENV{LIBUNWIND_VERSION} "1.1")
  set(ENV{GOOGLETEST_VERSION} "20151222")

  # Setting SYSTEM_GCC will use the toolchain dependencies compiled with the original
  # host's compiler.
  if ("$ENV{SYSTEM_GCC}" STREQUAL "")
    set(GCC_ROOT $ENV{NATIVE_TOOLCHAIN}/gcc-$ENV{GCC_VERSION})
    set(CMAKE_C_COMPILER ${GCC_ROOT}/bin/gcc)
    set(CMAKE_CXX_COMPILER ${GCC_ROOT}/bin/g++)
  endif()

  # If the toolchain directory does not yet exists, we assume that the dependencies
  # should be downloaded. If the download script is not available fail the
  # configuration.
  if (NOT IS_DIRECTORY $ENV{NATIVE_TOOLCHAIN})
    set(BOOTSTRAP_CMD "${BUILD_SUPPORT_DIR}/bootstrap_toolchain.py")
    # Download and unpack the dependencies
    message(STATUS "Downloading and extracting dependencies.")
    execute_process(COMMAND ${BOOTSTRAP_CMD} RESULT_VARIABLE BOOTSTRAP_RESULT)
    if (${BOOTSTRAP_RESULT} EQUAL 0)
      message(STATUS "Toolchain bootstrap complete.")
    else()
      message(FATAL_ERROR "Toolchain bootstrap failed.")
    endif()
  else()
    message(STATUS "Native toolchain picked up at $ENV{NATIVE_TOOLCHAIN}")
  endif()
else()
  set(NATIVE_TOOLCHAIN OFF)
  message(STATUS "Native toolchain was explicitly disabled using DISABLE_NATIVE_TOOLCHAIN.")
  message(STATUS "Assuming system search path for dependencies.")
endif()

project(arrow)

# Generate a Clang compile_commands.json "compilation database" file for use
# with various development tools, such as Vim's YouCompleteMe plugin.
# See http://clang.llvm.org/docs/JSONCompilationDatabase.html
if ("$ENV{CMAKE_EXPORT_COMPILE_COMMANDS}" STREQUAL "1")
  set(CMAKE_EXPORT_COMPILE_COMMANDS 1)
endif()

############################################################
# Compiler flags
############################################################

# compiler flags that are common across debug/release builds
#  - msse4.2: Enable sse4.2 compiler intrinsics.
#  - Wall: Enable all warnings.
#  - Wno-sign-compare: suppress warnings for comparison between signed and unsigned
#    integers
#  -Wno-deprecated: some of the gutil code includes old things like ext/hash_set, ignore that
#  - pthread: enable multithreaded malloc
#  - -D__STDC_FORMAT_MACROS: for PRI* print format macros
#  -fno-strict-aliasing
#     Assume programs do not follow strict aliasing rules.
#     GCC cannot always verify whether strict aliasing rules are indeed followed due to
#     fundamental limitations in escape analysis, which can result in subtle bad code generation.
#     This has a small perf hit but worth it to avoid hard to debug crashes.
set(CXX_COMMON_FLAGS "-std=c++11 -fno-strict-aliasing -msse4.2 -Wall -Wno-sign-compare -Wno-deprecated -pthread -D__STDC_FORMAT_MACROS")

# compiler flags for different build types (run 'cmake -DCMAKE_BUILD_TYPE=<type> .')
# For all builds:
# For CMAKE_BUILD_TYPE=Debug
#   -ggdb: Enable gdb debugging
# For CMAKE_BUILD_TYPE=FastDebug
#   Same as DEBUG, except with some optimizations on.
# For CMAKE_BUILD_TYPE=Release
#   -O3: Enable all compiler optimizations
#   -g: Enable symbols for profiler tools (TODO: remove for shipping)
#   -DNDEBUG: Turn off dchecks/asserts/debug only code.
#   -fno-omit-frame-pointer
#       use frame pointers to allow simple stack frame walking for backtraces.
#       This has a small perf hit but worth it for the ability to profile in production
# For profile guided optimization (PGO) builds, in addition to the flags for release builds:
#   1. Build first with CMAKE_BUILD_TYPE_PROFILE_GEN:
#     -fprofile-generate: Indicates compiler should insert profile guided optimization events
#   2. Run the benchmarks (generates *.gcda profiling data).
#   3. Build again with CMAKE_BUILD_TYPE_PROFILE_BUILD
#     -fprofile-use: Compiler will use the profile outputs for optimizations
set(CXX_FLAGS_DEBUG "-ggdb")
set(CXX_FLAGS_FASTDEBUG "-ggdb -O1 -fno-omit-frame-pointer")
set(CXX_FLAGS_RELEASE "-O3 -g -DNDEBUG -fno-omit-frame-pointer")

set(CXX_FLAGS_PROFILE_GEN "${CXX_FLAGS_RELEASE} -fprofile-generate")
set(CXX_FLAGS_PROFILE_BUILD "${CXX_FLAGS_RELEASE} -fprofile-use")

# if no build build type is specified, default to debug builds
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug)
endif(NOT CMAKE_BUILD_TYPE)

string (TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE)

# Alias RELEASE as RELWITHDEBINFO and MINSIZEREL. These are common CMake
# release type names and this provides compatibility with the CLion IDE.
if ("${CMAKE_BUILD_TYPE}" STREQUAL "RELWITHDEBINFO" OR "${CMAKE_BUILD_TYPE}" STREQUAL "MINSIZEREL")
  set(CMAKE_BUILD_TYPE RELEASE)
endif ()

# Set compile flags based on the build type.
message("Configured for ${CMAKE_BUILD_TYPE} build (set with cmake -DCMAKE_BUILD_TYPE={release,debug,...})")
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG")
  set(CMAKE_CXX_FLAGS ${CXX_FLAGS_DEBUG})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "FASTDEBUG")
  set(CMAKE_CXX_FLAGS ${CXX_FLAGS_FASTDEBUG})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE")
  set(CMAKE_CXX_FLAGS ${CXX_FLAGS_RELEASE})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "PROFILE_GEN")
  set(CMAKE_CXX_FLAGS ${CXX_FLAGS_PROFILE_GEN})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "PROFILE_BUILD")
  set(CMAKE_CXX_FLAGS ${CXX_FLAGS_PROFILE_BUILD})
else()
  message(FATAL_ERROR "Unknown build type: ${CMAKE_BUILD_TYPE}")
endif ()

# Add common flags
set(CMAKE_CXX_FLAGS "${CXX_COMMON_FLAGS} ${CMAKE_CXX_FLAGS}")

# Determine compiler version
include(CompilerInfo)

if ("${COMPILER_FAMILY}" STREQUAL "clang")
  # Clang helpfully provides a few extensions from C++11 such as the 'override'
  # keyword on methods. This doesn't change behavior, and we selectively enable
  # it in src/gutil/port.h only on clang. So, we can safely use it, and don't want
  # to trigger warnings when we do so.
  # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-c++11-extensions")

  # Using Clang with ccache causes a bunch of spurious warnings that are
  # purportedly fixed in the next version of ccache. See the following for details:
  #
  #   http://petereisentraut.blogspot.com/2011/05/ccache-and-clang.html
  #   http://petereisentraut.blogspot.com/2011/09/ccache-and-clang-part-2.html
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Qunused-arguments")

  # Only hardcode -fcolor-diagnostics if stderr is opened on a terminal. Otherwise
  # the color codes show up as noisy artifacts.
  #
  # This test is imperfect because 'cmake' and 'make' can be run independently
  # (with different terminal options), and we're testing during the former.
  execute_process(COMMAND test -t 2 RESULT_VARIABLE ARROW_IS_TTY)
  if ((${ARROW_IS_TTY} EQUAL 0) AND (NOT ("$ENV{TERM}" STREQUAL "dumb")))
    message("Running in a controlling terminal")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcolor-diagnostics")
  else()
    message("Running without a controlling terminal or in a dumb terminal")
  endif()

  # Use libstdc++ and not libc++. The latter lacks support for tr1 in OSX
  # and since 10.9 is now the default.
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libstdc++")
endif()

# Sanity check linking option.
if (NOT ARROW_LINK)
  set(ARROW_LINK "a")
elseif(NOT ("auto" MATCHES "^${ARROW_LINK}" OR
            "dynamic" MATCHES "^${ARROW_LINK}" OR
            "static" MATCHES "^${ARROW_LINK}"))
  message(FATAL_ERROR "Unknown value for ARROW_LINK, must be auto|dynamic|static")
else()
  # Remove all but the first letter.
  string(SUBSTRING "${ARROW_LINK}" 0 1 ARROW_LINK)
endif()

# Clang does not support using ASAN and TSAN simultaneously.
if ("${ARROW_USE_ASAN}" AND "${ARROW_USE_TSAN}")
  message(SEND_ERROR "Can only enable one of ASAN or TSAN at a time")
endif()

# Flag to enable clang address sanitizer
# This will only build if clang or a recent enough gcc is the chosen compiler
if (${ARROW_USE_ASAN})
  if(NOT (("${COMPILER_FAMILY}" STREQUAL "clang") OR
          ("${COMPILER_FAMILY}" STREQUAL "gcc" AND "${COMPILER_VERSION}" VERSION_GREATER "4.8")))
    message(SEND_ERROR "Cannot use ASAN without clang or gcc >= 4.8")
  endif()

  # If UBSAN is also enabled, and we're on clang < 3.5, ensure static linking is
  # enabled. Otherwise, we run into https://llvm.org/bugs/show_bug.cgi?id=18211
  if("${ARROW_USE_UBSAN}" AND
      "${COMPILER_FAMILY}" STREQUAL "clang" AND
      "${COMPILER_VERSION}" VERSION_LESS "3.5")
    if("${ARROW_LINK}" STREQUAL "a")
      message("Using static linking for ASAN+UBSAN build")
      set(ARROW_LINK "s")
    elseif("${ARROW_LINK}" STREQUAL "d")
      message(SEND_ERROR "Cannot use dynamic linking when ASAN and UBSAN are both enabled")
    endif()
  endif()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -DADDRESS_SANITIZER")
endif()

# For any C code, use the same flags.
set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS}")

# Flag to enable clang undefined behavior sanitizer
# We explicitly don't enable all of the sanitizer flags:
# - disable 'vptr' because it currently crashes somewhere in boost::intrusive::list code
# - disable 'alignment' because unaligned access is really OK on Nehalem and we do it
#   all over the place.
if (${ARROW_USE_UBSAN})
  if(NOT (("${COMPILER_FAMILY}" STREQUAL "clang") OR
          ("${COMPILER_FAMILY}" STREQUAL "gcc" AND "${COMPILER_VERSION}" VERSION_GREATER "4.9")))
    message(SEND_ERROR "Cannot use UBSAN without clang or gcc >= 4.9")
  endif()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined -fno-sanitize=alignment,vptr -fno-sanitize-recover")
endif ()

# Flag to enable thread sanitizer (clang or gcc 4.8)
if (${ARROW_USE_TSAN})
  if(NOT (("${COMPILER_FAMILY}" STREQUAL "clang") OR
          ("${COMPILER_FAMILY}" STREQUAL "gcc" AND "${COMPILER_VERSION}" VERSION_GREATER "4.8")))
    message(SEND_ERROR "Cannot use TSAN without clang or gcc >= 4.8")
  endif()

  add_definitions("-fsanitize=thread")

  # Enables dynamic_annotations.h to actually generate code
  add_definitions("-DDYNAMIC_ANNOTATIONS_ENABLED")

  # changes atomicops to use the tsan implementations
  add_definitions("-DTHREAD_SANITIZER")

  # Disables using the precompiled template specializations for std::string, shared_ptr, etc
  # so that the annotations in the header actually take effect.
  add_definitions("-D_GLIBCXX_EXTERN_TEMPLATE=0")

  # Some of the above also need to be passed to the linker.
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pie -fsanitize=thread")

  # Strictly speaking, TSAN doesn't require dynamic linking. But it does
  # require all code to be position independent, and the easiest way to
  # guarantee that is via dynamic linking (not all 3rd party archives are
  # compiled with -fPIC e.g. boost).
  if("${ARROW_LINK}" STREQUAL "a")
    message("Using dynamic linking for TSAN")
    set(ARROW_LINK "d")
  elseif("${ARROW_LINK}" STREQUAL "s")
    message(SEND_ERROR "Cannot use TSAN with static linking")
  endif()
endif()


if ("${ARROW_USE_UBSAN}" OR "${ARROW_USE_ASAN}" OR "${ARROW_USE_TSAN}")
  # GCC 4.8 and 4.9 (latest as of this writing) don't allow you to specify a
  # sanitizer blacklist.
  if("${COMPILER_FAMILY}" STREQUAL "clang")
    # Require clang 3.4 or newer; clang 3.3 has issues with TSAN and pthread
    # symbol interception.
    if("${COMPILER_VERSION}" VERSION_LESS "3.4")
      message(SEND_ERROR "Must use clang 3.4 or newer to run a sanitizer build."
        " Try using clang from $NATIVE_TOOLCHAIN/")
    endif()
    add_definitions("-fsanitize-blacklist=${BUILD_SUPPORT_DIR}/sanitize-blacklist.txt")
  else()
    message(WARNING "GCC does not support specifying a sanitizer blacklist. Known sanitizer check failures will not be suppressed.")
  endif()
endif()

# Code coverage
if ("${ARROW_GENERATE_COVERAGE}")
  if("${CMAKE_CXX_COMPILER}" MATCHES ".*clang.*")
    # There appears to be some bugs in clang 3.3 which cause code coverage
    # to have link errors, not locating the llvm_gcda_* symbols.
    # This should be fixed in llvm 3.4 with http://llvm.org/viewvc/llvm-project?view=revision&revision=184666
    message(SEND_ERROR "Cannot currently generate coverage with clang")
  endif()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage -DCOVERAGE_BUILD")

  # For coverage to work properly, we need to use static linkage. Otherwise,
  # __gcov_flush() doesn't properly flush coverage from every module.
  # See http://stackoverflow.com/questions/28164543/using-gcov-flush-within-a-library-doesnt-force-the-other-modules-to-yield-gc
  if("${ARROW_LINK}" STREQUAL "a")
    message("Using static linking for coverage build")
    set(ARROW_LINK "s")
  elseif("${ARROW_LINK}" STREQUAL "d")
    message(SEND_ERROR "Cannot use coverage with static linking")
  endif()
endif()

# If we still don't know what kind of linking to perform, choose based on
# build type (developers like fast builds).
if ("${ARROW_LINK}" STREQUAL "a")
  if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG" OR
      "${CMAKE_BUILD_TYPE}" STREQUAL "FASTDEBUG")
    message("Using dynamic linking for ${CMAKE_BUILD_TYPE} builds")
    set(ARROW_LINK "d")
  else()
    message("Using static linking for ${CMAKE_BUILD_TYPE} builds")
    set(ARROW_LINK "s")
  endif()
endif()

# Are we using the gold linker? It doesn't work with dynamic linking as
# weak symbols aren't properly overridden, causing tcmalloc to be omitted.
# Let's flag this as an error in RELEASE builds (we shouldn't release a
# product like this).
#
# See https://sourceware.org/bugzilla/show_bug.cgi?id=16979 for details.
#
# The gold linker is only for ELF binaries, which OSX doesn't use. We can
# just skip.
if (NOT APPLE)
  execute_process(COMMAND ${CMAKE_CXX_COMPILER} -Wl,--version OUTPUT_VARIABLE LINKER_OUTPUT)
endif ()
if (LINKER_OUTPUT MATCHES "gold")
  if ("${ARROW_LINK}" STREQUAL "d" AND
      "${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE")
    message(SEND_ERROR "Cannot use gold with dynamic linking in a RELEASE build "
      "as it would cause tcmalloc symbols to get dropped")
  else()
    message("Using gold linker")
  endif()
  set(ARROW_USING_GOLD 1)
else()
  message("Using ld linker")
endif()

# Having set ARROW_LINK due to build type and/or sanitizer, it's now safe to
# act on its value.
if ("${ARROW_LINK}" STREQUAL "d")
  set(BUILD_SHARED_LIBS ON)

  # Position independent code is only necessary when producing shared objects.
  add_definitions(-fPIC)
endif()

# set compile output directory
string (TOLOWER ${CMAKE_BUILD_TYPE} BUILD_SUBDIR_NAME)

# If build in-source, create the latest symlink. If build out-of-source, which is
# preferred, simply output the binaries in the build folder
if (${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_CURRENT_BINARY_DIR})
  set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/build/${BUILD_SUBDIR_NAME}/")
  # Link build/latest to the current build directory, to avoid developers
  # accidentally running the latest debug build when in fact they're building
  # release builds.
  FILE(MAKE_DIRECTORY ${BUILD_OUTPUT_ROOT_DIRECTORY})
  if (NOT APPLE)
    set(MORE_ARGS "-T")
  endif()
EXECUTE_PROCESS(COMMAND ln ${MORE_ARGS} -sf ${BUILD_OUTPUT_ROOT_DIRECTORY}
  ${CMAKE_CURRENT_BINARY_DIR}/build/latest)
else()
  set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/${BUILD_SUBDIR_NAME}/")
endif()

# where to put generated archives (.a files)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")
set(ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")

# where to put generated libraries (.so files)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")
set(LIBRARY_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")

# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}")
include_directories(src)

############################################################
# Visibility
############################################################
# For generate_export_header() and add_compiler_export_flags().
include(GenerateExportHeader)

# add_library() wrapper that adds a second variant of the library for use in the
# exported Arrow C++ client. This variant is suffixed with "_exported" and is
# compiled with special visibility flags to hide all symbols except those that
# are part of the public ABI.
#
# There are two different kinds of exported libraries: internal and leaf.
# Internal libraries are static archives while leaf libraries are shared
# objects built from internal libraries. In practice there is only one leaf
# library: the Arrow C++ client itself.
#
# Arguments:
#
# LIB_NAME is the name of the library. It must come first. Required.
#
# SRCS is the list of source files to compile into the library. Required.
#
# DEPS is the list of targets that both library variants depend on. Required.
#
# NONLINK_DEPS is the list of (non-linked) targets that both library variants
# depend on. Optional.
#
# COMPILE_FLAGS is a string containing any additional compilation flags that
# should be added to both library variants. Optional.
#
# EXPORTED_SHARED is a toggle that, if set, indicates that the exported variant
# is a "leaf" library. Otherwise it is an "internal" library. Optional.
#
# EXPORTED_OUTPUT_NAME is a string describing a different file name for the
# exported library variant. If not set, defaults to LIB_NAME. Optional.
#
# EXPORTED_OUTPUT_DIRECTORY is a string describing a different directory where
# the exported library variant should be written. If not set, defaults to the
# directory where this function was called. Optional.
#
# EXPORTED_DEPS is a list of targets that the exported library variant depends
# on. If not set, defaults to DEPS. Optional.
function(ADD_EXPORTABLE_LIBRARY LIB_NAME)
  # Parse the arguments.
  set(options EXPORTED_SHARED)
  set(one_value_args COMPILE_FLAGS EXPORTED_OUTPUT_NAME EXPORTED_OUTPUT_DIRECTORY)
  set(multi_value_args SRCS DEPS EXPORTED_DEPS NONLINK_DEPS)
  cmake_parse_arguments(ARG "${options}" "${one_value_args}" "${multi_value_args}" ${ARGN})
  if(ARG_UNPARSED_ARGUMENTS)
    message(SEND_ERROR "Error: unrecognized arguments: ${ARG_UNPARSED_ARGUMENTS}")
  endif()

  # First add the regular version of the library. It uses
  # whatever linkage was defined globally.
  add_library(${LIB_NAME} ${ARG_SRCS})
  if(ARG_COMPILE_FLAGS)
    set_target_properties(${LIB_NAME}
      PROPERTIES COMPILE_FLAGS ${ARG_COMPILE_FLAGS})
  endif()
  target_link_libraries(${LIB_NAME} ${ARG_DEPS})
  if(ARG_NONLINK_DEPS)
    add_dependencies(${LIB_NAME} ${ARG_NONLINK_DEPS})
  endif()

  # Now start setting up the exported variant.
  set(EXPORTED_LIB_NAME ${LIB_NAME}_exported)
  if(ARG_EXPORTED_SHARED)
    # Leaf library.
    set(EXPORTED_LINKAGE "SHARED")
    set(EXPORTED_LINK_PRIVATE "LINK_PRIVATE")
  else()
    # Internal library.
    set(EXPORTED_LINKAGE "STATIC")
    set(EXPORTED_LINK_PRIVATE)
  endif()
  add_library(${EXPORTED_LIB_NAME} ${EXPORTED_LINKAGE} ${ARG_SRCS})

  # Compile with visibility flags:
  # - default for classes annotated with ARROW_EXPORT.
  # - hidden for classes annotated with ARROW_NO_EXPORT.
  # - hidden for everything else.
  add_compiler_export_flags(EXPORTED_FLAGS)

  # Exported variants are either static archives that will be linked to a shared
  # object, or shared objects. Either way, -fPIC is needed.
  if("${ARROW_LINK}" STREQUAL "s")
    set(EXPORTED_FLAGS "${EXPORTED_FLAGS} -fPIC")
  endif()

  # We need to remove some definitions previously added at directory scope.
  # There doesn't appear to be a good way to do this in cmake, so we do it via
  # the compiler with -U (e.g. "-UFOO" means "undefine the FOO definition").
  # Adding insult to injury, the COMPILE_DEFINITIONS property adds a -D prefix
  # to anything passed into it, so we're forced to handle the removal via
  # COMPILE_FLAGS, which, lucky for us, is emitted on the command line after
  # COMPILE_DEFINITIONS.

  # Exported variants need ARROW_EXPORT definitions to take effect.
  set(EXPORTED_FLAGS "${EXPORTED_FLAGS} -UARROW_STATIC_DEFINE")

  # Exported variants may not use tcmalloc.
  set(EXPORTED_FLAGS "${EXPORTED_FLAGS} -UTCMALLOC_ENABLED")

  set_target_properties(${EXPORTED_LIB_NAME}
    PROPERTIES COMPILE_FLAGS "${ARG_COMPILE_FLAGS} ${EXPORTED_FLAGS}")

  # Handle EXPORTED_OUTPUT_NAME and EXPORTED_OUTPUT_DIRECTORY.
  if(ARG_EXPORTED_OUTPUT_NAME)
    set_target_properties(${EXPORTED_LIB_NAME}
      PROPERTIES LIBRARY_OUTPUT_NAME ${ARG_EXPORTED_OUTPUT_NAME})
  endif()
  if(ARG_EXPORTED_OUTPUT_DIRECTORY)
    set_target_properties(${EXPORTED_LIB_NAME}
      PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${ARG_EXPORTED_OUTPUT_DIRECTORY})
  endif()

  # Set up exported variant dependent targets.
  #
  # Every linked dependency is suffixed with "_exported". This is fine; the
  # exported target graph is expected to be complete, and ADD_THIRDPARTY_LIB
  # will provide an "exported variant" for each third party target.
  if(ARG_EXPORTED_DEPS)
    set(EXPORTED_DEPS ${ARG_EXPORTED_DEPS})
  else()
    set(EXPORTED_DEPS ${ARG_DEPS})
  endif()
  foreach(DEP ${EXPORTED_DEPS})
    list(APPEND EXPORTED_SUFFIXED_DEPS "${DEP}_exported")
  endforeach()
  target_link_libraries(${EXPORTED_LIB_NAME} ${EXPORTED_LINK_PRIVATE} ${EXPORTED_SUFFIXED_DEPS})
  if(ARG_NONLINK_DEPS)
    add_dependencies(${EXPORTED_LIB_NAME} ${ARG_NONLINK_DEPS})
  endif()
endfunction()

############################################################
# Testing
############################################################

# Add a new test case, with or without an executable that should be built.
#
# REL_TEST_NAME is the name of the test. It may be a single component
# (e.g. monotime-test) or contain additional components (e.g.
# net/net_util-test). Either way, the last component must be a globally
# unique name.
#
# Arguments after the test name will be passed to set_tests_properties().
function(ADD_ARROW_TEST REL_TEST_NAME)
  if(NO_TESTS)
    return()
  endif()
  get_filename_component(TEST_NAME ${REL_TEST_NAME} NAME_WE)

  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${REL_TEST_NAME}.cc)
    # This test has a corresponding .cc file, set it up as an executable.
    set(TEST_PATH "${EXECUTABLE_OUTPUT_PATH}/${TEST_NAME}")
    add_executable(${TEST_NAME} "${REL_TEST_NAME}.cc")
    target_link_libraries(${TEST_NAME} ${ARROW_TEST_LINK_LIBS})
  else()
    # No executable, just invoke the test (probably a script) directly.
    set(TEST_PATH ${CMAKE_CURRENT_SOURCE_DIR}/${REL_TEST_NAME})
  endif()

  add_test(${TEST_NAME}
    ${BUILD_SUPPORT_DIR}/run-test.sh ${TEST_PATH})
  if(ARGN)
    set_tests_properties(${TEST_NAME} PROPERTIES ${ARGN})
  endif()
endfunction()

# A wrapper for add_dependencies() that is compatible with NO_TESTS.
function(ADD_ARROW_TEST_DEPENDENCIES REL_TEST_NAME)
  if(NO_TESTS)
    return()
  endif()
  get_filename_component(TEST_NAME ${REL_TEST_NAME} NAME_WE)

  add_dependencies(${TEST_NAME} ${ARGN})
endfunction()

enable_testing()

############################################################
# Dependencies
############################################################
function(ADD_THIRDPARTY_LIB LIB_NAME)
  set(options)
  set(one_value_args SHARED_LIB STATIC_LIB)
  set(multi_value_args DEPS)
  cmake_parse_arguments(ARG "${options}" "${one_value_args}" "${multi_value_args}" ${ARGN})
  if(ARG_UNPARSED_ARGUMENTS)
    message(SEND_ERROR "Error: unrecognized arguments: ${ARG_UNPARSED_ARGUMENTS}")
  endif()

  if(("${ARROW_LINK}" STREQUAL "s" AND ARG_STATIC_LIB) OR (NOT ARG_SHARED_LIB))
    if(NOT ARG_STATIC_LIB)
      message(FATAL_ERROR "No static or shared library provided for ${LIB_NAME}")
    endif()
    add_library(${LIB_NAME} STATIC IMPORTED)
    set_target_properties(${LIB_NAME}
      PROPERTIES IMPORTED_LOCATION "${ARG_STATIC_LIB}")
    message("Added static library dependency ${LIB_NAME}: ${ARG_STATIC_LIB}")
  else()
    add_library(${LIB_NAME} SHARED IMPORTED)
    set_target_properties(${LIB_NAME}
      PROPERTIES IMPORTED_LOCATION "${ARG_SHARED_LIB}")
    message("Added shared library dependency ${LIB_NAME}: ${ARG_SHARED_LIB}")
  endif()

  if(ARG_DEPS)
    set_target_properties(${LIB_NAME}
      PROPERTIES IMPORTED_LINK_INTERFACE_LIBRARIES "${ARG_DEPS}")
  endif()

  # Set up an "exported variant" for this thirdparty library (see "Visibility"
  # above). It's the same as the real target, just with an "_exported" suffix.
  # We prefer the static archive if it exists (as it's akin to an "internal"
  # library), but we'll settle for the shared object if we must.
  #
  # A shared object exported variant will force any "leaf" library that
  # transitively depends on it to also depend on it at runtime; this is
  # desirable for some libraries (e.g. cyrus_sasl).
  set(LIB_NAME_EXPORTED ${LIB_NAME}_exported)
  if(ARG_STATIC_LIB)
    add_library(${LIB_NAME_EXPORTED} STATIC IMPORTED)
    set_target_properties(${LIB_NAME_EXPORTED}
      PROPERTIES IMPORTED_LOCATION "${ARG_STATIC_LIB}")
  else()
    add_library(${LIB_NAME_EXPORTED} SHARED IMPORTED)
    set_target_properties(${LIB_NAME_EXPORTED}
      PROPERTIES IMPORTED_LOCATION "${ARG_SHARED_LIB}")
  endif()
  if(ARG_DEPS)
    set_target_properties(${LIB_NAME_EXPORTED}
      PROPERTIES IMPORTED_LINK_INTERFACE_LIBRARIES "${ARG_DEPS}")
  endif()
endfunction()

## GLog
find_package(GLog REQUIRED)
include_directories(SYSTEM ${GLOG_INCLUDE_DIR})
ADD_THIRDPARTY_LIB(glog
  STATIC_LIB "${GLOG_STATIC_LIB}"
  SHARED_LIB "${GLOG_SHARED_LIB}")
list(APPEND ARROW_BASE_LIBS glog)

## libunwind (dependent of glog)
## Doesn't build on OSX.
if (NOT APPLE)
  find_package(LibUnwind REQUIRED)
  include_directories(SYSTEM ${UNWIND_INCLUDE_DIR})
  ADD_THIRDPARTY_LIB(unwind
    STATIC_LIB "${UNWIND_STATIC_LIB}"
    SHARED_LIB "${UNWIND_SHARED_LIB}")
  list(APPEND ARROW_BASE_LIBS unwind)
endif()

## GFlags
find_package(GFlags REQUIRED)
include_directories(SYSTEM ${GFLAGS_INCLUDE_DIR})
ADD_THIRDPARTY_LIB(gflags
  STATIC_LIB "${GFLAGS_STATIC_LIB}"
  SHARED_LIB "${GFLAGS_SHARED_LIB}")
list(APPEND ARROW_BASE_LIBS gflags)

## GMock
find_package(GMock REQUIRED)
include_directories(SYSTEM ${GMOCK_INCLUDE_DIR} ${GTEST_INCLUDE_DIR})
ADD_THIRDPARTY_LIB(gmock
  STATIC_LIB ${GMOCK_STATIC_LIBRARY})

## Google PerfTools
##
## Disabled with TSAN/ASAN as well as with gold+dynamic linking (see comment
## near definition of ARROW_USING_GOLD).
find_package(GPerf REQUIRED)
if (NOT "${ARROW_USE_ASAN}" AND
    NOT "${ARROW_USE_TSAN}" AND
    NOT ("${ARROW_USING_GOLD}" AND "${ARROW_LINK}" STREQUAL "d"))
  ADD_THIRDPARTY_LIB(tcmalloc
    STATIC_LIB "${TCMALLOC_STATIC_LIB}"
    SHARED_LIB "${TCMALLOC_SHARED_LIB}")
  ADD_THIRDPARTY_LIB(profiler
    STATIC_LIB "${PROFILER_STATIC_LIB}"
    SHARED_LIB "${PROFILER_SHARED_LIB}")
  list(APPEND ARROW_BASE_LIBS tcmalloc profiler)
  add_definitions("-DTCMALLOC_ENABLED")
  set(ARROW_TCMALLOC_AVAILABLE 1)
endif()

## Boost
# It's important that Boost come last in the list of packages, because it's the only
# dependency that we don't currently bundle in thirdparty/. If we put this earlier
# in list, then we might end up with /usr/local/include taking precedence over
# thirdparty/installed/include and pulling in the wrong version of other dependencies
# that might be installed in the system.

### Workaround for http://stackoverflow.com/questions/9948375/cmake-find-package-succeeds-but-returns-wrong-path
# set(Boost_NO_BOOST_CMAKE ON)

# Find Boost static libraries.
# set(Boost_USE_STATIC_LIBS ON)
# find_package(Boost COMPONENTS system thread REQUIRED)
# set(BOOST_STATIC_LIBS ${Boost_LIBRARIES})
# list(LENGTH BOOST_STATIC_LIBS BOOST_STATIC_LIBS_LEN)
# list(SORT BOOST_STATIC_LIBS)

# Find Boost shared libraries.
# set(Boost_USE_STATIC_LIBS OFF)
# find_package(Boost COMPONENTS system thread REQUIRED)
# set(BOOST_SHARED_LIBS ${Boost_LIBRARIES})
# list(LENGTH BOOST_SHARED_LIBS BOOST_SHARED_LIBS_LEN)
# list(SORT BOOST_SHARED_LIBS)

# We should have found the same number of libraries both times.
# if(NOT ${BOOST_SHARED_LIBS_LEN} EQUAL ${BOOST_STATIC_LIBS_LEN})
#   set(ERROR_MSG "Boost static and shared libraries are inconsistent.")
#   set(ERROR_MSG "${ERROR_MSG} Static libraries: ${BOOST_STATIC_LIBS}.")
#   set(ERROR_MSG "${ERROR_MSG} Shared libraries: ${BOOST_SHARED_LIBS}.")
#   message(FATAL_ERROR "${ERROR_MSG}")
# endif()

# Add each pair of static/shared libraries.
# math(EXPR LAST_IDX "${BOOST_STATIC_LIBS_LEN} - 1")
# foreach(IDX RANGE ${LAST_IDX})
#   list(GET BOOST_STATIC_LIBS ${IDX} BOOST_STATIC_LIB)
#   list(GET BOOST_SHARED_LIBS ${IDX} BOOST_SHARED_LIB)

#   # Remove the prefix/suffix from the library name.
#   #
#   # e.g. libboost_system-mt --> boost_system
#   get_filename_component(LIB_NAME ${BOOST_STATIC_LIB} NAME_WE)
#   string(REGEX REPLACE "lib([^-]*)(-mt)?" "\\1" LIB_NAME_NO_PREFIX_SUFFIX ${LIB_NAME})
#   ADD_THIRDPARTY_LIB(${LIB_NAME_NO_PREFIX_SUFFIX}
#     STATIC_LIB "${BOOST_STATIC_LIB}"
#     SHARED_LIB "${BOOST_SHARED_LIB}")
#   list(APPEND ARROW_BASE_LIBS ${LIB_NAME_NO_PREFIX_SUFFIX})
# endforeach()
# include_directories(SYSTEM ${Boost_INCLUDE_DIR})

############################################################
# Linker setup
############################################################
set(ARROW_MIN_TEST_LIBS arrow_test_main arrow_test_util ${ARROW_BASE_LIBS})
set(ARROW_TEST_LINK_LIBS ${ARROW_MIN_TEST_LIBS})

############################################################
# "make ctags" target
############################################################
if (UNIX)
  add_custom_target(ctags ctags -R --languages=c++,c)
endif (UNIX)

############################################################
# "make etags" target
############################################################
if (UNIX)
  add_custom_target(tags etags --members --declarations
  `find ${CMAKE_CURRENT_SOURCE_DIR}/src
   -name \\*.cc -or -name \\*.hh -or -name \\*.cpp -or -name \\*.h -or -name \\*.c -or
   -name \\*.f`)
  add_custom_target(etags DEPENDS tags)
endif (UNIX)

############################################################
# "make cscope" target
############################################################
if (UNIX)
  add_custom_target(cscope find ${CMAKE_CURRENT_SOURCE_DIR}
  ( -name \\*.cc -or -name \\*.hh -or -name \\*.cpp -or
    -name \\*.h -or -name \\*.c -or -name \\*.f )
  -exec echo \"{}\" \; > cscope.files && cscope -q -b VERBATIM)
endif (UNIX)

############################################################
# "make lint" target
############################################################
if (UNIX)
  # Full lint
  add_custom_target(lint ${BUILD_SUPPORT_DIR}/cpplint.py
  --verbose=4
  --filter=-whitespace/comments,-readability/todo,-build/header_guard,-build/include_order
    `find ${CMAKE_CURRENT_SOURCE_DIR}/src -name \\*.cc -or -name \\*.h`)
endif (UNIX)


############################################################
# Subdirectories
############################################################

add_subdirectory(src/arrow)
add_subdirectory(src/arrow/util)


set(LINK_LIBS
  arrow_util)


ADD_EXPORTABLE_LIBRARY(arrow
  SRCS ${ARROW_SRCS}
  DEPS ${LINK_LIBS})
set_target_properties(arrow PROPERTIES LINKER_LANGUAGE CXX)
set_target_properties(arrow_exported PROPERTIES LINKER_LANGUAGE CXX)
