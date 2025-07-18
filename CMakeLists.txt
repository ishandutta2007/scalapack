cmake_minimum_required(VERSION 3.26...4.0)

project(SCALAPACK VERSION 2.2.2 LANGUAGES C Fortran)
set(CMAKE_C_STANDARD 17)
set(CMAKE_C_EXTENSIONS OFF)

# Add the CMake directory for custon CMake modules
set(CMAKE_MODULE_PATH "${SCALAPACK_SOURCE_DIR}/CMAKE" ${CMAKE_MODULE_PATH})

if (UNIX)
   if ("${CMAKE_Fortran_COMPILER_ID}" STREQUAL "Intel")
      set( CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fltconsistency -fp_port" )
   endif ()
endif ()

#
#   MPI
#
set(CMAKE_PREFIX_PATH "${MPI_BASE_DIR};${CMAKE_PREFIX_PATH}")

find_package(MPI)
if (MPI_FOUND)
   message(STATUS "Found MPI_LIBRARY : ${MPI_FOUND} ")

   find_program(MPI_C_COMPILER
      NAMES mpicc
      HINTS "${MPI_BASE_DIR}"
      PATH_SUFFIXES bin
      DOC "MPI C compiler.")
   MARK_AS_ADVANCED(MPI_C_COMPILER)
   if ("${MPI_C_COMPILER}" STREQUAL "MPI_C_COMPILER-NOTFOUND")
      message(ERROR "--> MPI C Compiler NOT FOUND (please set MPI_BASE_DIR accordingly")
   else()
      message(STATUS "--> MPI C Compiler : ${MPI_C_COMPILER}")
      #SET(CMAKE_C_COMPILER "${MPI_C_COMPILER}")
      message(STATUS "--> C Compiler : ${CMAKE_C_COMPILER}")
   endif()
   find_program(MPI_Fortran_COMPILER
      NAMES mpif77
      HINTS "${MPI_BASE_DIR}"
      PATH_SUFFIXES bin
      DOC "MPI Fortran compiler.")
   MARK_AS_ADVANCED(MPI_Fortran_COMPILER)

   if ("${MPI_Fortran_COMPILER}" STREQUAL "MPI_Fortran_COMPILER-NOTFOUND")
      message(ERROR "--> MPI Fortran Compiler NOT FOUND (please set MPI_BASE_DIR accordingly")
   else()
      message(STATUS "--> MPI Fortran Compiler : ${MPI_Fortran_COMPILER}")
      SET(Fortran_COMPILER "${CMAKE_Fortran_COMPILER}")
      #SET(CMAKE_Fortran_COMPILER "${MPI_Fortran_COMPILER}")
      message(STATUS "--> Fortran Compiler : ${CMAKE_Fortran_COMPILER}")
   endif()

else()
   message(STATUS "Found MPI_LIBRARY : ${MPI_FOUND} ")
   set(MPI_BASE_DIR ${MPI_BASE_DIR} CACHE PATH "MPI Path")
   UNSET(MPIEXEC CACHE)
   UNSET(MPIEXEC_POSTFLAGS CACHE)
   UNSET(MPIEXEC_PREFLAGS CACHE)
   UNSET(MPIEXEC_MAX_NUMPROCS CACHE)
   UNSET(MPIEXEC_NUMPROC_FLAG CACHE)
   UNSET(MPI_COMPILE_FLAGS CACHE)
   UNSET(MPI_LINK_FLAGS CACHE)
   UNSET(MPI_INCLUDE_PATH CACHE)
   message(FATAL_ERROR "--> MPI Library NOT FOUND -- please set MPI_BASE_DIR accordingly --")
endif()


if (UNIX)
   if ("${CMAKE_Fortran_COMPILER_ID}" STREQUAL "Intel")
      set( CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fltconsistency -fp_port" )
   endif ()
endif ()

macro(SCALAPACK_install_library lib)
  install(TARGETS ${lib} EXPORT scalapack-targets
    RUNTIME DESTINATION Testing
  )
endmacro()

# --------------------------------------------------
# Testing
SET(DART_TESTING_TIMEOUT 600)

enable_testing()
# --------------------------------------------------

# Organize output files.  On Windows this also keeps .dll files next
# to the .exe files that need them, making tests easy to run.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/TESTING)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/lib)

# --------------------------------------------------
# By default static library
function (invertBoolean varName varValue)
  if (${varValue})
    set(${varName} false PARENT_SCOPE)
  else ()
    set(${varName} true PARENT_SCOPE)
  endif ()
endfunction ()
option(BUILD_SHARED_LIBS "Build shared libraries" OFF )
invertBoolean("BUILD_STATIC_LIBS" ${BUILD_SHARED_LIBS})
if ((${BUILD_SHARED_LIBS} EQUAL ON) AND NOT CMAKE_POSITION_INDEPENDENT_CODE)
  set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif ()

# --------------------------------------------------
# Check for any necessary platform specific compiler flags
include( CheckBLACSCompilerFlags )
CheckBLACSCompilerFlags()

include(GNUInstallDirs)
set(PKG_CONFIG_DIR ${CMAKE_INSTALL_LIBDIR}/pkgconfig CACHE PATH "pkg-config install path")

# --------------------------------------------------
# BLACS Internal variables
#
#   Fortran Mangling, MPI Tests and BLACS settings
#
include(FortranMangling)

#MESSAGE(STATUS "Setting CDEFS = ${CDEFS}")
#set(CDEFS ${CDEFS} CACHE STRING "Fortran Mangling" FORCE)
MESSAGE(STATUS "=========")

# --------------------------------------------------
# Compiler Flags
ADD_DEFINITIONS( "-D${CDEFS}")

# --------------------------------------------------
# ScaLAPACK needs BLAS and LAPACK

option(USE_OPTIMIZED_LAPACK_BLAS "Whether or not to search for optimized LAPACK and BLAS libraries on your machine (if not found, Reference LAPACK and BLAS will be downloaded and installed)" ON)

message(STATUS "CHECKING BLAS AND LAPACK LIBRARIES")

IF(LAPACK_LIBRARIES)
  include(CheckFortranFunctionExists)
  message(STATUS "--> LAPACK supplied by user is ${LAPACK_LIBRARIES}.")
  set(CMAKE_REQUIRED_LIBRARIES ${LAPACK_LIBRARIES})
  CHECK_FORTRAN_FUNCTION_EXISTS("dgesv" LAPACK_FOUND)
  unset( CMAKE_REQUIRED_LIBRARIES )
  message(STATUS "--> LAPACK routine dgesv is found: ${LAPACK_FOUND}.")
ENDIF()

if(LAPACK_FOUND)
      message(STATUS "--> LAPACK supplied by user is WORKING, will use ${LAPACK_LIBRARIES}.")
else(LAPACK_FOUND)
   if(USE_OPTIMIZED_LAPACK_BLAS)
      message(STATUS "--> Searching for optimized LAPACK and BLAS libraries on your machine.")
      find_package(LAPACK)
   ENDIF(USE_OPTIMIZED_LAPACK_BLAS)
   if(NOT LAPACK_FOUND)
      message(STATUS "--> LAPACK and BLAS were not found. Reference LAPACK and BLAS will be downloaded and installed")
      include(ExternalProject)
      ExternalProject_Add(
        lapack
        URL http://www.netlib.org/lapack/lapack.tgz
        CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${SCALAPACK_BINARY_DIR}
        PREFIX ${SCALAPACK_BINARY_DIR}/dependencies
      )
      if (NOT MSVC)
         SET(LAPACK_LIBRARIES ${SCALAPACK_BINARY_DIR}/lib/liblapack.a CACHE STRING "LAPACK library" FORCE)
         SET(BLAS_LIBRARIES ${SCALAPACK_BINARY_DIR}/lib/libblas.a CACHE STRING "BLAS library" FORCE)
      else () # MSVC
         SET(LAPACK_LIBRARIES ${SCALAPACK_BINARY_DIR}/lib/liblapack.lib CACHE STRING "LAPACK library" FORCE)
         SET(BLAS_LIBRARIES ${SCALAPACK_BINARY_DIR}/lib/libblas.lib CACHE STRING "BLAS library" FORCE)
      endif ()
   ENDIF()
ENDIF()

message(STATUS "BLAS library: ${BLAS_LIBRARIES}")
message(STATUS "LAPACK library: ${LAPACK_LIBRARIES}")

MESSAGE(STATUS "=========")

# --------------------------------------------------
# Subdirectories that need to be processed

macro(append_subdir_files variable dirname)
get_directory_property(holder DIRECTORY ${dirname} DEFINITION ${variable})
foreach(depfile ${holder})
  list(APPEND ${variable} "${dirname}/${depfile}")
endforeach()
endmacro()


#
# BLACS
#
add_subdirectory(BLACS)
append_subdir_files(blacs "BLACS/SRC")

#
# TOOLS
#
add_subdirectory(TOOLS)
append_subdir_files(tools TOOLS)
append_subdir_files(tools-C TOOLS)
append_subdir_files(extra_lapack "TOOLS/LAPACK")

#
# PBLAS
#
add_subdirectory(PBLAS)
append_subdir_files(pblas "PBLAS/SRC")
append_subdir_files(pblas-F "PBLAS/SRC")
append_subdir_files(pbblas "PBLAS/SRC/PBBLAS")
append_subdir_files(ptzblas "PBLAS/SRC/PTZBLAS")
append_subdir_files(ptools "PBLAS/SRC/PTOOLS")

#
# REDIST
#
add_subdirectory(REDIST)
append_subdir_files(redist "REDIST/SRC")

#
# SRC
#
add_subdirectory(SRC)
append_subdir_files(src "SRC")
append_subdir_files(src-C "SRC")

if (NOT MSVC)
   set_source_files_properties(TOOLS/SL_gridreshape.c PROPERTIES COMPILE_OPTIONS "-Wno-error=implicit-function-declaration")
   add_library(scalapack ${blacs} ${tools} ${tools-C} ${extra_lapack} ${pblas} ${pblas-F} ${ptzblas} ${ptools} ${pbblas} ${redist} ${src} ${src-C})
   set_target_properties(scalapack PROPERTIES
    VERSION ${SCALAPACK_VERSION}
    SOVERSION ${SCALAPACK_VERSION_MAJOR}.${SCALAPACK_VERSION_MINOR})
   target_link_libraries( scalapack ${LAPACK_LIBRARIES} ${BLAS_LIBRARIES} MPI::MPI_C)
   scalapack_install_library(scalapack)
else () # Need to separate Fortran and C Code
   OPTION(BUILD_SHARED_LIBS "Build shared libraries" ON )
   add_library(scalapack ${blacs} ${tools-C} ${pblas} ${ptools} ${redist} ${src-C})
   set_target_properties(scalapack PROPERTIES
    VERSION ${SCALAPACK_VERSION}
    SOVERSION ${SCALAPACK_VERSION_MAJOR}.${SCALAPACK_VERSION_MINOR})
   target_link_libraries( scalapack ${LAPACK_LIBRARIES} ${BLAS_LIBRARIES} MPI::MPI_C)
   add_library(scalapack-F ${pblas-F} ${pbblas} ${ptzblas} ${tools} ${src} ${extra_lapack} )
   set_target_properties(scalapack-F PROPERTIES
    VERSION ${SCALAPACK_VERSION}
    SOVERSION ${SCALAPACK_VERSION_MAJOR}.${SCALAPACK_VERSION_MINOR})
   target_link_libraries( scalapack-F ${LAPACK_LIBRARIES} ${BLAS_LIBRARIES} MPI::MPI_Fortran)
   scalapack_install_library(scalapack)
   scalapack_install_library(scalapack-F)
endif ()

if(TARGET lapack)
   add_dependencies(scalapack lapack)
endif()

option(SCALAPACK_BUILD_TESTS "Build all tests of the ScaLAPACK library" ON)
if(${SCALAPACK_BUILD_TESTS})
  add_subdirectory(TESTING)
endif()

# --------------------------------------------------
# CPACK Packaging

SET(CPACK_PACKAGE_NAME "ScaLAPACK")
SET(CPACK_PACKAGE_VENDOR "University of Tennessee, Univ. of California Berkeley, Univ. of Colorado Denver and NAG Ltd")
SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "ScaLAPACK- Linear Algebra Package")
set(CPACK_PACKAGE_VERSION_MAJOR ${SCALAPACK_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${SCALAPACK_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${SCALAPACK_VERSION_PATCH})
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
SET(CPACK_PACKAGE_INSTALL_DIRECTORY "SCALAPACK")
IF(WIN32 AND NOT UNIX)
  # There is a bug in NSI that does not handle full unix paths properly. Make
  # sure there is at least one set of four (4) backlasshes.
  SET(CPACK_NSIS_HELP_LINK "http:\\\\\\\\http://icl.cs.utk.edu/lapack-forum")
  SET(CPACK_NSIS_URL_INFO_ABOUT "http:\\\\\\\\www.netlib.org/scalapack")
  SET(CPACK_NSIS_CONTACT "scalapack@eecs.utk.edu")
  SET(CPACK_NSIS_MODIFY_PATH ON)
  SET(CPACK_NSIS_DISPLAY_NAME "SCALAPACK-${SCALAPACK_VERSION}")
  set(CPACK_PACKAGE_RELOCATABLE "true")
ELSE(WIN32 AND NOT UNIX)
  SET(CPACK_GENERATOR "TGZ")
  SET(CPACK_SOURCE_GENERATOR TGZ)
  SET(CPACK_SOURCE_PACKAGE_FILE_NAME "scalapack-${SCALAPACK_VERSION}" )
  SET(CPACK_SOURCE_IGNORE_FILES ~$ .svn ${CPACK_SOURCE_IGNORE_FILES} )
ENDIF(WIN32 AND NOT UNIX)
INCLUDE(CPack)


# --------------------------------------------------


export(TARGETS scalapack FILE scalapack-targets.cmake)

if( NOT LAPACK_FOUND )
 install(FILES
  ${BLAS_LIBRARIES} ${LAPACK_LIBRARIES}
  DESTINATION ${CMAKE_INSTALL_LIBDIR}
   )
endif( NOT LAPACK_FOUND )

configure_file(${SCALAPACK_SOURCE_DIR}/CMAKE/scalapack-config-version.cmake.in
  ${SCALAPACK_BINARY_DIR}/scalapack-config-version.cmake @ONLY)
configure_file(${SCALAPACK_SOURCE_DIR}/CMAKE/scalapack-config-build.cmake.in
  ${SCALAPACK_BINARY_DIR}/scalapack-config.cmake @ONLY)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scalapack.pc.in ${CMAKE_CURRENT_BINARY_DIR}/scalapack.pc)
 install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/scalapack.pc
  DESTINATION ${PKG_CONFIG_DIR}
   )

configure_file(${SCALAPACK_SOURCE_DIR}/CMAKE/scalapack-config-install.cmake.in
  ${SCALAPACK_BINARY_DIR}/CMakeFiles/scalapack-config.cmake @ONLY)
install(FILES
  ${SCALAPACK_BINARY_DIR}/CMakeFiles/scalapack-config.cmake
  ${SCALAPACK_BINARY_DIR}/scalapack-config-version.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/scalapack-${SCALAPACK_VERSION}
  )

install(EXPORT scalapack-targets
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/scalapack-${SCALAPACK_VERSION})
