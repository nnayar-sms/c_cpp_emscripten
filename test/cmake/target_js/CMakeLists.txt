cmake_minimum_required(VERSION 3.16)

project(test_cmake)

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../cpp_lib ${CMAKE_CURRENT_BINARY_DIR}/cpp_lib)

if (CMAKE_BUILD_TYPE STREQUAL Debug)
  set(linkFlags "-g")
else()
  # Either MinSizeRel, RelWithDebInfo or Release, all which run with optimizations enabled.
  set(linkFlags "-O2")
endif()

# Ensure synchronous startup for this test, whose output expects it
set(linkFlags "${linkFlags} -sWASM_ASYNC_COMPILATION=0")

# Test that the CMake-provided macro check_function_exists() works.
include(CheckFunctionExists)
check_function_exists("strtod" HAVE_STRTOD)
if (NOT HAVE_STRTOD)
  message(FATAL_ERROR "CMake script check_function_exists failed! Could not detect if strtod() is supported!")
endif()

# Test that the CMake-provided macro check_function_exists() is not actually
# trivially returning true for everything.
check_function_exists("some_nonexisting_function" HAVE_NONEXISTING_FUNCTION)
if (${HAVE_NONEXISTING_FUNCTION})
  message(FATAL_ERROR "CMake script check_function_exists failed! It erroneously reports that function some_nonexisting_function() would exist!")
endif()

# Test that the CMake-provided macro check_include_file() works.
include(CheckIncludeFile)
check_include_file(stdlib.h HAVE_STDLIB_H)
if (NOT HAVE_STDLIB_H)
  message(FATAL_ERROR "CMake script check_include_file failed! Could not find stdlib.h via it!")
endif()

# Test that the CMake-provided macro check_include_file() is not actually
# trivially returning true for everything.
check_include_file(some_nonexisting_include.h HAVE_NONEXISTING_INCLUDE_H)
if (${HAVE_NONEXISTING_INCLUDE_H})
  message(FATAL_ERROR "CMake script check_include_file failed! It erroneously reports that the C header some_nonexisting_include.h would exist!")
endif()

if (NOT CMAKE_EXECUTABLE_SUFFIX STREQUAL ".js")
  message(FATAL_ERROR "The default suffix for building executables should be .js!")
endif()

# CMake built-in check_include_file() macro interacts with
# CMAKE_EXECUTABLE_SUFFIX, so make sure it works after user changes that.

check_include_file(math.h HAVE_MATH_H)
if (NOT HAVE_MATH_H)
  message(FATAL_ERROR "CMake script check_include_file failed! Could not find math.h via it!")
endif()

if (WIN32)
  message(FATAL_ERROR "WIN32 should not be defined when cross-compiling!")
endif()

if (APPLE)
  message(FATAL_ERROR "APPLE should not be defined when cross-compiling!")
endif()

if (NOT EMSCRIPTEN)
  message(FATAL_ERROR "EMSCRIPTEN should be defined when cross-compiling!")
endif()

if (NOT CMAKE_C_SIZEOF_DATA_PTR)
  message(FATAL_ERROR "CMAKE_C_SIZEOF_DATA_PTR was not defined!")
endif()

include(TestBigEndian)
test_big_endian(endian_result)
if (NOT ${endian_result} EQUAL 0)
  message(FATAL_ERROR "TEST_BIG_ENDIAN did not return 0!")
endif()

add_executable(test_cmake "main.cpp")
target_link_libraries( test_cmake cpp_lib)

# GOTCHA: If your project has custom link flags, these must be set *before*
# calling any of the em_link_xxx functions!
set_target_properties(test_cmake PROPERTIES LINK_FLAGS "${linkFlags}")

file(GLOB libraryJsFiles jslibrary*.js)
message(STATUS "js libs '${libraryJsFiles}'")
# To link .js files using the --js-library flag, use the following helper function.
em_link_js_library(test_cmake ${libraryJsFiles})

# To link .js files using the --pre-js flag, use the following helper function.
em_link_pre_js(test_cmake "prejs.js")

# Ensure that calling em_link_pre_js multiple times works as expected.
em_link_pre_js(test_cmake "prejs.js")

# To link .js files using the --post-js flag, use the following helper function.
em_link_post_js(test_cmake "postjs.js")
