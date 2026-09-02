
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
  # A Windows DLL exports nothing unless the sources say so, and these carry no
  # __declspec(dllexport). Without exports no import library is produced and
  # linking the executable fails with LNK1181 on matc.lib. Export everything,
  # which is what the ELF and Mach-O builds already do by default. Ignored on
  # platforms where it does not apply.
  SET_TARGET_PROPERTIES(matc PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
ENDIF()

# Link libm only where it is a separate library. It is on glibc and the BSDs,
# it is not on MSVC or on macOS, where the math functions live in the C
# runtime, and linking a nonexistent "m" fails with LNK1181 there. Detect it
# rather than testing the compiler name, so a toolchain nobody here has tried
# still gets the right answer.
INCLUDE(CheckLibraryExists)
CHECK_LIBRARY_EXISTS(m sin "" MATC_HAVE_LIBM)
IF(MATC_HAVE_LIBM)
  TARGET_LINK_LIBRARIES(matc m)
ENDIF()

IF(WITH_OpenMP)
  target_compile_definitions(matc PUBLIC MATC_OPENMP)
ENDIF()

ADD_EXECUTABLE(Matc_bin main.c)
TARGET_LINK_LIBRARIES(Matc_bin matc)

# C99 as a property of the targets, not of the standalone build.
#
# com_init() takes a callback declared "VARIABLE *(*)()", meaning unspecified
# arguments, and the functions handed to it genuinely have different
# signatures. C23 redefines () as (void), so a compiler defaulting to it, such
# as GCC 15, rejects every one of those calls as an incompatible pointer type.
#
# This used to be set as CMAKE_C_STANDARD inside the standalone block, which
# meant it did not apply when MATC was added as a subdirectory: a consumer
# inherited the parent's C standard and hit the exact failure the setting
# exists to prevent. Found by consuming this repository as a submodule from
# ElmerFront. Set on the targets so it travels with them however MATC is built.
SET_TARGET_PROPERTIES(matc Matc_bin PROPERTIES
  C_STANDARD 99
  C_STANDARD_REQUIRED ON)

IF(MATC_STATIC_CLI)
  IF(MSVC)
    # MSVC has no -static; the equivalent is linking the CRT statically.
    SET_PROPERTY(TARGET matc PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded")
    SET_PROPERTY(TARGET Matc_bin PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded")
  ELSEIF(NOT APPLE)
    # Apple ships no static system libraries, so -static cannot work there. A
    # macOS binary linking only libSystem is portable across macOS versions
    # anyway, so there is nothing to do for that case.
    SET_TARGET_PROPERTIES(Matc_bin PROPERTIES LINK_FLAGS "-static")
  ENDIF()
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
