# Copyright(c) 2019 spdlog authors
# Distributed under the MIT License (http://opensource.org/licenses/MIT)

cmake_minimum_required(VERSION 3.1)

#---------------------------------------------------------------------------------------
# Start spdlog project
#---------------------------------------------------------------------------------------
include(cmake/utils.cmake)
spdlog_extract_version()
project(spdlog VERSION ${SPDLOG_VERSION} LANGUAGES CXX)
message(STATUS "Build spdlog: ${SPDLOG_VERSION}")

include(GNUInstallDirs)
include(cmake/ide.cmake)

#---------------------------------------------------------------------------------------
# Set default build to release
#---------------------------------------------------------------------------------------
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose Release or Debug" FORCE)
endif ()

#---------------------------------------------------------------------------------------
# Compiler config
#---------------------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

#---------------------------------------------------------------------------------------
# Set SPDLOG_MASTER_PROJECT to ON if we are building spdlog
#---------------------------------------------------------------------------------------
# Check if spdlog is being used directly or via add_subdirectory, but allow overriding
if (NOT DEFINED SPDLOG_MASTER_PROJECT)
  if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(SPDLOG_MASTER_PROJECT ON)
  else ()
    set(SPDLOG_MASTER_PROJECT OFF)
  endif ()
endif ()

option(SPDLOG_BUILD_EXAMPLES "Build examples" OFF)
option(SPDLOG_BUILD_BENCH "Build benchmarks (Requires https://github.com/google/benchmark.git to be installed)" OFF)
option(SPDLOG_BUILD_TESTS "Build tests" OFF)
option(SPDLOG_BUILD_HO_TESTS "Build tests using the header only version" OFF)
option(SPDLOG_SANITIZE_ADDRESS "Enable address sanitizer in tests" OFF)
option(SPDLOG_INSTALL "Generate the install target." ${SPDLOG_MASTER_PROJECT})
option(SPDLOG_FMT_EXTERNAL "Use external fmt library instead of bundled" OFF)

set(SPDLOG_INSTALL ON)
message(STATUS "Build type: " ${CMAKE_BUILD_TYPE} ${SPDLOG_INSTALL}
  ${SPDLOG_MASTER_PROJECT})

find_package(Threads REQUIRED)

#---------------------------------------------------------------------------------------
# Static/Shared library (shared not supported in windows yet)
#---------------------------------------------------------------------------------------
if (BUILD_SHARED_LIBS AND WIN32)
  message(WARNING "shared libs is not supported in spdlog - building static instead")
  add_library(spdlog STATIC src/spdlog.cpp ${SPDLOG_ALL_HEADERS})
else ()
  add_library(spdlog src/spdlog.cpp ${SPDLOG_ALL_HEADERS})
endif ()

add_library(spdlog::spdlog ALIAS spdlog)

target_compile_definitions(spdlog PUBLIC SPDLOG_COMPILED_LIB)
target_include_directories(spdlog PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>"
  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
target_link_libraries(spdlog PUBLIC Threads::Threads)
spdlog_enable_warnings(spdlog)

#---------------------------------------------------------------------------------------
# Shared library version
#---------------------------------------------------------------------------------------
add_library(spdlog_shared SHARED src/spdlog.cpp ${SPDLOG_ALL_HEADERS})
add_library(spdlog::spdlog_shared ALIAS spdlog_shared)

target_compile_definitions(spdlog_shared PUBLIC SPDLOG_COMPILED_LIB)
target_include_directories(spdlog_shared PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>"
  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
target_link_libraries(spdlog_shared PUBLIC Threads::Threads)
spdlog_enable_warnings(spdlog_shared)

#---------------------------------------------------------------------------------------
# Header only version
#---------------------------------------------------------------------------------------
add_library(spdlog_header_only INTERFACE)
add_library(spdlog::spdlog_header_only ALIAS spdlog_header_only)

target_include_directories(spdlog_header_only INTERFACE
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>"
  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
target_link_libraries(spdlog_header_only INTERFACE Threads::Threads)


#---------------------------------------------------------------------------------------
# Use fmt package if using exertnal fmt
#---------------------------------------------------------------------------------------
if (SPDLOG_FMT_EXTERNAL)
  if (NOT TARGET fmt::fmt)
    find_package(fmt REQUIRED)
  endif ()

  target_compile_definitions(spdlog PUBLIC SPDLOG_FMT_EXTERNAL)
  target_link_libraries(spdlog PUBLIC fmt::fmt)

  target_compile_definitions(spdlog_header_only INTERFACE SPDLOG_FMT_EXTERNAL)
  target_link_libraries(spdlog_header_only INTERFACE fmt::fmt)
endif ()

#---------------------------------------------------------------------------------------
# Build binaries
#---------------------------------------------------------------------------------------
if (SPDLOG_BUILD_EXAMPLES)
  message(STATUS "Generating examples")
  add_subdirectory(example)
endif ()

if (SPDLOG_BUILD_TESTS)
  message(STATUS "Generating tests")
  include(CTest)
  add_subdirectory(tests)
endif ()

if (SPDLOG_BUILD_BENCH)
  message(STATUS "Generating benchmarks")
  add_subdirectory(bench)
endif ()

#---------------------------------------------------------------------------------------
# Install
#---------------------------------------------------------------------------------------
if (SPDLOG_INSTALL)
  message(STATUS "Generating install")

  set(PROJECT_PREFIX ${PROJECT_NAME}-${${PROJECT_NAME}_VERSION})
  set(INSTALL_RUNTIME_DIR ${CMAKE_INSTALL_BINDIR})
  set(INSTALL_CONFIG_DIR ${CMAKE_INSTALL_LIBDIR}/${PROJECT_PREFIX}/cmake)
  set(INSTALL_LIBRARY_DIR ${CMAKE_INSTALL_LIBDIR}/${PROJECT_PREFIX}/${CMAKE_BUILD_TYPE})
  set(INSTALL_ARCHIVE_DIR ${CMAKE_INSTALL_LIBDIR}/${PROJECT_PREFIX}/${CMAKE_BUILD_TYPE}/static)
  set(INSTALL_INCLUDE_DIR ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_PREFIX}/${PROJECT_NAME})

  # Define the configuration file locations.
  set(PROJECT_CONFIG_VERSION_FILE "${PROJECT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake")
  set(PROJECT_CONFIG_FILE         "${PROJECT_BINARY_DIR}/${PROJECT_NAME}-config.cmake")

  set(project_config_in "${CMAKE_CURRENT_LIST_DIR}/cmake/spdlogConfig.cmake.in")
  set(project_config_out "${CMAKE_CURRENT_BINARY_DIR}/spdlogConfig.cmake")
  set(config_targets_file "spdlogConfigTargets.cmake")
  set(version_config_file "${CMAKE_CURRENT_BINARY_DIR}/spdlogConfigVersion.cmake")
  set(export_dest_dir "${CMAKE_INSTALL_LIBDIR}/spdlog/cmake")

  #---------------------------------------------------------------------------------------
  # Include files
  #---------------------------------------------------------------------------------------
  install(DIRECTORY include/ DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  install(TARGETS spdlog spdlog_header_only EXPORT spdlog DESTINATION "${CMAKE_INSTALL_LIBDIR}/spdlog")

  #---------------------------------------------------------------------------------------
  # Package and version files
  #---------------------------------------------------------------------------------------
  install(EXPORT spdlog
    DESTINATION ${export_dest_dir}
    NAMESPACE spdlog::
    FILE ${config_targets_file})

  include(CMakePackageConfigHelpers)
  configure_file("${project_config_in}" "${project_config_out}" @ONLY)
  configure_package_config_file(
    cmake/spdlogConfig.cmake.in
    ${PROJECT_CONFIG_FILE}
    INSTALL_DESTINATION ${INSTALL_CONFIG_DIR}
  )
  write_basic_package_version_file(
    "${version_config_file}"
    COMPATIBILITY SameMajorVersion)
  install(FILES
    "${project_config_out}"
    "${version_config_file}" DESTINATION "${export_dest_dir}")

  # The INSTALL(TARGET) command specifies where to install specific targets.
  # By specifying EXPORT here, we add these targets to an export group.
  install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}-targets
    RUNTIME DESTINATION ${INSTALL_RUNTIME_DIR}
    LIBRARY DESTINATION ${INSTALL_LIBRARY_DIR}
    ARCHIVE DESTINATION ${INSTALL_ARCHIVE_DIR})

  # The INSTALL(FILES) command specifies where to install specific files, such as headers.
  install(FILES ${${PROJECT_NAME}_HEADER_FILES}
    DESTINATION ${INSTALL_INCLUDE_DIR})

  # The INSTALL(FILES) command specifies where to install specific files, such as headers.
  install(FILES
    ${PROJECT_CONFIG_VERSION_FILE}
    ${PROJECT_CONFIG_FILE}
    DESTINATION ${INSTALL_CONFIG_DIR})

  # The INSTALL(EXPORT) command installs a previously defined export group to a script.
  install(EXPORT ${PROJECT_NAME}-targets
    FILE ${PROJECT_NAME}-targets.cmake
    NAMESPACE ${PROJECT_NAME}::
    DESTINATION ${INSTALL_CONFIG_DIR})

  export(EXPORT ${PROJECT_NAME}-targets
    FILE ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-targets.cmake
    NAMESPACE ${PROJECT_NAME}::)

  export(PACKAGE ${PROJECT_NAME})

  # message("afawefwefawefwaefwae")
  # if(BZSLAM_WITH_CATKIN)
  # message("afawefwefawefwaefwae")
  install(
    TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}_CATKIN_EXPORT
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
  )

  install(
    EXPORT ${PROJECT_NAME}_CATKIN_EXPORT
    DESTINATION share/${PROJECT_NAME}/cmake/
    FILE ${PROJECT_NAME}-targets.cmake
  )

  install(
    FILES ${PROJECT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
    DESTINATION share/${PROJECT_NAME}/
  )

  #---------------------------------------------------------------------------------------
  # Support creation of installable packages
  #---------------------------------------------------------------------------------------
  include(cmake/spdlogCPack.cmake)

endif ()
