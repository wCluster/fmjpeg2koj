cmake_minimum_required(VERSION 3.28)

cmake_policy(SET CMP0091 NEW)

project(fmjpeg2k
  VERSION 1.0.3
  LANGUAGES CXX
)

option(BUILD_SHARED_LIBS "Build fmjpeg2k shared library and link executables against it." OFF)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# vcpkg triplet x64-windows-static-md-v142 uses static libraries with dynamic CRT.
# Release should use /MD, Debug should use /MDd.
if(MSVC)
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
endif()

find_package(DCMTK REQUIRED CONFIG)
find_package(OpenJPEG REQUIRED)

set(fmjpeg2k_HEADERS
  include/fmjpeg2k/djcodecd.h
  include/fmjpeg2k/djcodece.h
  include/fmjpeg2k/djcparam.h
  include/fmjpeg2k/djdecode.h
  include/fmjpeg2k/djencode.h
  include/fmjpeg2k/djerror.h
  include/fmjpeg2k/djlsutil.h
  include/fmjpeg2k/djrparam.h
  include/fmjpeg2k/dldefine.h
  include/fmjpeg2k/memory_file.h
)

set(fmjpeg2k_SRCS
  ${fmjpeg2k_HEADERS}
  djcodecd.cc
  djcodece.cc
  djcparam.cc
  djdecode.cc
  djencode.cc
  djrparam.cc
  djutils.cc
  memory_file.cpp
)

# OpenJPEG target names differ across packages. Prefer imported targets when present.
if(TARGET OpenJPEG::OpenJPEG)
  set(FMJPEG2K_OPENJPEG_LINK OpenJPEG::OpenJPEG)
elseif(TARGET OpenJPEG::openjp2)
  set(FMJPEG2K_OPENJPEG_LINK OpenJPEG::openjp2)
elseif(TARGET openjp2)
  set(FMJPEG2K_OPENJPEG_LINK openjp2)
else()
  set(FMJPEG2K_OPENJPEG_LINK ${OPENJPEG_LIBRARIES})
endif()

add_library(fmjpeg2k ${fmjpeg2k_SRCS})
set(fmjpeg2k_LIBRARY_NAME fmjpeg2k)

target_include_directories(fmjpeg2k
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

if(OPENJPEG_INCLUDE_DIRS)
  target_include_directories(fmjpeg2k PRIVATE ${OPENJPEG_INCLUDE_DIRS})
endif()

if(DCMTK_INCLUDE_DIRS)
  target_include_directories(fmjpeg2k PRIVATE ${DCMTK_INCLUDE_DIRS})
endif()

# Static fmjpeg2k consumers must not see __declspec(dllimport) on fmjpeg2k symbols.
if(NOT BUILD_SHARED_LIBS)
  target_compile_definitions(fmjpeg2k
    PUBLIC
      FMJPEG2K_STATIC
      OPJ_STATIC
  )
endif()

if(WIN32)
  target_compile_definitions(fmjpeg2k
    PRIVATE
      _BIND_TO_CURRENT_VCLIBS_VERSION=1
  )
endif()

target_link_libraries(fmjpeg2k
  PUBLIC
    DCMTK::DCMTK
    ${FMJPEG2K_OPENJPEG_LINK}
)

set_property(TARGET fmjpeg2k PROPERTY VERSION ${PROJECT_VERSION})
set_property(TARGET fmjpeg2k PROPERTY SOVERSION 1)
set_property(TARGET fmjpeg2k PROPERTY INTERFACE_fmjpeg2k_MAJOR_VERSION 1)
set_property(TARGET fmjpeg2k APPEND PROPERTY
  COMPATIBLE_INTERFACE_STRING fmjpeg2k_MAJOR_VERSION
)

add_executable(dcmcjp2k dcmcjp2k.cc)
target_link_libraries(dcmcjp2k PRIVATE fmjpeg2k)

add_executable(dcmdjp2k dcmdjp2k.cc)
target_link_libraries(dcmdjp2k PRIVATE fmjpeg2k)

include(CMakePackageConfigHelpers)

set(ConfigPackageLocation lib/cmake/fmjpeg2k)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/fmjpeg2kConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/fmjpeg2kConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/fmjpeg2kConfig.cmake"
  INSTALL_DESTINATION ${ConfigPackageLocation}
)

install(TARGETS fmjpeg2k dcmcjp2k dcmdjp2k
  EXPORT fmjpeg2kTargets
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)

install(
  FILES
    ${fmjpeg2k_HEADERS}
  DESTINATION
    include/fmjpeg2k
  COMPONENT
    Devel
)

install(EXPORT fmjpeg2kTargets
  FILE
    fmjpeg2kTargets.cmake
  DESTINATION
    ${ConfigPackageLocation}
)

install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/fmjpeg2kConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/fmjpeg2kConfigVersion.cmake"
  DESTINATION
    ${ConfigPackageLocation}
  COMPONENT
    Devel
)

export(TARGETS fmjpeg2k
  FILE "${CMAKE_CURRENT_BINARY_DIR}/fmjpeg2kTargets.cmake"
)
