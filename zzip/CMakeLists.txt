cmake_minimum_required (VERSION 3.1)
project(zzip VERSION "0.13.72" LANGUAGES C)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/CMakeScripts")

# generate _config.h
include ( CheckIncludeFiles )
include ( CheckFunctionExists )
include ( CheckSymbolExists )
include ( CheckTypeSize )
include ( TestBigEndian )
include ( GNUInstallDirs )
include ( JoinPaths )
include ( CheckCompilerFlag )

# options ###########################################################
option(BUILD_SHARED_LIBS "Build a shared library" ON)
option(MSVC_STATIC_RUNTIME "Build with static runtime libs (/MT)" ON)
option(ZZIPMMAPPED "Build libzzipmmapped (not fully portable)" ON)
option(ZZIPFSEEKO "Build libzzipfseeko (based on posix.1 api)" ON)

if(UNIX OR MINGW)
option(ZZIP_COMPAT "Build compatibility with old libzzip releases" ON)
option(ZZIP_LIBTOOL "Ensure binary compatibility with libtool" ON)
option(ZZIP_PKGCONFIG "Generate pkg-config files for linking" ON)
else()
option(ZZIP_COMPAT "Build compatibility with old libzzip releases" OFF)
option(ZZIP_LIBTOOL "Ensure binary compatibility with libtool" OFF)
option(ZZIP_PKGCONFIG "Generate pkg-config files for linking" OFF)
endif()

if(ZZIP_LIBTOOL OR ZZIP_PKGCONFIG)
option(ZZIP_LIBLATEST "Ensure libname.lib links to libname-REL.lib" ON)
else()
option(ZZIP_LIBLATEST "Ensure libname.lib links to libname-REL.lib" OFF)
endif()

# used in zzip/_config.h
set(ZZIP_PACKAGE "${PROJECT_NAME}lib")
set(ZZIP_VERSION "${PROJECT_VERSION}")
set(ZZIP_PACKAGE_NAME "${PROJECT_NAME}lib")
set(ZZIP_PACKAGE_VERSION "${PROJECT_VERSION}")

if(ZZIP_COMPAT OR ZZIP_PKGCONFIG)
find_package ( UnixCommands REQUIRED ) # bash cp mv rm gzip tar
endif()

check_include_files ( byteswap.h ZZIP_HAVE_BYTESWAP_H )
check_include_files ( direct.h ZZIP_HAVE_DIRECT_H )

if(NOT ANDROID)
    check_include_files ( dirent.h ZZIP_HAVE_DIRENT_H )
endif()

check_include_files ( dlfcn.h ZZIP_HAVE_DLFCN_H )
check_include_files ( fnmatch.h ZZIP_HAVE_FNMATCH_H )
check_include_files ( inttypes.h ZZIP_HAVE_INTTYPES_H )
check_include_files ( io.h ZZIP_HAVE_IO_H )
check_include_files ( memory.h ZZIP_HAVE_MEMORY_H )
check_include_files ( ndir.h ZZIP_HAVE_NDIR_H )
check_include_files ( stdint.h ZZIP_HAVE_STDINT_H )
check_include_files ( stdlib.h ZZIP_HAVE_STDLIB_H )
check_function_exists ( strcasecmp ZZIP_HAVE_STRCASECMP )
check_include_files ( strings.h ZZIP_HAVE_STRINGS_H )
check_include_files ( string.h ZZIP_HAVE_STRING_H )
check_function_exists ( strndup ZZIP_HAVE_STRNDUP )
check_include_files ( "sys/dir.h" ZZIP_HAVE_SYS_DIR_H )
check_include_files ( "sys/int_types.h" ZZIP_HAVE_SYS_INT_TYPES_H )
check_include_files ( "sys/mman.h" ZZIP_HAVE_SYS_MMAN_H )
check_include_files ( "sys/ndir.h" ZZIP_HAVE_SYS_NDIR_H )
check_include_files ( "sys/param.h" ZZIP_HAVE_SYS_PARAM_H )
check_include_files ( "sys/stat.h" ZZIP_HAVE_SYS_STAT_H )
check_include_files ( "sys/types.h" ZZIP_HAVE_SYS_TYPES_H )
check_include_files ( unistd.h ZZIP_HAVE_UNISTD_H )
check_function_exists ( pread ZZIP_HAVE_PREAD )
check_include_files ( winbase.h ZZIP_HAVE_WINBASE_H )
check_include_files ( windows.h ZZIP_HAVE_WINDOWS_H )
check_include_files ( winnt.h ZZIP_HAVE_WINNT_H )
set ( ZZIP_HAVE_ZLIB_H 1 )

check_type_size ( int ZZIP_SIZEOF_INT )
check_type_size ( long ZZIP_SIZEOF_LONG )
check_type_size ( short ZZIP_SIZEOF_SHORT )
check_include_files ( "stdlib.h;stdarg.h;string.h;float.h" ZZIP_STDC_HEADERS )
test_big_endian ( ZZIP_WORDS_BIGENDIAN )

# simplified largefile support detection for unix
set(CMAKE_REQUIRED_DEFINITIONS -D_LARGEFILE_SOURCE -Doff_t_32=off_t)
check_type_size( off_t_32 ZZIP_SIZEOF_OFF_T_32 )
set(CMAKE_REQUIRED_DEFINITIONS -D_LARGEFILE_SOURCE -Doff_t_64=off_t -D_FILE_OFFSET_BITS=64 -D_LARGE_FILES)
check_type_size( off_t_64 ZZIP_SIZEOF_OFF_T_64 )
set(CMAKE_REQUIRED_DEFINITIONS -D_LARGEFILE64_SOURCE)
check_type_size( off64_t ZZIP_SIZEOF_OFF64_T )

if(ZZIP_SIZEOF_OFF_T_32)
  message(STATUS "found sizeof(off_t/32)=${ZZIP_SIZEOF_OFF_T_32} and sizeof(off_t/64)=${ZZIP_SIZEOF_OFF_T_64}")
  if(NOT ZZIP_SIZEOF_OFF_T_32 EQUAL ZZIP_SIZEOF_OFF_T_64)
    set(ZZIP_LARGEFILE_SENSITIVE 1)
    # this will turn into ZZIP_LARGEFILE_RENAME so that zzip_opendir becomes zzip_opendir64 in the lib
    # which is needed on Linux/Solaris where the default zzip_opendir is expected to use a 32bit off_t
    message(STATUS "system is large file sensitive - do rename symbols to xx64")
    set(largefiles -D_LARGEFILE_SOURCE -D_LARGEFILE64_SOURCE -D_FILE_OFFSET_BITS=64 -D_LARGE_FILES)
  endif()
endif()

configure_file ( ${CMAKE_CURRENT_SOURCE_DIR}/_config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/_config.h )

# Zlib library needed
find_package ( ZLIB REQUIRED )

if(UNIX)
CHECK_COMPILER_FLAG(C "-Warray-bounds" HAS_WARRAY_BOUNDS)
    add_definitions(
        -Wpointer-arith
        -Wsign-compare
        -Wmissing-declarations
        # -Wdeclaration-after-statement
        -Werror-implicit-function-declaration
        -Wstrict-aliasing
    )
  if(HAS_WARRAY_BOUNDS)
    add_definitions(
        -Warray-bounds
    )
  endif()
endif()

set(CMAKE_POSITION_INDEPENDENT_CODE ${BUILD_SHARED_LIBS})
add_definitions ( -DHAVE_CONFIG_H )

set(RELNUM "${PROJECT_VERSION_MAJOR}")
set(VERNUM "${PROJECT_VERSION_MINOR}")
set(FIXNUM "0.${PROJECT_VERSION_PATCH}")
# the "0." on FIXNUM is just for compatibility with automake's libtool.

set(srcdir ${CMAKE_CURRENT_SOURCE_DIR})
set(outdir ${CMAKE_CURRENT_BINARY_DIR})

# targets ############################################################

# libzlib library
set(libzzip_SRCS 
    dir.c
    err.c 
    file.c 
    info.c 
    plugin.c 
    stat.c
    write.c 
    zip.c 
    fetch.c)

set(libzzip_HDRS
    lib.h
    zzip.h
    format.h
    types.h
    conf.h 
    _msvc.h 
    file.h
    info.h
    plugin.h
    write.h
    fetch.h
    stdint.h
    zzip32.h
    autoconf.h
    ${outdir}/_config.h)

set(libzzip_INCS
    __debug.h
    __dirent.h
    __errno.h
    __fnmatch.h
    __hints.h
    __mkdir.h
    __mmap.h
    __string.h)

set(libzzipfseeko_SRCS fseeko.c fetch.c)
set(libzzipfseeko_HDRS fseeko.h)

set(libzzipmmapped_SRCS mmapped.c memdisk.c fetch.c)
set(libzzipmmapped_HDRS mmapped.h memdisk.h)


add_library(libzzip ${libzzip_SRCS} )
target_link_libraries(libzzip ZLIB::ZLIB )
target_include_directories (libzzip PRIVATE ${CMAKE_SOURCE_DIR} ${CMAKE_BINARY_DIR} PUBLIC $<INSTALL_INTERFACE:include/zzip>)

if(ZZIPFSEEKO)
add_library(libzzipfseeko ${libzzipfseeko_SRCS} )
target_link_libraries(libzzipfseeko ZLIB::ZLIB )
target_include_directories (libzzipfseeko PRIVATE ${CMAKE_SOURCE_DIR} ${CMAKE_BINARY_DIR})
endif()

if(ZZIPMMAPPED)
add_library(libzzipmmapped ${libzzipmmapped_SRCS} )
target_link_libraries(libzzipmmapped ZLIB::ZLIB )
target_include_directories (libzzipmmapped PRIVATE ${CMAKE_SOURCE_DIR} ${CMAKE_BINARY_DIR})
endif()

set_target_properties(libzzip PROPERTIES OUTPUT_NAME "zzip" RELEASE_POSTFIX "-${RELNUM}")
SET_TARGET_PROPERTIES(libzzip PROPERTIES VERSION ${VERNUM}.${FIXNUM} SOVERSION ${VERNUM})

if(ZZIPFSEEKO)
set_target_properties(libzzipfseeko PROPERTIES OUTPUT_NAME "zzipfseeko" RELEASE_POSTFIX "-${RELNUM}")
SET_TARGET_PROPERTIES(libzzipfseeko PROPERTIES VERSION ${VERNUM}.${FIXNUM} SOVERSION ${VERNUM})
endif()

if(ZZIPMMAPPED)
set_target_properties(libzzipmmapped PROPERTIES OUTPUT_NAME "zzipmmapped" RELEASE_POSTFIX "-${RELNUM}")
SET_TARGET_PROPERTIES(libzzipmmapped PROPERTIES VERSION ${VERNUM}.${FIXNUM} SOVERSION ${VERNUM})
endif()

if(ZZIP_COMPAT)
add_custom_command(OUTPUT compat/zzip.h
   COMMAND ${BASH} -c "rm -rf compat; mkdir compat"
   COMMAND ${BASH} -c "echo '#ifndef ZZIP_WARNING'     > compat/zzip.h"
   COMMAND ${BASH} -c "echo '#define ZZIP_WARNING 1'   >>compat/zzip.h"
   COMMAND ${BASH} -c "echo '#ifdef __GNUC__'          >>compat/zzip.h"
   COMMAND ${BASH} -c "echo '#warning do no not use <zzip.h>, update to include <zzip/lib.h>' >>compat/zzip.h"
   COMMAND ${BASH} -c "echo '#else'                    >>compat/zzip.h"
   COMMAND ${BASH} -c "echo '#error   do no not use <zzip.h>, update to include <zzip/lib.h>' >>compat/zzip.h"
   COMMAND ${BASH} -c "echo '#endif'                   >>compat/zzip.h"
   COMMAND ${BASH} -c "echo '#endif'                   >>compat/zzip.h"
   COMMAND ${BASH} -c "echo '#include \"zzip/lib.h\"'  >>compat/zzip.h"
   COMMAND ${BASH} -c "sed -e 's|zzip.h|zziplib.h|' -e 's|zzip/lib.h|zzip/zzip.h|' \
   compat/zzip.h >compat/zziplib.h"
   COMMAND ${BASH} -c "sed -e 's|zzip.h|zzip-io.h|' -e 's|zzip/lib.h|zzip/plugin.h|' \
   compat/zzip.h >compat/zzip-io.h"
   BYPRODUCTS compat/zziplib.h compat/zzip-io.h
   VERBATIM)
add_custom_target(compat ALL DEPENDS compat/zzip.h)
endif(ZZIP_COMPAT)

if(ZZIP_PKGCONFIG)
join_paths(libdir "\${prefix}" "${CMAKE_INSTALL_LIBDIR}")
join_paths(includedir "\${prefix}" "${CMAKE_INSTALL_INCLUDEDIR}")
add_custom_command(OUTPUT zziplib.pc
   COMMAND ${BASH} -c "echo 'prefix=${CMAKE_INSTALL_PREFIX}' > zziplib.pc"
   COMMAND ${BASH} -c "echo 'libdir=${libdir}' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'includedir=${includedir}' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'package=zziplib' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'suffix=${CMAKE_EXECUTABLE_SUFFIX}' >> zziplib.pc"
   COMMAND ${BASH} -c "echo '' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'Name: ${PROJECT_NAME}lib' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'Description: ZZipLib - libZ-based ZIP-access Library with an Easy-to-Use API' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'Version: ${PROJECT_VERSION}' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'Requires: zlib' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'Libs: -L\${libdir} -lzzip' >> zziplib.pc"
   COMMAND ${BASH} -c "echo 'Cflags: -I\${includedir}' >> zziplib.pc"
   COMMAND ${BASH} -c "sed -e 's/-lzzip/-lzzipmmapped/' -e 's/zziplib/zzipmmapped/' zziplib.pc > zzipmmapped.pc"
   COMMAND ${BASH} -c "sed -e 's/-lzzip/-lzzipfseeko/' -e 's/zziplib/zzipfseeko/' zziplib.pc > zzipfseeko.pc"
   BYPRODUCTS zzipmmapped.pc zzipfseeko.pc
   VERBATIM)
add_custom_target(pkgconfig ALL DEPENDS zziplib.pc)
endif(ZZIP_PKGCONFIG)

# install ########################################################
set(outdir ${CMAKE_CURRENT_BINARY_DIR})

if(ZZIP_PKGCONFIG)
install(FILES ${outdir}/zziplib.pc ${outdir}/zzipmmapped.pc ${outdir}/zzipfseeko.pc
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig" )
endif()

install(FILES ${libzzip_HDRS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/zzip )
install(TARGETS libzzip EXPORT zziplibTargets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

if(ZZIPFSEEKO)
install(FILES ${libzzipfseeko_HDRS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/zzip )
install(TARGETS libzzipfseeko EXPORT zziplibTargets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
endif()

if(ZZIPMMAPPED)
install(FILES ${libzzipmmapped_HDRS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/zzip )
install(TARGETS libzzipmmapped EXPORT zziplibTargets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
endif()

install(EXPORT zziplibTargets
    NAMESPACE zziplib::
    DESTINATION share/zziplib
)

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/zziplib-config.cmake.in"
[[include(CMakeFindDependencyMacro)
find_dependency(ZLIB)
file(GLOB TARGET_FILES "${CMAKE_CURRENT_LIST_DIR}/*Targets.cmake")
foreach (TARGET_FILE ${TARGET_FILES})
    include("${TARGET_FILE}")
endforeach()
]])
configure_file("${CMAKE_CURRENT_BINARY_DIR}/zziplib-config.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/zziplib-config.cmake" @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/zziplib-config.cmake DESTINATION share/zziplib)

if(ZZIP_COMPAT OR ZZIP_LIBTOOL)
  if(BUILD_SHARED_LIBS AND CMAKE_SHARED_LIBRARY_SONAME_C_FLAG)
    set(lib ${CMAKE_SHARED_LIBRARY_PREFIX})
    set(dll ${CMAKE_SHARED_LIBRARY_SUFFIX})
    get_target_property(libname libzzip OUTPUT_NAME)
    get_target_property(librelease libzzip RELEASE_POSTFIX)
    add_custom_target(libzzip_links ALL
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzip> ${lib}${libname}${librelease}${dll}.10
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzip> ${lib}${libname}${librelease}${dll}.11
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzip> ${lib}${libname}${librelease}${dll}.12
        )
    install(FILES
      ${outdir}/${lib}${libname}${librelease}${dll}.10
      ${outdir}/${lib}${libname}${librelease}${dll}.11
      ${outdir}/${lib}${libname}${librelease}${dll}.12
      DESTINATION ${CMAKE_INSTALL_LIBDIR})
  endif()
endif()

if(ZZIP_COMPAT)
  install(FILES ${outdir}/compat/zzip.h ${outdir}/compat/zziplib.h ${outdir}/compat/zzip-io.h
          DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
endif()

if(ZZIP_LIBTOOL)
  if(BUILD_SHARED_LIBS AND CMAKE_SHARED_LIBRARY_SONAME_C_FLAG)
    set(lib ${CMAKE_SHARED_LIBRARY_PREFIX})
    set(dll ${CMAKE_SHARED_LIBRARY_SUFFIX})
    if(ZZIPFSEEKO)
    get_target_property(libname libzzipfseeko OUTPUT_NAME)
    get_target_property(librelease libzzipfseeko RELEASE_POSTFIX)
    add_custom_target(libzzipfseeko_links ALL
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzipfseeko> ${lib}${libname}${librelease}${dll}.10
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzipfseeko> ${lib}${libname}${librelease}${dll}.11
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzipfseeko> ${lib}${libname}${librelease}${dll}.12
        )
    install(FILES
        ${outdir}/${lib}${libname}${librelease}${dll}.10
        ${outdir}/${lib}${libname}${librelease}${dll}.11
        ${outdir}/${lib}${libname}${librelease}${dll}.12
        DESTINATION ${CMAKE_INSTALL_LIBDIR})
    endif(ZZIPFSEEKO)
    if(ZZIPMMAPPED)
    get_target_property(libname libzzipmmapped OUTPUT_NAME)
    get_target_property(librelease libzzipmmapped RELEASE_POSTFIX)
    add_custom_target(libzzipmmaped_links ALL
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzipmmapped> ${lib}${libname}${librelease}${dll}.10
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzipmmapped> ${lib}${libname}${librelease}${dll}.11
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzipmmapped> ${lib}${libname}${librelease}${dll}.12
        )
    install(FILES
        ${outdir}/${lib}${libname}${librelease}${dll}.10
        ${outdir}/${lib}${libname}${librelease}${dll}.11
        ${outdir}/${lib}${libname}${librelease}${dll}.12
        DESTINATION ${CMAKE_INSTALL_LIBDIR})
    endif(ZZIPMMAPPED)
  endif(CMAKE_SHARED_LIBRARY_SONAME_C_FLAG)
endif(ZZIP_LIBTOOL)

if(ZZIP_LIBLATEST)
  if(BUILD_SHARED_LIBS)
    set(lib ${CMAKE_SHARED_LIBRARY_PREFIX})
    set(dll ${CMAKE_SHARED_LIBRARY_SUFFIX})
  else()
    set(lib ${CMAKE_STATIC_LIBRARY_PREFIX})
    set(dll ${CMAKE_STATIC_LIBRARY_SUFFIX})
  endif()
    get_target_property(libname libzzip OUTPUT_NAME)
    get_target_property(librelease libzzip RELEASE_POSTFIX)
    set(libzzip_target "${lib}${libname}${dll}")
    add_custom_command(OUTPUT ${libzzip_target} 
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzip> ${libzzip_target}
        )
    add_custom_target(libzzip_latest ALL
        DEPENDS ${libzzip_target}
        )
    install(FILES
        ${outdir}/${lib}${libname}${dll}
        DESTINATION ${CMAKE_INSTALL_LIBDIR})
    if(ZZIPFSEEKO)
    get_target_property(libname libzzipfseeko OUTPUT_NAME)
    get_target_property(librelease libzzipfseeko RELEASE_POSTFIX)
    set(libzzipfseeko_target "${lib}${libname}${dll}")
    add_custom_command(OUTPUT ${libzzipfseeko_target} 
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzipfseeko> ${libzzipfseeko_target}
        )
    add_custom_target(libzzipfseeko_latest ALL
        DEPENDS ${libzzipfseeko_target}
        )
    install(FILES
        ${outdir}/${lib}${libname}${dll}
        DESTINATION ${CMAKE_INSTALL_LIBDIR})
    endif(ZZIPFSEEKO)
    if(ZZIPMMAPPED)
    get_target_property(libname libzzipmmapped OUTPUT_NAME)
    get_target_property(librelease libzzipmmapped RELEASE_POSTFIX)
    set(libzzipmmapped_target "${lib}${libname}${dll}")
    add_custom_command(OUTPUT ${libzzipmmapped_target} 
        COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE_NAME:libzzipmmapped> ${libzzipmmapped_target}
        )
    add_custom_target(libzzipmmaped_latest ALL
        DEPENDS ${libzzipmmapped_target}
        )
    install(FILES
        ${outdir}/${lib}${libname}${dll}
        DESTINATION ${CMAKE_INSTALL_LIBDIR})
    endif(ZZIPMMAPPED)
endif(ZZIP_LIBLATEST)


## messages ##############################################

message(STATUS "lib zzipfseeko  to be compiled: ${ZZIPFSEEKO}")
message(STATUS "lib zzipmmapped to be compiled: ${ZZIPFSEEKO}")
message(STATUS "lib zzip symlinks as by libtool: ${ZZIP_LIBTOOL}")
message(STATUS "installing zzip compat headers: ${ZZIP_COMPAT}")
message(STATUS "installing zzip pkgconfig files: ${ZZIP_PKGCONFIG}")
