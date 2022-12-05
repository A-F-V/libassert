cmake_minimum_required(VERSION 3.8...3.23)

project(
  libassert
  VERSION 1.0.0
  LANGUAGES CXX
)

option(ASSERT_BUILD_TESTS "Enables tests" OFF)
option(ASSERT_DECOMPOSE_BINARY_LOGICAL "Enables expression decomposition of && and ||" OFF)
option(ASSERT_LOWERCASE "Enables assert alias for ASSERT" OFF)
option(ASSERT_USE_MAGIC_ENUM "Use the MagicEnum library to print better diagnostics for enum classes" ON)
set(ASSERT_FAIL "" CACHE STRING "ASSERT_FAIL")

# libassert uses relocs, -fpic has to be used to allow shared libs to link libassert as a static library.
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include(GNUInstallDirs)

add_library(magic_nums INTERFACE third_party/magic_enum.hpp)
#target_include_directories(magic_nums INTERFACE)
add_library(assert src/assert.cpp)

#set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/install)
target_link_libraries(assert PRIVATE magic_nums)

target_include_directories(
  assert
  PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/>
  $<INSTALL_INTERFACE:include>
)

target_compile_features(
  assert
  PUBLIC
  cxx_std_17
)

set_target_properties(
  assert
  PROPERTIES
  CXX_STANDARD_REQUIRED TRUE
  CXX_EXTENSIONS OFF
)

target_compile_options(
  assert
  PRIVATE
  $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall -Wextra -Werror=return-type -Wshadow>
  $<$<CXX_COMPILER_ID:MSVC>:/W4 /WX /permissive->
)

target_link_libraries(
  assert
  PRIVATE
  $<$<CXX_COMPILER_ID:MSVC>:dbghelp>
  ${CMAKE_DL_LIBS}
)

if(ASSERT_DECOMPOSE_BINARY_LOGICAL)
  target_compile_definitions(assert PUBLIC ASSERT_DECOMPOSE_BINARY_LOGICAL)
endif()

if(ASSERT_LOWERCASE)
  target_compile_definitions(assert PUBLIC ASSERT_LOWERCASE)
endif()

if(ASSERT_USE_MAGIC_ENUM)
  target_compile_definitions(assert PUBLIC ASSERT_USE_MAGIC_ENUM)
endif()

if(NOT "${ASSERT_FAIL}" STREQUAL "")
  target_compile_definitions(assert PUBLIC ASSERT_FAIL=${ASSERT_FAIL})
endif()


if(CMAKE_BUILD_TYPE STREQUAL "")
  message(FATAL_ERROR "Setting CMAKE_BUILD_TYPE is required")
endif()

if(ASSERT_BUILD_TESTS)
  include(CTest)
  add_test(
    NAME run-tests.py
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests
    COMMAND
    python3 run-tests.py --build --build_type=${CMAKE_BUILD_TYPE} --compiler=${CMAKE_CXX_COMPILER_ID}
  )
endif()

if(NOT CMAKE_SKIP_INSTALL_RULES)
  include(CMakePackageConfigHelpers)

  install(
    TARGETS assert magic_nums
    EXPORT assert_targets
    LIBRARY DESTINATION lib
    INCLUDES DESTINATION include
    ARCHIVE DESTINATION lib
  )

  install(
    FILES
    include/assert.hpp
    DESTINATION include
  )


  export(
    EXPORT assert_targets
    FILE assert_targets.cmake
    NAMESPACE assert::
  )

  configure_package_config_file(
    cmake/assert-config.cmake.in
    assert-config.cmake
    INSTALL_DESTINATION lib/cmake
  )

  write_basic_package_version_file(
    lib/cmake/assert-config-version.cmake
    COMPATIBILITY SameMajorVersion
  )

  install(
    EXPORT assert_targets
    FILE assert_targets.cmake
    NAMESPACE assert::
    DESTINATION lib/cmake
  )

  
endif()
