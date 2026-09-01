
SET(matc_SRCS c3d.c clip.c dri_ps.c eig.c error.c eval.c files.c
  funcs.c gra.c gra_com.c jacobi.c lists.c lu.c matc.c matrix.c oper.c
  optim.c parser.c str.c urand.c variable.c)

if (CMAKE_C_COMPILER_ID STREQUAL "GNU" AND
    CMAKE_C_COMPILER_VERSION VERSION_GREATER_EQUAL 16)
set_source_files_properties(str.c PROPERTIES COMPILE_OPTIONS "-O0")
endif()


# A self-contained executable for the release artifacts: the library is built
# static and, where the platform allows it, the runtime is linked in too, so
# the binary runs on a machine that has neither MATC nor a matching libc
# installed. Off by default, since the Elmer build wants the shared library.
OPTION(MATC_STATIC_CLI "Build a self-contained matc executable" OFF)

IF(MATC_STATIC_CLI)
  ADD_LIBRARY(matc STATIC ${matc_SRCS})
ELSE()
  ADD_LIBRARY(matc SHARED ${matc_SRCS})
ENDIF()

TARGET_LINK_LIBRARIES(matc m)

IF(WITH_OpenMP)
  target_compile_definitions(matc PUBLIC MATC_OPENMP)
ENDIF()

ADD_EXECUTABLE(Matc_bin main.c)
TARGET_LINK_LIBRARIES(Matc_bin matc)

IF(MATC_STATIC_CLI AND NOT APPLE)
  # Apple does not ship static system libraries, so -static cannot work there;
  # a macOS binary linking only libSystem is portable across macOS versions
  # anyway.
  SET_TARGET_PROPERTIES(Matc_bin PROPERTIES LINK_FLAGS "-static")
ENDIF()

# MATC is not buildable with MSVC: src/elmer/matc.h includes <unistd.h> and
# <sys/types.h> unconditionally, and MSVC has no unistd.h. Making it work
# would be a porting exercise rather than a build tweak, so say so plainly
# instead of failing later with a confusing missing-header error.
IF(MSVC)
  MESSAGE(FATAL_ERROR
    "MATC requires a POSIX-like C library and does not build with MSVC. "
    "On Windows use mingw-w64, for example through MSYS2.")
ENDIF()

SET_TARGET_PROPERTIES(Matc_bin PROPERTIES OUTPUT_NAME matc)
IF(WIN32)
  SET_TARGET_PROPERTIES(Matc_bin PROPERTIES ENABLE_EXPORTS OFF)
ELSE()
  SET_TARGET_PROPERTIES(Matc_bin PROPERTIES INSTALL_RPATH "${ELMERSOLVER_RPATH_STRING}")
ENDIF()

INSTALL(TARGETS matc Matc_bin
  RUNTIME DESTINATION "bin"
  LIBRARY DESTINATION ${ELMER_INSTALL_LIB_DIR}
  ARCHIVE DESTINATION ${ELMER_INSTALL_LIB_DIR})
