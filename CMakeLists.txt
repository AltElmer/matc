# MATC builds either as part of the Elmer tree, where the top level project
# supplies the install layout and RPATH, or on its own out of this directory.
# The block below runs only in the standalone case, so the in-tree build is
# unaffected.
#
# Standalone build:
#   cmake -S . -B build && cmake --build build
IF(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  CMAKE_MINIMUM_REQUIRED(VERSION 3.12)
  PROJECT(MATC C)

  IF(NOT CMAKE_BUILD_TYPE)
    SET(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type" FORCE)
  ENDIF()

  # The same standard the Elmer top level build selects. It matters here: the
  # callback type in com_init() is declared "VARIABLE *(*)()", meaning
  # unspecified arguments, and the functions passed to it genuinely have
  # different signatures. C23 redefines () as (void), so compilers defaulting
  # to it, such as GCC 15 and newer, reject every one of those calls with
  # "incompatible pointer type". Compiling as C99 keeps the meaning the code
  # was written against instead of suppressing the diagnostic.
  SET(CMAKE_C_STANDARD 99)
  SET(CMAKE_C_STANDARD_REQUIRED ON)

  # Normally provided by the Elmer top level build.
  IF(NOT DEFINED ELMER_INSTALL_LIB_DIR)
    SET(ELMER_INSTALL_LIB_DIR "lib")
  ENDIF()
  IF(NOT DEFINED ELMERSOLVER_RPATH_STRING)
    SET(ELMERSOLVER_RPATH_STRING "")
  ENDIF()

  INCLUDE(GNUInstallDirs OPTIONAL)
ENDIF()

INCLUDE(CheckIncludeFiles)

CHECK_INCLUDE_FILES(inttypes.h HAVE_INTTYPES_H)

CONFIGURE_FILE(config.h.cmake config.h)

# The sources include "../config.h", so the include directory has to be the
# src subdirectory of wherever config.h was written, not that directory itself.
# CMAKE_CURRENT_BINARY_DIR is the matc binary directory in both cases: the
# build root standalone, and <build>/matc in the Elmer tree. The previous
# spelling, "${PROJECT_BINARY_DIR}/matc/src", named the same place in-tree but
# a nonexistent one otherwise.
INCLUDE_DIRECTORIES("${CMAKE_CURRENT_BINARY_DIR}/src")
ADD_SUBDIRECTORY(src)
