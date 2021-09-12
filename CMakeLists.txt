cmake_minimum_required(VERSION 3.10)
project(
  PVFMM
  VERSION 1.2
  LANGUAGES CXX)

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_EXTENSIONS OFF)
include(GNUInstallDirs)

# compiler flags
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  # using Clang
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  # using GCC
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
  # using intel
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -qno-offload")
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  # using Visual Studio C++
endif()

# Duplicate the MakeVariables.in flags introduced in b54ef24f7f0d6cd36c3e5ccba06fbbdb11ad2073.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSCTL_HAVE_BLAS")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSCTL_HAVE_FFTW")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSCTL_HAVE_LAPACK")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSCTL_QUAD_T=__float128")

add_library(sctl_includes INTERFACE)
target_include_directories(sctl_includes INTERFACE SCTL/include/)

set(MPI_CXX_SKIP_MPICXX
    true
    CACHE BOOL
          "If true, the MPI-2 C++ bindings are disabled using definitions.")
          
# required compiler features
find_package(MPI REQUIRED)

# library
find_package(MKL CONFIG)
if(MKL_FOUND)
  set(DEP_INC ${MKL_INCLUDE} ${MKL_INCLUDE}/fftw)
  set(DEP_LIB MKL::MKL)
else()
  find_package(OpenMP REQUIRED)
  find_package(BLAS REQUIRED)
  find_package(LAPACK REQUIRED)
  find_package(FFTW REQUIRED)
  set(DEP_INC ${FFTW_INCLUDE_DIRS})
  set(DEP_LIB OpenMP::OpenMP_CXX
              ${FFTW_FLOAT_OPENMP_LIB} ${FFTW_FLOAT_LIB}
              ${FFTW_DOUBLE_OPENMP_LIB} ${FFTW_DOUBLE_LIB} 
              ${BLAS_LIBRARIES}
              ${LAPACK_LIBRARIES})
endif()

find_file(SYS_STAT sys/stat.h OPTIONAL)

# generate pvfmm_config.h

# switch for periodic boundary condition
option(PVFMM_EXTENDED_BC "PVFMM_EXTENDED_BC" ON)

if(MKL_FOUND)
  option(PVFMM_FFTW3_MKL "PVFMM_FFTW3_MKL" ON)
  option(PVFMM_HAVE_FFTW "PVFMM_HAVE_FFTW" ON)
  option(PVFMM_HAVE_FFTWF "PVFMM_HAVE_FFTWF" ON)
endif()

if(FFTW_DOUBLE_LIB_FOUND)
  option(PVFMM_HAVE_FFTW "PVFMM_HAVE_FFTW" ON)
endif()

if(FFTW_FLOAT_LIB_FOUND)
  option(PVFMM_HAVE_FFTWF "PVFMM_HAVE_FFTWF" ON)
endif()

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
  option(PVFMM_HAVE_INTEL_SVML "PVFMM_HAVE_INTEL_SVML" ON)
endif()

if(SYS_STAT)
  option(PVFMM_HAVE_SYS_STAT_H "PVFMM_HAVE_SYS_STAT_H" ON)
endif()

configure_file(pvfmm_config.h.in pvfmm_config.h @ONLY)

# part 1, core library
file(GLOB pvfmm_SRC "src/*.cpp")

# shared lib
add_library(pvfmm SHARED ${pvfmm_SRC})
set_target_properties(pvfmm PROPERTIES OUTPUT_NAME pvfmm)
target_include_directories(
  pvfmm
  PUBLIC $<INSTALL_INTERFACE:include>
         $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/>
         ${CMAKE_BINARY_DIR} ${DEP_INC})
target_link_libraries(pvfmm PUBLIC ${DEP_LIB} MPI::MPI_CXX sctl_includes)

# static lib
add_library(pvfmmStatic STATIC ${pvfmm_SRC})
set_target_properties(pvfmmStatic PROPERTIES OUTPUT_NAME pvfmm)
target_include_directories(
  pvfmmStatic
  PUBLIC $<INSTALL_INTERFACE:include>
         $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/>
         ${CMAKE_BINARY_DIR} ${DEP_INC})
target_link_libraries(pvfmmStatic PUBLIC ${DEP_LIB} MPI::MPI_CXX sctl_includes)

# Config.cmake files
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  ${CMAKE_BINARY_DIR}/pvfmmConfigVersion.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion)

get_target_property(PVFMM_SHARED_NAME pvfmm OUTPUT_NAME)
get_target_property(PVFMM_STATIC_NAME pvfmmStatic OUTPUT_NAME)

configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/pvfmmConfig.cmake.in
  ${CMAKE_BINARY_DIR}/pvfmmConfig.cmake INSTALL_DESTINATION
  ${CMAKE_INSTALL_PREFIX}/share/pvfmm)

install(FILES ${CMAKE_BINARY_DIR}/pvfmmConfig.cmake
              ${CMAKE_BINARY_DIR}/pvfmmConfigVersion.cmake
        DESTINATION ${CMAKE_INSTALL_PREFIX}/share/pvfmm)

# install core library and headers
include(GNUInstallDirs)
install(
  TARGETS pvfmm 
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(
  TARGETS pvfmmStatic
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/pvfmm/)
install(FILES ${CMAKE_BINARY_DIR}/pvfmm_config.h
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/pvfmm/)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/SCTL/include/
	DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/SCTL/)

add_subdirectory(examples)
