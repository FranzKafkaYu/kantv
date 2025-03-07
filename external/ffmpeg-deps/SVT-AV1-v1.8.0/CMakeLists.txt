#
# Copyright(c) 2018 Intel Corporation
#
# This source code is subject to the terms of the BSD 2 Clause License and
# the Alliance for Open Media Patent License 1.0. If the BSD 2 Clause License
# was not distributed with this source code in the LICENSE file, you can
# obtain it at https://www.aomedia.org/license/software-license. If the Alliance for Open
# Media Patent License 1.0 was not distributed with this source code in the
# PATENTS file, you can obtain it at https://www.aomedia.org/license/patent-license.
#

cmake_minimum_required(VERSION 3.16)

#set(CMAKE_VERBOSE_MAKEFILE ON)

#weiguo added
SET(CMAKE_VERBOSE_MAKEFILE ON)
message(STATUS "prefix path: ${PREFIX}")
message("BUILD_LIB_TYPE: ${BUILD_LIB_TYPE}")
set(CMAKE_INSTALL_PREFIX ${PREFIX})
IF (${BUILD_LIB_TYPE} MATCHES "STATIC")
message("disable shared libs")
set(BUILD_SHARED_LIBS OFF)
ENDIF()
#end added
set (CMAKE_C_FLAGS "${CMAKE_C_FLAgs} -fPIC")
set (CMAKE_CXX_FLAGS "${CMAKE_C_FLAgs} -fPIC")


if("${CMAKE_CURRENT_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_BINARY_DIR}")
    message(WARNING "Building in-source is highly not recommended\n"
                    "Please use the Build folder or create your own.")
endif()

project(svt-av1 VERSION 1.8.0
    LANGUAGES C CXX)

if(POLICY CMP0063)
  cmake_policy(SET CMP0063 NEW)
endif()
if(POLICY CMP0069)
    cmake_policy(SET CMP0069 NEW)
endif()
if(POLICY CMP0077)
    cmake_policy(SET CMP0077 NEW)
endif()


set(CMAKE_C_VISIBILITY_PRESET hidden)
# Default build type to release if the generator does not has its own set of build types
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release)
endif()

if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
    message(WARNING "32-bit is not supported")
endif()

option(COMPILE_C_ONLY "Compile only C code with no simds (autodetect, default off for x86)" OFF)

include(CheckCSourceCompiles)

check_c_source_compiles("
#if defined(_M_X64) || defined(__x86_64__)
#else
#error \"Non-x64\"
#endif
int main(void) {}
" HAVE_X86_PLATFORM)

check_c_source_compiles("
#if defined(__aarch64__) || defined(_M_ARM64)
#else
#error \"Non-arm64\"
#endif
int main(void) {}
" HAVE_ARM_PLATFORM)

if(NOT COMPILE_C_ONLY AND HAVE_X86_PLATFORM)
    find_program(YASM_EXE yasm)
    option(ENABLE_NASM "Use nasm if available (Uses yasm by default if found)" OFF)
    if(YASM_EXE AND NOT CMAKE_ASM_NASM_COMPILER MATCHES "yasm" AND NOT ENABLE_NASM)
        set(CMAKE_ASM_NASM_COMPILER "${YASM_EXE}" CACHE FILEPATH "Path to nasm compatible compiler" FORCE)
    else()
        set(NASM_VERSION "0.0.0")
        include(CheckLanguage)
        check_language(ASM_NASM)
        execute_process(COMMAND ${CMAKE_ASM_NASM_COMPILER} -v
            OUTPUT_VARIABLE NASM_VERSION
            ERROR_QUIET
            OUTPUT_STRIP_TRAILING_WHITESPACE)
        string(REGEX MATCH "([.0-9])+" NASM_VERSION "${NASM_VERSION}")
        # NASM_VERSION should now contain something like 2.14.02
        # Needs to error out on a version less than 2.14
        if(NASM_VERSION VERSION_LESS "2.14" AND CMAKE_ASM_NASM_COMPILER MATCHES "nasm")
            message(FATAL_ERROR "Found nasm is too old (requires at least 2.14, found ${NASM_VERSION})!")
        endif()
    endif()
    enable_language(ASM_NASM)
    add_definitions(-DARCH_X86_64=1)
elseif(NOT COMPILE_C_ONLY AND HAVE_ARM_PLATFORM)
    add_definitions(-DARCH_AARCH64=1)
endif()

include(GNUInstallDirs)
include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

set(CMAKE_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/Bin/${CMAKE_BUILD_TYPE}/" CACHE PATH "Location to write the resulting binaries to")

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_OUTPUT_DIRECTORY})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_OUTPUT_DIRECTORY})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_OUTPUT_DIRECTORY})

#Clang support, required to build static with LTO
if(CMAKE_C_COMPILER_ID MATCHES "Clang" AND UNIX AND NOT APPLE)
    find_program(LLVM_LD_EXE llvm-ld)
    find_program(LLVM_AR_EXE llvm-ar)
    find_program(LLVM_RANLIB_EXE llvm-ranlib)
    if(LLVM_LD_EXE)
        SET (CMAKE_LINKER  "llvm-ld")
    endif()
    if(LLVM_AR_EXE)
        SET (CMAKE_AR      "llvm-ar")
    endif()
    if(LLVM_RANLIB_EXE)
        SET (CMAKE_RANLIB  "llvm-ranlib")
    endif()
endif()

if(MSVC)
    if(DEFINED SANITIZER)
        message(WARNING "Sanitizers are not available with MSVC currently")
    endif()
    unset(SANITIZER)
    unset(SANITIZER CACHE)
else()
    set(SANITIZER "" CACHE STRING "Sanitizer to enable. Currently known values are Address, Memory, Thread, Undefined, and Integer")
    set_property(CACHE SANITIZER PROPERTY STRINGS Address Memory Thread Undefined Integer)
endif()

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 11)

# BUILD_SHARED_LIBS is a standard CMake variable, but we declare it here to
# make it prominent in the GUI.
option(BUILD_SHARED_LIBS "Build shared libraries (DLLs)." ON)
message(STATUS "BUILD_SHARED_LIBS: ${BUILD_SHARED_LIBS}")

option(BUILD_TESTING "Build SvtAv1UnitTests, SvtAv1ApiTests, and SvtAv1E2ETests unit tests")
option(COVERAGE "Generate coverage report")
if(NOT MSVC)
    option(SVT_AV1_PGO "Enable profile guided optimization. Creates the RunPGO and CompilePGO targets")
    # options for internal use
    option(SVT_INTERNAL_PGO_GENERATE "Generate a profile guided optimization profile")
    option(SVT_INTERNAL_PGO_USE "Use a profile for profile guided optimization")
    mark_as_advanced(SVT_INTERNAL_PGO_GENERATE SVT_INTERNAL_PGO_USE)
    set(SVT_AV1_PGO_DIR "${CMAKE_BINARY_DIR}" CACHE PATH "Path to the directory passed to -fprofile-dir for use with profile guided optimization")
    set(SVT_AV1_PGO_CUSTOM_VIDEOS "" CACHE STRING "Semi-colon separated list of directories with y4m videos to run PGO on instead of objective-1-fast")
else()
    set(SVT_AV1_PGO OFF)
endif()

if(MSVC AND SVT_AV1_PGO)
    message(WARNING "PGO is not yet supported for MSVC, disabling")
    unset(SVT_AV1_PGO)
    unset(SVT_AV1_PGO CACHE)
endif()

option(BUILD_APPS "Build Enc and Dec Apps" ON)
option(EXCLUDE_HASH "Disables the use of the git hash in the build to produce hash exact binaries, implies REPRODUCIBLE_BUILDS")
option(REPRODUCIBLE_BUILDS "Disable use of certain C macros to always generate reproducable builds" ${EXCLUDE_HASH})
option(BUILD_ENC "Build Encoder lib and app" ON)
option(BUILD_DEC "Build Decoder lib and app" ON)

#weiguo
set(BUILD_APPS OFF)

if(NOT BUILD_ENC AND NOT BUILD_DEC)
    message(FATAL_ERROR "Not building either the encoder and decoder doesn't make sense.")
endif()
if(SVT_AV1_PGO AND NOT BUILD_APPS)
    message(FATAL_ERROR "Can't use pgo and not build the apps")
endif()

if(SVT_INTERNAL_PGO_USE)
    if(MSVC)
        set(PGO_COMPILER_FLAGS "/GL")
        set(PGO_LINKER_FLAGS "/LTCG /USEPROFILE")
    else()
        set(PGO_COMPILER_FLAGS "-fprofile-use=${SVT_AV1_PGO_DIR} -fprofile-correction")
        set(PGO_LINKER_FLAGS "-fprofile-use=${SVT_AV1_PGO_DIR} -fprofile-correction")
    endif()
elseif(SVT_INTERNAL_PGO_GENERATE)
    if(MSVC)
        set(PGO_COMPILER_FLAGS "/GL")
        set(PGO_LINKER_FLAGS "/LTCG /GENPROFILE")
    else()
        set(PGO_COMPILER_FLAGS "-fprofile-generate=${SVT_AV1_PGO_DIR} -fprofile-correction")
        set(PGO_LINKER_FLAGS "-fprofile-generate=${SVT_AV1_PGO_DIR} -fprofile-correction")
    endif()
endif()

if(SVT_AV1_PGO)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${PGO_COMPILER_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${PGO_COMPILER_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${PGO_LINKER_FLAGS}")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${PGO_LINKER_FLAGS}")
    set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${PGO_LINKER_FLAGS}")
endif()

if(EXCLUDE_HASH)
    add_link_options(-Wl,--build-id=none)
    add_definitions(-DEXCLUDE_HASH=1)
else()
    add_definitions(-DEXCLUDE_HASH=0)
endif()

if(REPRODUCIBLE_BUILDS)
    add_definitions(-DREPRODUCIBLE_BUILDS=1)
else()
    add_definitions(-DREPRODUCIBLE_BUILDS=0)
endif()


if(WIN32)
    set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -DWIN64")
else()
    set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -DUNIX64")
endif()

if(UNIX)
    if(APPLE)
        set(CMAKE_MACOSX_RPATH 1)
        if(CMAKE_C_COMPILER_ID MATCHES "AppleClang")
            set(CMAKE_C_ARCHIVE_CREATE   "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
            set(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
            set(CMAKE_C_ARCHIVE_FINISH   "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
            set(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
        endif()
    else()
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -z noexecstack -z relro -z now")
    endif()
endif()

# Always build with -fPIC
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
#failed with nasm
#add_compile_options(-fPIC)
#ok
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fpic")
set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -fpic")

function(check_flag lang flag)
    string(REGEX REPLACE "[^A-Za-z0-9]" "_" flag_var "${flag}")
    if(NOT DEFINED ${lang}_FLAG${flag_var})
        execute_process(COMMAND ${CMAKE_COMMAND} -E echo_append "-- Checking ${lang} flag support for: [${flag}] - ")
        if(CMAKE_C_COMPILER_ID MATCHES "Clang")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Werror=unused-command-line-argument")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror=unused-command-line-argument")
        endif()
        if("${lang}" STREQUAL "CXX")
            check_cxx_compiler_flag("${flag}" "${lang}_FLAG${flag_var}")
        else()
            check_c_compiler_flag("${flag}" "${lang}_FLAG${flag_var}")
        endif()
        if(${lang}_FLAG${flag_var})
            execute_process(COMMAND ${CMAKE_COMMAND} -E echo "Yes")
        else()
            execute_process(COMMAND ${CMAKE_COMMAND} -E echo "No")
        endif()
    endif()
    set(check_flag ${${lang}_FLAG${flag_var}} PARENT_SCOPE)
endfunction()

function(check_flags_add lang)
    cmake_parse_arguments(check_${lang}_flags_add
        "PREPEND"
        "TYPE"
        ""
        ${ARGN})

    if(check_${lang}_flags_add_TYPE)
        set(check_${lang}_flags_add_TYPE "_${check_${lang}_flags_add_TYPE}")
    endif()
    if(check_${lang}_flags_add_PREPEND)
        list(REVERSE check_${lang}_flags_add_UNPARSED_ARGUMENTS)
    endif()

    set(CMAKE_REQUIRED_QUIET true)

    foreach(flag IN LISTS check_${lang}_flags_add_UNPARSED_ARGUMENTS)
        check_flag("${lang}" "${flag}")
        if(NOT check_flag)
            continue()
        endif()

        if(check_${lang}_flags_add_PREPEND)
            set(CMAKE_${lang}_FLAGS${check_${lang}_flags_add_TYPE} "${flag} ${CMAKE_${lang}_FLAGS${check_${lang}_flags_add_TYPE}}")
        else()
            set(CMAKE_${lang}_FLAGS${check_${lang}_flags_add_TYPE} "${CMAKE_${lang}_FLAGS${check_${lang}_flags_add_TYPE}} ${flag}")
        endif()
    endforeach()
    if(CMAKE_BUILD_TYPE MATCHES "[Dd][Ee][Bb][Uu][Gg]")
        message(STATUS "${CMAKE_CURRENT_SOURCE_DIR} ${lang}FLAGS: ${CMAKE_${lang}_FLAGS${check_${lang}_flags_add_TYPE}}")
    endif()
    set(CMAKE_${lang}_FLAGS${check_${lang}_flags_add_TYPE} "${CMAKE_${lang}_FLAGS${check_${lang}_flags_add_TYPE}}" PARENT_SCOPE)
endfunction()

macro(check_both_flags_add)
    check_flags_add(C "${ARGV}")
    check_flags_add(CXX "${ARGV}")
endmacro()

macro(add_clike_and_ld_flags)
    string(REPLACE ";" " " add_clike_and_ld_flags "${ARGV}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${add_clike_and_ld_flags}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${add_clike_and_ld_flags}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${add_clike_and_ld_flags}")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${add_clike_and_ld_flags}")
endmacro()


option(SVT_AV1_LTO "Attempt to enable Link Time Optimization if available" OFF)
if(SVT_AV1_LTO)
    if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.9)
        include(CheckIPOSupported)
        check_ipo_supported(RESULT svt_av1_ipo_supported)
        if(svt_av1_ipo_supported AND NOT DEFINED CMAKE_INTERPROCEDURAL_OPTIMIZATION)
            set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
        endif()
    else()
        if(MSVC)
            add_clike_and_ld_flags(/LTCG:INCREMENTAL)
        else()
            add_clike_and_ld_flags(-flto)
        endif()
    endif()
endif()

check_both_flags_add(PREPEND -Wextra -Wformat -Wformat-security)

if(MSVC)
    check_both_flags_add(PREPEND /W3)
    check_both_flags_add(/MP)
    check_both_flags_add(TYPE DEBUG /Od)
else()
    check_both_flags_add(PREPEND -Wall)
    option(NATIVE "Build for native performance (march=native)" OFF)
    if(NATIVE)
        check_both_flags_add(-march=native)
    endif()
    if(MINGW)
        check_both_flags_add(-mxsave -fno-asynchronous-unwind-tables)
    else()
        check_both_flags_add(-fstack-protector-strong)
        check_both_flags_add(PREPEND -fno-stack-clash-protection)
    endif()
    check_both_flags_add(-mno-avx)
endif()

if(CMAKE_C_FLAGS MATCHES "-O" AND NOT CMAKE_C_FLAGS MATCHES "-O0" AND NOT MINGW)
    add_definitions(-D_FORTIFY_SOURCE=2)
endif()

if(CMAKE_ASM_NASM_OBJECT_FORMAT MATCHES "win")
    set(CMAKE_ASM_NASM_FLAGS_DEBUG "${CMAKE_ASM_NASM_FLAGS_DEBUG} -gcv8")
elseif(CMAKE_ASM_NASM_COMPILER MATCHES "nasm")
    set(CMAKE_ASM_NASM_FLAGS_DEBUG "${CMAKE_ASM_NASM_FLAGS_DEBUG} -gdwarf")
elseif(CMAKE_ASM_NASM_COMPILER MATCHES "yasm")
    if(CMAKE_ASM_NASM_OBJECT_FORMAT MATCHES "macho")
        set(CMAKE_ASM_NASM_FLAGS_DEBUG "${CMAKE_ASM_NASM_FLAGS_DEBUG} -gnull")
    else()
        set(CMAKE_ASM_NASM_FLAGS_DEBUG "${CMAKE_ASM_NASM_FLAGS_DEBUG} -gdwarf2")
    endif()
endif()

include(CheckSymbolExists)

check_symbol_exists("_mm512_extracti64x4_epi64" "immintrin.h" HAS_AVX512)
if(HAS_AVX512)
    option(ENABLE_AVX512 "Enable building avx512 code" OFF)
else()
    set(ENABLE_AVX512 OFF CACHE INTERNAL "Enable building avx512 code")
endif()
if(ENABLE_AVX512)
    add_definitions(-DEN_AVX512_SUPPORT=1)
else()
    add_definitions(-DEN_AVX512_SUPPORT=0)
endif()

# ASM compiler macro
macro(ASM_COMPILE_TO_TARGET target)
    if(CMAKE_GENERATOR STREQUAL "Xcode")
        if(CMAKE_BUILD_TYPE MATCHES Release)
            set(ASM_FLAGS "${CMAKE_ASM_NASM_FLAGS_RELEASE}")
        elseif(CMAKE_BUILD_TYPE MATCHES Debug)
            set(ASM_FLAGS "${CMAKE_ASM_NASM_FLAGS_DEBUG}")
        elseif(CMAKE_BUILD_TYPE MATCHES MinSizeRel)
            set(ASM_FLAGS "${CMAKE_ASM_NASM_FLAGS_MINSIZEREL}")
        elseif(CMAKE_BUILD_TYPE MATCHES RelWithDebInfo)
            set(ASM_FLAGS "${CMAKE_ASM_NASM_FLAGS_RELWITHDEBINFO}")
        endif()

        get_property(inc_dirs DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY INCLUDE_DIRECTORIES)
        foreach(inc_dir ${inc_dirs})
            set(ASM_FLAGS "${ASM_FLAGS} -I${inc_dir}")
        endforeach()

        string(REGEX REPLACE " " ";" ASM_FLAGS "${ASM_FLAGS} ${CMAKE_ASM_NASM_FLAGS}")

        foreach(asm_file ${ARGN})
            get_filename_component(filename "${asm_file}" NAME_WE)
            set(SIMD_OBJ "${CMAKE_CURRENT_BINARY_DIR}/${filename}${CMAKE_C_OUTPUT_EXTENSION}")
            message(STATUS "Pregenerating ASM_NASM file ${filename}${CMAKE_C_OUTPUT_EXTENSION}")
            execute_process(COMMAND ${CMAKE_ASM_NASM_COMPILER}
                    -f${CMAKE_ASM_NASM_OBJECT_FORMAT}
                    -I${CMAKE_CURRENT_SOURCE_DIR}
                    -o${SIMD_OBJ}
                    ${ASM_FLAGS}
                    ${CMAKE_CURRENT_SOURCE_DIR}/${asm_file}
                    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR})
            list(APPEND SIMD_OBJS "${SIMD_OBJ}")
        endforeach()
        #target_sources(${target} PUBLIC ${SIMD_OBJS})
        target_sources(common_lib INTERFACE ${SIMD_OBJS})
    else()
        # For every other generator
        foreach(asm_file ${ARGN})
            get_filename_component(filename "${asm_file}" ABSOLUTE)
            target_sources(${target} PUBLIC ${filename})
        endforeach()
    endif()
endmacro()

check_symbol_exists(strnlen_s "string.h" HAVE_STRNLEN_S)
check_symbol_exists(strncpy_s "string.h" HAVE_STRNCPY_S)
check_symbol_exists(strcpy_s "string.h" HAVE_STRCPY_S)
set_property(DIRECTORY .
    APPEND
    PROPERTY COMPILE_DEFINITIONS
        SAFECLIB_STR_NULL_SLACK=1
        $<$<BOOL:${HAVE_STRNLEN_S}>:HAVE_STRNLEN_S=1>
        $<$<BOOL:${HAVE_STRNCPY_S}>:HAVE_STRNCPY_S=1>
        $<$<BOOL:${HAVE_STRCPY_S}>:HAVE_STRCPY_S=1>
        $<$<BOOL:${WIN32}>:_WIN32_WINNT=0x0601>)
if(NOT HAVE_STRCPY_S OR NOT HAVE_STRNCPY_S OR NOT HAVE_STRNLEN_S)
    add_library(safestringlib OBJECT
        third_party/safestringlib/safeclib_private.h
        third_party/safestringlib/safe_lib.h
        third_party/safestringlib/safe_types.h
        third_party/safestringlib/safe_lib_errno.h
        third_party/safestringlib/safe_str_lib.h
        third_party/safestringlib/safe_str_constraint.h
        third_party/safestringlib/safe_str_constraint.c
        third_party/safestringlib/ignore_handler_s.c
        $<$<NOT:$<BOOL:${HAVE_STRNLEN_S}>>:third_party/safestringlib/strnlen_s.c>
        $<$<NOT:$<BOOL:${HAVE_STRNCPY_S}>>:third_party/safestringlib/strncpy_s.c>
        $<$<NOT:$<BOOL:${HAVE_STRCPY_S}>>:third_party/safestringlib/strcpy_s.c>)
endif()

# Try to find valgrind.h for the RUNNING_ON_VALGRIND macro
# needed for now for svt_dav1d_inv_txfm_add_dct_dct_32x16_8bpc_avx2
# due to the later crashing libVEX when running on valgrind
# TODO: remove this once the issue is fixed
include(CheckIncludeFile)
check_include_file("valgrind/valgrind.h" HAVE_VALGRIND_H)
if(HAVE_VALGRIND_H)
    set_property(DIRECTORY .
        APPEND
        PROPERTY COMPILE_DEFINITIONS
            HAVE_VALGRIND_H=1)
else()
    set_property(DIRECTORY .
        APPEND
        PROPERTY COMPILE_DEFINITIONS
            HAVE_VALGRIND_H=0)
endif()

# try to see if __builtin_expect is available
check_c_source_compiles("
    int main() {
        if (__builtin_expect(1, 0)) {
            return 1;
        }
        return 0;
    }" HAVE_BUILTIN_EXPECT)
if(HAVE_BUILTIN_EXPECT)
    set_property(DIRECTORY .
        APPEND
        PROPERTY COMPILE_DEFINITIONS
            HAVE_BUILTIN_EXPECT=1)
else()
    set_property(DIRECTORY .
        APPEND
        PROPERTY COMPILE_DEFINITIONS
            HAVE_BUILTIN_EXPECT=0)
endif()

include_directories(.)

# Find out if we have threading available
set(CMAKE_THREAD_PREFER_PTHREADS ON)
find_package(Threads)

string(TOLOWER "${SANITIZER}" SANITIZER)

# Address Memory Thread Undefined
if(SANITIZER STREQUAL "address")
    add_clike_and_ld_flags(-fsanitize=address)
elseif(SANITIZER STREQUAL "memory")
    add_clike_and_ld_flags(-fsanitize=memory -fsanitize-memory-track-origins)
elseif(SANITIZER STREQUAL "thread")
    add_clike_and_ld_flags(-fsanitize=thread)
elseif(SANITIZER STREQUAL "undefined")
    add_clike_and_ld_flags(-fsanitize=undefined)
elseif(SANITIZER STREQUAL "integer")
    add_clike_and_ld_flags(-fsanitize=integer)
elseif(SANITIZER)
    message(WARNING "Unknown sanitizer: ${SANITIZER}, going to try adding it anyway")
    add_clike_and_ld_flags(-fsanitize=${SANITIZER})
endif()

if(SANITIZER)
    check_both_flags_add(-fno-omit-frame-pointer -fno-optimize-sibling-calls)
endif()

if(COVERAGE)
    add_clike_and_ld_flags(--coverage)
endif()

set(SVT_AV1_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}")
set(SVT_AV1_LIBDIR "${CMAKE_INSTALL_LIBDIR}")

if(NOT IS_ABSOLUTE "${SVT_AV1_INCLUDEDIR}")
    string(PREPEND SVT_AV1_INCLUDEDIR "\${prefix}/")
endif()
if(NOT IS_ABSOLUTE "${CMAKE_INSTALL_LIBDIR}")
    string(PREPEND SVT_AV1_LIBDIR "\${exec_prefix}/")
endif()

# Add Subdirectories
add_subdirectory(Source/Lib/Common)
if(BUILD_ENC)
    add_subdirectory(Source/Lib/Encoder)
endif()
if(BUILD_DEC)
    add_subdirectory(Source/Lib/Decoder)
endif()
if(BUILD_APPS AND BUILD_ENC)
    add_subdirectory(Source/App/EncApp)
endif()
if(BUILD_APPS AND BUILD_DEC)
    add_subdirectory(Source/App/DecApp)
endif()
if(SVT_AV1_PGO)
    set_target_properties(SvtAv1EncApp SvtAv1DecApp PROPERTIES EXCLUDE_FROM_ALL TRUE)
    string(REPLACE ";" "::" SVT_AV1_PGO_CUSTOM_VIDEOS "${SVT_AV1_PGO_CUSTOM_VIDEOS}")

    add_custom_target(PGOCompileGen
        ${CMAKE_COMMAND} ${CMAKE_BINARY_DIR} -DSVT_INTERNAL_PGO_GENERATE=ON -DSVT_INTERNAL_PGO_USE=OFF
        COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}
        --clean-first
        --target SvtAv1EncApp
        --target SvtAv1DecApp
        COMMENT "Compiling with PGO generate"
        DEPENDS SvtAv1EncApp SvtAv1DecApp)
    add_custom_target(PGOCompileGenUse
        ${CMAKE_COMMAND} ${CMAKE_BINARY_DIR} -DSVT_INTERNAL_PGO_GENERATE=ON -DSVT_INTERNAL_PGO_USE=ON
        COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}
        --clean-first
        --target SvtAv1EncApp
        --target SvtAv1DecApp
        COMMENT "Compiling PGO gen+use"
        DEPENDS SvtAv1EncApp SvtAv1DecApp)
    add_custom_target(PGOCompileUse
        ${CMAKE_COMMAND} ${CMAKE_BINARY_DIR} -DSVT_INTERNAL_PGO_GENERATE=OFF -DSVT_INTERNAL_PGO_USE=ON
        COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}
        --clean-first
        --target SvtAv1EncApp
        --target SvtAv1DecApp
        COMMENT "Compiling PGO use"
        DEPENDS SvtAv1EncApp SvtAv1DecApp)

    if(SVT_AV1_PGO_CUSTOM_VIDEOS)
        add_custom_target(PGOGenerateProfile ${CMAKE_COMMAND} -P
            ${PROJECT_SOURCE_DIR}/Build/pgohelper.cmake
            ${CMAKE_BINARY_DIR}
            ${SVT_AV1_PGO_CUSTOM_VIDEOS}
            $<TARGET_FILE:SvtAv1EncApp>
            $<TARGET_FILE:SvtAv1DecApp>
            COMMENT "Generating PGO profile")
    else()
        add_custom_target(DownloadObjOneFast ${CMAKE_COMMAND} -P
            ${PROJECT_SOURCE_DIR}/Build/obj1fastdownloader.cmake
            ${CMAKE_BINARY_DIR}
            COMMENT "Downloading objective-1-fast")
        add_custom_target(PGOGenerateProfile ${CMAKE_COMMAND} -P
            ${PROJECT_SOURCE_DIR}/Build/pgohelper.cmake
            ${CMAKE_BINARY_DIR}
            ${CMAKE_BINARY_DIR}/objective-1-fast
            $<TARGET_FILE:SvtAv1EncApp>
            $<TARGET_FILE:SvtAv1DecApp>
            COMMENT "Generating PGO profile"
            DEPENDS DownloadObjOneFast)
    endif()

    add_custom_target(RunPGO ALL
        ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --config $<CONFIG> --target PGOCompileGen
        COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --config $<CONFIG> --target PGOGenerateProfile
        COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --config $<CONFIG> --target PGOCompileGenUse
        COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --config $<CONFIG> --target PGOGenerateProfile
        COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --config $<CONFIG> --target PGOCompileUse
        COMMENT "Compiling PGO Binary"
        DEPENDS PGOCompileUse)
endif()

if(BUILD_TESTING)
    include(CTest)
    message(STATUS "Building UnitTests")
    add_subdirectory(test)
    add_subdirectory(third_party/googletest)
endif()

add_subdirectory(third_party/fastfeat)

if(NOT COMPILE_C_ONLY AND HAVE_X86_PLATFORM)
    add_subdirectory(third_party/cpuinfo)
endif()

install(DIRECTORY ${PROJECT_SOURCE_DIR}/Source/API/ DESTINATION "${CMAKE_INSTALL_FULL_INCLUDEDIR}/svt-av1" FILES_MATCHING PATTERN "*.h")
