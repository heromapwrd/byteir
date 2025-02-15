# Match the minimum required version of LLVM and MLIR
cmake_minimum_required(VERSION 3.13.4)

project(byteir C CXX)

include(CheckCXXCompilerFlag)
include(CheckLanguage)

set(CMAKE_CXX_STANDARD 17)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if (NOT CMAKE_BUILD_TYPE)
  message(STATUS "No build type selected, default to Debug")
  set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build type (default Debug)" FORCE)
endif()

# CMP0116: Ninja generators transform `DEPFILE`s from `add_custom_command()`
# New in CMake 3.20. https://cmake.org/cmake/help/latest/policy/CMP0116.html
# Use the same setting as llvm-project
if (POLICY CMP0116)
  cmake_policy(SET CMP0116 OLD)
endif()

if (MSVC)
  # Disable warnings generated by onnx but treated as errors by onnx
  # See: https://github.com/onnx/onnx/issues/3530
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -wd4125")
  add_compile_definitions(_SILENCE_NONFLOATING_COMPLEX_DEPRECATION_WARNING)
else()
  if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    add_compile_options(-fno-gnu-unique)
  endif()
  # Global compile options for GCC and clang
  add_compile_options(-fno-operator-names)
endif()

option(BYTEIR_INCLUDE_TESTS "Generate build targets for the ByteIR unit tests." ON)
if (BYTEIR_INCLUDE_TESTS)
  add_definitions(-DBYTEIR_INCLUDE_TESTS)
endif()

option(BYTEIR_BUILD_EMBEDDED "Build byteir as part of another project" OFF)

option(BYTEIR_ENABLE_ASAN "Build byteir with asan" OFF)
if (BYTEIR_ENABLE_ASAN)
  if (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    message(FATAL_ERROR "ASAN should be with Debug build.")
  endif()
  set (CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fno-omit-frame-pointer -fsanitize=address")
  set (CMAKE_LINKER_FLAGS_DEBUG "${CMAKE_LINKER_FLAGS_DEBUG} -fno-omit-frame-pointer -fsanitize=address")
endif()

option(BYTEIR_ENABLE_BINDINGS_PYTHON "Enable byteir python binding" OFF)
if (BYTEIR_ENABLE_BINDINGS_PYTHON)
  set(MHLO_ENABLE_BINDINGS_PYTHON ON)
  set(MLIR_ENABLE_BINDINGS_PYTHON ON)
endif()

# Set Path
set(BYTEIR_SRC_DIR ${PROJECT_SOURCE_DIR}/..)
get_filename_component(BYTEIR_SRC_DIR "${BYTEIR_SRC_DIR}" ABSOLUTE)
message("BYTEIR_SRC_DIR = ${BYTEIR_SRC_DIR}")

if(BYTEIR_BUILD_EMBEDDED)
  set(BYTEIR_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR})
else()
  set(BYTEIR_BINARY_DIR ${CMAKE_BINARY_DIR})
endif()
get_filename_component(BYTEIR_BINARY_DIR "${BYTEIR_BINARY_DIR}" ABSOLUTE)
message("BYTEIR_BINARY_DIR = ${BYTEIR_BINARY_DIR}")

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${BYTEIR_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${BYTEIR_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${BYTEIR_BINARY_DIR}/bin)

## Build the off tree projects
set(BYTEIR_SRC_INCLUDE_DIR ${BYTEIR_SRC_DIR}/include)
set(BYTEIR_BIN_INCLUDE_DIR ${BYTEIR_BINARY_DIR}/include)

if(MSVC)
  set(BYTEIR_BIN_LIB_DIR ${BYTEIR_BINARY_DIR}/lib/${CMAKE_BUILD_TYPE})
else()
  set(BYTEIR_BIN_LIB_DIR ${BYTEIR_BINARY_DIR}/lib)
endif()

option(LLVM_INSTALL_PATH "The path to the built LLVM library")
get_filename_component(LLVM_INSTALL_PATH ${LLVM_INSTALL_PATH} ABSOLUTE)

set(LLVM_INSTALL_DIR "${LLVM_INSTALL_PATH}")
if(MSVC)
  set(LLVM_TOOLS_DIR "${LLVM_INSTALL_PATH}/${CMAKE_BUILD_TYPE}/bin")
else()
  set(LLVM_TOOLS_DIR "${LLVM_INSTALL_PATH}/bin")
endif()

if(NOT DEFINED LLVM_EXTERNAL_LIT)
  set(LLVM_EXTERNAL_LIT "${LLVM_TOOLS_DIR}/llvm-lit.py" CACHE PATH "PATH to llvm-lit")
endif()

set(MLIR_DIR "${LLVM_INSTALL_DIR}/lib/cmake/mlir")
include(MLIR.cmake)
include(mhlo.cmake)

include_directories(${BYTEIR_SRC_INCLUDE_DIR})
include_directories(${BYTEIR_BIN_INCLUDE_DIR})
include_directories(${BYTEIR_SRC_DIR}/dialects/include)
include_directories(${BYTEIR_BINARY_DIR}/dialects/include)

## Build all dialects, lib and include
add_subdirectory(${BYTEIR_SRC_DIR}/dialects ${BYTEIR_BINARY_DIR}/dialects)
add_subdirectory(${BYTEIR_SRC_DIR}/lib ${BYTEIR_BINARY_DIR}/lib)
add_subdirectory(${BYTEIR_SRC_DIR}/include ${BYTEIR_BINARY_DIR}/include)

# Build tools
add_subdirectory(${BYTEIR_SRC_DIR}/tools ${BYTEIR_BINARY_DIR}/tools)

# Build python binding
if (BYTEIR_ENABLE_BINDINGS_PYTHON)
  include(MLIRDetectPythonEnv)
  mlir_configure_python_dev_packages()
  add_subdirectory(${BYTEIR_SRC_DIR}/python ${CMAKE_BINARY_DIR}/python)
endif()

if(MSVC)
  set(BYTEIR_TOOLS_DIR ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${CMAKE_BUILD_TYPE})
else()
  set(BYTEIR_TOOLS_DIR ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
endif()
message(STATUS "BYTEIR_TOOLS_DIR ${BYTEIR_TOOLS_DIR}")

# Install headers, following https://github.com/llvm/llvm-project/blob/main/mlir/CMakeLists.txt#L158-L188
install(DIRECTORY ${BYTEIR_SRC_INCLUDE_DIR}/byteir
  DESTINATION include
  COMPONENT byteir-headers
  FILES_MATCHING
  PATTERN "*.def"
  PATTERN "*.h"
  PATTERN "*.inc"
  PATTERN "*.td"
  )

install(DIRECTORY ${BYTEIR_BIN_INCLUDE_DIR}/byteir
  DESTINATION include
  COMPONENT byteir-headers
  FILES_MATCHING
  PATTERN "*.def"
  PATTERN "*.h"
  PATTERN "*.gen"
  PATTERN "*.inc"
  PATTERN "*.td"
  PATTERN "CMakeFiles" EXCLUDE
  PATTERN "config.h" EXCLUDE
  )

if(NOT LLVM_ENABLE_IDE)
  add_llvm_install_targets(install-byteir-headers
                           DEPENDS byteir-headers
                           COMPONENT byteir-headers)
endif()

if (BYTEIR_INCLUDE_TESTS)
  ## Add unit tests
  set(BYTEIR_TEST_BUILD_DIR "${BYTEIR_BINARY_DIR}/test")
  set(BYTEIR_TEST_SOURCE_DIR "${BYTEIR_SRC_DIR}/test")
  add_subdirectory(${BYTEIR_TEST_SOURCE_DIR} ${BYTEIR_TEST_BUILD_DIR})

  ## Add numerical tests
  set(BYTEIR_NUMERICAL_BUILD_DIR "${BYTEIR_BINARY_DIR}/numerical")
  set(BYTEIR_NUMERICAL_SOURCE_DIR "${BYTEIR_SRC_DIR}/numerical")
  add_subdirectory(${BYTEIR_NUMERICAL_SOURCE_DIR} ${BYTEIR_NUMERICAL_BUILD_DIR})
endif()
