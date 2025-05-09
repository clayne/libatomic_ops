#
# Copyright (c) 2021-2022 Ivan Maidanski
##
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
##

cmake_minimum_required(VERSION 3.10)

set(PACKAGE_VERSION 7.9.0)
# Version must match that in AC_INIT of configure.ac and that in README.
# Version must conform to: [0-9]+[.][0-9]+[.][0-9]+

# Info (current:revision:age) for the Libtool versioning system.
# These values should match those in src/Makefile.am.
set(LIBATOMIC_OPS_VER_INFO      4:0:3)
set(LIBATOMIC_OPS_GPL_VER_INFO  4:0:3)

project(libatomic_ops C)

if (POLICY CMP0057)
  # Required for CheckLinkerFlag, at least.
  cmake_policy(SET CMP0057 NEW)
endif()

include(CheckCCompilerFlag)
include(CheckFunctionExists)
include(CMakePackageConfigHelpers)
include(CTest)
include(GNUInstallDirs)

if (NOT (${CMAKE_VERSION} VERSION_LESS "3.18.0"))
  include(CheckLinkerFlag)
endif()

# Customize the build by passing "-D<option_name>=ON|OFF" in the command line.
option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(build_tests "Build tests" OFF)
option(enable_assertions "Enable assertion checking" OFF)
option(enable_werror "Treat warnings as errors" OFF)
option(enable_atomic_intrinsics "Use GCC atomic intrinsics" ON)
option(enable_docs "Build and install documentation" ON)
option(enable_gpl "Build atomic_ops_gpl library" ON)
option(install_headers "Install header and pkg-config metadata files" ON)

# Override the default build type to RelWithDebInfo (this instructs cmake to
# pass -O2 -g -DNDEBUG options to the compiler).
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE
      STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
               STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")
endif()

# Convert VER_INFO values to [SO]VERSION ones.
if (BUILD_SHARED_LIBS)
  # atomic_ops:
  string(REGEX REPLACE "(.+):.+:.+"  "\\1" ao_cur ${LIBATOMIC_OPS_VER_INFO})
  string(REGEX REPLACE ".+:(.+):.+"  "\\1" ao_rev ${LIBATOMIC_OPS_VER_INFO})
  string(REGEX REPLACE ".+:.+:(.+)$" "\\1" ao_age ${LIBATOMIC_OPS_VER_INFO})
  math(EXPR AO_SOVERSION "${ao_cur} - ${ao_age}")
  set(AO_VERSION_PROP "${AO_SOVERSION}.${ao_age}.${ao_rev}")
  message(STATUS "AO_VERSION_PROP = ${AO_VERSION_PROP}")
  # atomic_ops_gpl:
  string(REGEX REPLACE "(.+):.+:.+"  "\\1" ao_gpl_cur
         ${LIBATOMIC_OPS_GPL_VER_INFO})
  string(REGEX REPLACE ".+:(.+):.+"  "\\1" ao_gpl_rev
         ${LIBATOMIC_OPS_GPL_VER_INFO})
  string(REGEX REPLACE ".+:.+:(.+)$" "\\1" ao_gpl_age
         ${LIBATOMIC_OPS_GPL_VER_INFO})
  math(EXPR AO_GPL_SOVERSION "${ao_gpl_cur} - ${ao_gpl_age}")
  set(AO_GPL_VERSION_PROP "${AO_GPL_SOVERSION}.${ao_gpl_age}.${ao_gpl_rev}")
  message(STATUS "AO_GPL_VERSION_PROP = ${AO_GPL_VERSION_PROP}")
endif(BUILD_SHARED_LIBS)

# Output all warnings.
if (MSVC)
  # All warnings but ignoring "conditional expression is constant" ones.
  add_compile_options(/W4 /wd4127)
else()
  # TODO: add -[W]pedantic -Wno-long-long
  add_compile_options(-Wall -Wextra)
endif()

find_package(Threads REQUIRED)
message(STATUS "Thread library: ${CMAKE_THREAD_LIBS_INIT}")
include_directories(${Threads_INCLUDE_DIR})
set(THREADDLLIBS_LIST ${CMAKE_THREAD_LIBS_INIT})

if (CMAKE_USE_PTHREADS_INIT)
  # Required define if using POSIX threads.
  add_compile_options(-D_REENTRANT)
else()
  # No pthreads library available.
  add_compile_options(-DAO_NO_PTHREADS)
endif()

if (enable_assertions)
  # In case NDEBUG macro is defined e.g. by cmake -DCMAKE_BUILD_TYPE=Release.
  add_compile_options(-UNDEBUG)
else()
  # Define to disable assertion checking.
  add_compile_options(-DNDEBUG)
endif()

if (NOT enable_atomic_intrinsics)
  # Define to avoid GCC atomic intrinsics even if available.
  add_compile_options(-DAO_DISABLE_GCC_ATOMICS)
endif()

# AO API symbols export control.
if (BUILD_SHARED_LIBS)
  add_compile_options(-DAO_DLL)
endif()

if (enable_werror)
  if (MSVC)
    add_compile_options(/WX)
  else()
    add_compile_options(-Werror)
  endif()
endif(enable_werror)

# Extra user-defined flags to pass to the C compiler.
if (DEFINED CFLAGS_EXTRA)
  separate_arguments(CFLAGS_EXTRA_LIST UNIX_COMMAND "${CFLAGS_EXTRA}")
  add_compile_options(${CFLAGS_EXTRA_LIST})
endif()

set(SRC src/atomic_ops.c)

if (CMAKE_C_COMPILER_ID STREQUAL "SunPro")
  # SunCC compiler on SunOS (Solaris).
  enable_language(ASM)
  set(SRC ${SRC} src/atomic_ops_sysdeps.S)
endif()

add_library(atomic_ops ${SRC})
target_link_libraries(atomic_ops PRIVATE ${THREADDLLIBS_LIST})
target_include_directories(atomic_ops
                PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>"
                INTERFACE "$<INSTALL_INTERFACE:include>")

if (enable_gpl)
  set(AO_GPL_SRC src/atomic_ops_malloc.c src/atomic_ops_stack.c)
  add_library(atomic_ops_gpl ${AO_GPL_SRC})
  check_function_exists(mmap HAVE_MMAP)
  if (HAVE_MMAP)
    target_compile_definitions(atomic_ops_gpl PRIVATE HAVE_MMAP)
  endif()
  target_link_libraries(atomic_ops_gpl PRIVATE atomic_ops)
  target_include_directories(atomic_ops_gpl
                PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>"
                INTERFACE "$<INSTALL_INTERFACE:include>")
  if (BUILD_SHARED_LIBS)
    set_property(TARGET atomic_ops_gpl PROPERTY VERSION ${AO_GPL_VERSION_PROP})
    set_property(TARGET atomic_ops_gpl PROPERTY SOVERSION ${AO_GPL_SOVERSION})
  endif()
  install(TARGETS atomic_ops_gpl EXPORT Atomic_opsTargets
          LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
          ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
          RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
          INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
endif(enable_gpl)

if (BUILD_SHARED_LIBS)
  if (NOT (${CMAKE_SYSTEM_NAME} MATCHES "BSD"))
    if (${CMAKE_VERSION} VERSION_LESS "3.18.0")
      set(WL_NO_UNDEFINED_OPT "-Wl,--no-undefined")
      check_c_compiler_flag(${WL_NO_UNDEFINED_OPT} HAVE_FLAG_WL_NO_UNDEFINED)
    else()
      set(WL_NO_UNDEFINED_OPT "LINKER:--no-undefined")
      check_linker_flag(C "${WL_NO_UNDEFINED_OPT}" HAVE_FLAG_WL_NO_UNDEFINED)
    endif()
    if (HAVE_FLAG_WL_NO_UNDEFINED)
      # Declare that the libraries do not refer to external symbols.
      if (${CMAKE_VERSION} VERSION_LESS "3.13.0")
        target_link_libraries(atomic_ops PRIVATE ${WL_NO_UNDEFINED_OPT})
        if (enable_gpl)
          target_link_libraries(atomic_ops_gpl PRIVATE ${WL_NO_UNDEFINED_OPT})
        endif(enable_gpl)
      else()
        target_link_options(atomic_ops PRIVATE ${WL_NO_UNDEFINED_OPT})
        if (enable_gpl)
          target_link_options(atomic_ops_gpl PRIVATE ${WL_NO_UNDEFINED_OPT})
        endif(enable_gpl)
      endif()
    endif(HAVE_FLAG_WL_NO_UNDEFINED)
  endif()
  set_property(TARGET atomic_ops PROPERTY VERSION ${AO_VERSION_PROP})
  set_property(TARGET atomic_ops PROPERTY SOVERSION ${AO_SOVERSION})
endif(BUILD_SHARED_LIBS)

install(TARGETS atomic_ops EXPORT Atomic_opsTargets
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
        INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")

if (install_headers)
  install(FILES src/atomic_ops.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  if (enable_gpl)
    install(FILES src/atomic_ops_malloc.h
                  src/atomic_ops_stack.h
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
  endif()

  install(FILES src/atomic_ops/ao_version.h
                src/atomic_ops/generalize-arithm.h
                src/atomic_ops/generalize-small.h
                src/atomic_ops/generalize.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops")

  install(FILES src/atomic_ops/sysdeps/all_acquire_release_volatile.h
                src/atomic_ops/sysdeps/all_aligned_atomic_load_store.h
                src/atomic_ops/sysdeps/all_atomic_load_store.h
                src/atomic_ops/sysdeps/all_atomic_only_load.h
                src/atomic_ops/sysdeps/ao_t_is_int.h
                src/atomic_ops/sysdeps/emul_cas.h
                src/atomic_ops/sysdeps/generic_pthread.h
                src/atomic_ops/sysdeps/ordered.h
                src/atomic_ops/sysdeps/ordered_except_wr.h
                src/atomic_ops/sysdeps/read_ordered.h
                src/atomic_ops/sysdeps/standard_ao_double_t.h
                src/atomic_ops/sysdeps/test_and_set_t_is_ao_t.h
                src/atomic_ops/sysdeps/test_and_set_t_is_char.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps")

  install(FILES src/atomic_ops/sysdeps/armcc/arm_v6.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps/armcc")
  install(FILES src/atomic_ops/sysdeps/gcc/aarch64.h
                src/atomic_ops/sysdeps/gcc/alpha.h
                src/atomic_ops/sysdeps/gcc/arm.h
                src/atomic_ops/sysdeps/gcc/avr32.h
                src/atomic_ops/sysdeps/gcc/cris.h
                src/atomic_ops/sysdeps/gcc/e2k.h
                src/atomic_ops/sysdeps/gcc/generic-arithm.h
                src/atomic_ops/sysdeps/gcc/generic-small.h
                src/atomic_ops/sysdeps/gcc/generic.h
                src/atomic_ops/sysdeps/gcc/hexagon.h
                src/atomic_ops/sysdeps/gcc/hppa.h
                src/atomic_ops/sysdeps/gcc/ia64.h
                src/atomic_ops/sysdeps/gcc/m68k.h
                src/atomic_ops/sysdeps/gcc/mips.h
                src/atomic_ops/sysdeps/gcc/powerpc.h
                src/atomic_ops/sysdeps/gcc/riscv.h
                src/atomic_ops/sysdeps/gcc/s390.h
                src/atomic_ops/sysdeps/gcc/sh.h
                src/atomic_ops/sysdeps/gcc/sparc.h
                src/atomic_ops/sysdeps/gcc/tile.h
                src/atomic_ops/sysdeps/gcc/x86.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps/gcc")

  install(FILES src/atomic_ops/sysdeps/hpc/hppa.h
                src/atomic_ops/sysdeps/hpc/ia64.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps/hpc")
  install(FILES src/atomic_ops/sysdeps/ibmc/powerpc.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps/ibmc")
  install(FILES src/atomic_ops/sysdeps/icc/ia64.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps/icc")

  install(FILES src/atomic_ops/sysdeps/loadstore/acquire_release_volatile.h
            src/atomic_ops/sysdeps/loadstore/atomic_load.h
            src/atomic_ops/sysdeps/loadstore/atomic_store.h
            src/atomic_ops/sysdeps/loadstore/char_acquire_release_volatile.h
            src/atomic_ops/sysdeps/loadstore/char_atomic_load.h
            src/atomic_ops/sysdeps/loadstore/char_atomic_store.h
            src/atomic_ops/sysdeps/loadstore/double_atomic_load_store.h
            src/atomic_ops/sysdeps/loadstore/int_acquire_release_volatile.h
            src/atomic_ops/sysdeps/loadstore/int_atomic_load.h
            src/atomic_ops/sysdeps/loadstore/int_atomic_store.h
            src/atomic_ops/sysdeps/loadstore/ordered_loads_only.h
            src/atomic_ops/sysdeps/loadstore/ordered_stores_only.h
            src/atomic_ops/sysdeps/loadstore/short_acquire_release_volatile.h
            src/atomic_ops/sysdeps/loadstore/short_atomic_load.h
            src/atomic_ops/sysdeps/loadstore/short_atomic_store.h
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps/loadstore")

  install(FILES src/atomic_ops/sysdeps/msftc/arm.h
                src/atomic_ops/sysdeps/msftc/arm64.h
                src/atomic_ops/sysdeps/msftc/common32_defs.h
                src/atomic_ops/sysdeps/msftc/x86.h
                src/atomic_ops/sysdeps/msftc/x86_64.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps/msftc")
  install(FILES src/atomic_ops/sysdeps/sunc/sparc.h
                src/atomic_ops/sysdeps/sunc/x86.h
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/atomic_ops/sysdeps/sunc")

  # Provide pkg-config metadata.
  set(prefix "${CMAKE_INSTALL_PREFIX}")
  set(exec_prefix \${prefix})
  set(includedir "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
  set(libdir "${CMAKE_INSTALL_FULL_LIBDIR}")
  string(REPLACE ";" " " THREADDLLIBS "${THREADDLLIBS_LIST}")
  # PACKAGE_VERSION is defined above.
  configure_file(pkgconfig/atomic_ops.pc.in pkgconfig/atomic_ops.pc @ONLY)
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/pkgconfig/atomic_ops.pc"
          DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
  # TODO: handle atomic_ops-uninstalled.pc.in
endif(install_headers)

if (build_tests)
  add_executable(test_atomic tests/test_atomic.c)
  target_link_libraries(test_atomic PRIVATE atomic_ops ${THREADDLLIBS_LIST})
  add_test(NAME test_atomic COMMAND test_atomic)

  add_executable(test_atomic_generalized tests/test_atomic.c)
  target_compile_definitions(test_atomic_generalized
                             PRIVATE AO_PREFER_GENERALIZED AO_TEST_EMULATION)
  target_link_libraries(test_atomic_generalized
                        PRIVATE atomic_ops ${THREADDLLIBS_LIST})
  add_test(NAME test_atomic_generalized COMMAND test_atomic_generalized)

  if (CMAKE_USE_PTHREADS_INIT)
    add_executable(test_atomic_pthreads tests/test_atomic.c)
    target_compile_definitions(test_atomic_pthreads
                               PRIVATE AO_USE_PTHREAD_DEFS)
    target_link_libraries(test_atomic_pthreads
                          PRIVATE atomic_ops ${THREADDLLIBS_LIST})
    add_test(NAME test_atomic_pthreads COMMAND test_atomic_pthreads)
  endif()

  if (enable_gpl)
    add_executable(test_stack tests/test_stack.c)
    target_link_libraries(test_stack
                PRIVATE atomic_ops atomic_ops_gpl ${THREADDLLIBS_LIST})
    add_test(NAME test_stack COMMAND test_stack)

    add_executable(test_malloc tests/test_malloc.c)
    target_link_libraries(test_malloc
                PRIVATE atomic_ops atomic_ops_gpl ${THREADDLLIBS_LIST})
    add_test(NAME test_malloc COMMAND test_malloc)
  endif()
endif(build_tests)

if (enable_docs)
  install(FILES AUTHORS ChangeLog LICENSE README.md
                README_details.txt README_win32.txt
          DESTINATION "${CMAKE_INSTALL_DOCDIR}")
  if (enable_gpl)
    install(FILES COPYING README_malloc.txt README_stack.txt
            DESTINATION "${CMAKE_INSTALL_DOCDIR}")
  endif()
endif(enable_docs)

# CMake config/targets files.
install(EXPORT Atomic_opsTargets FILE Atomic_opsTargets.cmake
        NAMESPACE Atomic_ops::
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/atomic_ops")

configure_package_config_file("${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/Atomic_opsConfig.cmake"
        INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/atomic_ops"
        NO_SET_AND_CHECK_MACRO)

write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/Atomic_opsConfigVersion.cmake"
        VERSION "${PACKAGE_VERSION}" COMPATIBILITY AnyNewerVersion)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/Atomic_opsConfig.cmake"
              "${CMAKE_CURRENT_BINARY_DIR}/Atomic_opsConfigVersion.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/atomic_ops")

export(EXPORT Atomic_opsTargets
       FILE "${CMAKE_CURRENT_BINARY_DIR}/Atomic_opsTargets.cmake")
