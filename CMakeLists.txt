################################################################################
# Project:  Lib bzip2
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2019, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################

cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(bz2)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# Some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# Set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR "${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Headers" CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    set(SKIP_INSTALL_HEADERS ON)
    set(SKIP_INSTALL_FILES ON)
    set(SKIP_INSTALL_EXPORT ON)
    set(CMAKE_MACOSX_RPATH ON)
else()
    include(GNUInstallDirs)

    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_DOC_DIR ${CMAKE_INSTALL_DOCDIR} CACHE INTERNAL "Share directory for data" FORCE)
    set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
endif()

if(CMAKE_GENERATOR_TOOLSET MATCHES "v([0-9]+)_xp")
    add_definitions(-D_WIN32_WINNT=0x0501)
endif()

if(WIN32)
    add_definitions( -D_USRDLL -D_WINDOWS )
endif()

add_definitions(-D_FILE_OFFSET_BITS=64)

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

include(util)
check_version(VERSION_MAJOR VERSION_MINOR VERSION_PATCH)
set(VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})

if(OSX_FRAMEWORK)
    set(FRAMEWORK_VERSION "${VERSION_MAJOR}")
endif()

report_version(${PROJECT_NAME} ${VERSION})

# Specify library include directories
include_directories("${PROJECT_SOURCE_DIR}")

set(PUBLIC_HEADERS include/ft2build.h)
file(GLOB PUBLIC_FT_HEADERS "include/freetype/*.h")
file(GLOB PUBLIC_CONFIG_HEADERS "include/freetype/config/*.h")
file(GLOB PRIVATE_HEADERS "include/freetype/internal/*.h")


set(LIB_SRCS
    blocksort.c
    huffman.c
    crctable.c
    randtable.c
    compress.c
    decompress.c
    bzlib.c
)

set(PUBLIC_HEADERS
    bzlib.h
)

if (WIN32)
    set(LIB_SRCS ${LIB_SRCS} libbz2.def)
endif ()

set(LIB_NAME bz2)
if(BUILD_SHARED_LIBS)
    add_library(${LIB_NAME} SHARED ${LIB_SRCS} ${PUBLIC_HEADERS})
    set_target_properties (${LIB_NAME} PROPERTIES
        VERSION ${VERSION}
        SOVERSION ${VERSION_MAJOR}.${VERSION_MINOR}
    DEFINE_SYMBOL BZ_EXPORT)
    if(MSVC)
      # msvc does not append 'lib' - do it here to have consistent name
      set_target_properties(${LIB_NAME} PROPERTIES PREFIX "lib")
      set_target_properties(${LIB_NAME} PROPERTIES IMPORT_PREFIX "lib")
    endif()
elseif(OSX_FRAMEWORK)
    add_library(${LIB_NAME} SHARED ${LIB_SRCS} ${PUBLIC_HEADERS})
    set_target_properties(${LIB_NAME} PROPERTIES
      FRAMEWORK TRUE
      FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
      MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
      MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
      MACOSX_FRAMEWORK_IDENTIFIER org.bzip2.libzip2
      XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
      PUBLIC_HEADER "${PUBLIC_HEADERS}"
      VERSION ${VERSION}
      SOVERSION ${VERSION_MAJOR}.${VERSION_MINOR}
      )
else()
    add_library(${LIB_NAME} STATIC ${LIB_SRCS} ${PUBLIC_HEADERS})
endif()

target_compile_definitions(${LIB_NAME} INTERFACE BZ_IMPORT)

if (NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
  install(FILES ${PUBLIC_HEADERS}
    DESTINATION ${INSTALL_INC_DIR}
  )
endif()

add_executable(bzip2 bzip2.c)
target_link_libraries(bzip2 ${LIB_NAME})
target_compile_definitions(bzip2 PUBLIC BZ_IMPORT)

add_executable(bzip2recover bzip2recover.c)
target_link_libraries(bzip2recover ${LIB_NAME})
target_compile_definitions(bzip2recover PUBLIC BZ_IMPORT)

if (NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    install(TARGETS ${LIB_NAME}
        EXPORT ${PACKAGE_UPPER_NAME}Targets
        RUNTIME DESTINATION ${INSTALL_BIN_DIR}
        LIBRARY DESTINATION ${INSTALL_LIB_DIR}
        ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
        INCLUDES DESTINATION ${INSTALL_INC_DIR}
        FRAMEWORK DESTINATION ${INSTALL_LIB_DIR}
    )
    install(TARGETS bzip2 bzip2recover
          RUNTIME DESTINATION ${INSTALL_BIN_DIR}
          LIBRARY DESTINATION ${INSTALL_LIB_DIR}
          ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
          INCLUDES DESTINATION ${INSTALL_INC_DIR}
          FRAMEWORK DESTINATION ${INSTALL_LIB_DIR}
      )
endif ()

if(NOT OSX_FRAMEWORK)
  install(FILES manual.pdf DESTINATION ${INSTALL_DOC_DIR})
endif()

option(BUILD_TESTING "Build unit tests" OFF)
if(BUILD_TESTING AND NOT WIN32)

    include(CTest)

    add_test(NAME sample1_ref_build COMMAND sh -c "$<TARGET_FILE:bzip2> -1 < ${PROJECT_SOURCE_DIR}/sample1.ref > ${CMAKE_CURRENT_BINARY_DIR}/sample1.rb2"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    add_test(NAME sample1_ref_cmp COMMAND ${CMAKE_COMMAND} -E compare_files
        ${CMAKE_CURRENT_BINARY_DIR}/sample1.rb2 ${PROJECT_SOURCE_DIR}/sample1.bz2)

    add_test(NAME sample2_ref_build COMMAND sh -c "$<TARGET_FILE:bzip2> -2 < ${PROJECT_SOURCE_DIR}/sample2.ref > ${CMAKE_CURRENT_BINARY_DIR}/sample2.rb2"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    add_test(NAME sample2_ref_cmp COMMAND ${CMAKE_COMMAND} -E compare_files
    ${CMAKE_CURRENT_BINARY_DIR}/sample2.rb2 ${PROJECT_SOURCE_DIR}/sample2.bz2)

    add_test(NAME sample3_ref_build COMMAND sh -c "$<TARGET_FILE:bzip2> -3 < ${PROJECT_SOURCE_DIR}/sample3.ref > ${CMAKE_CURRENT_BINARY_DIR}/sample3.rb2"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    add_test(NAME sample3_ref_cmp COMMAND ${CMAKE_COMMAND} -E compare_files
    ${CMAKE_CURRENT_BINARY_DIR}/sample3.rb2 ${PROJECT_SOURCE_DIR}/sample3.bz2)

    add_test(NAME sample1_bz2_build COMMAND sh -c "$<TARGET_FILE:bzip2> -d < ${PROJECT_SOURCE_DIR}/sample1.bz2 > ${CMAKE_CURRENT_BINARY_DIR}/sample1.tst"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    add_test(NAME sample1_bz2_cmp COMMAND ${CMAKE_COMMAND} -E compare_files
    ${CMAKE_CURRENT_BINARY_DIR}/sample1.tst ${PROJECT_SOURCE_DIR}/sample1.ref)

    add_test(NAME sample2_bz2_build COMMAND sh -c "$<TARGET_FILE:bzip2> -d < ${PROJECT_SOURCE_DIR}/sample2.bz2 > ${CMAKE_CURRENT_BINARY_DIR}/sample2.tst"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    add_test(NAME sample2_bz2_cmp COMMAND ${CMAKE_COMMAND} -E compare_files
    ${CMAKE_CURRENT_BINARY_DIR}/sample2.tst ${PROJECT_SOURCE_DIR}/sample2.ref)

    add_test(NAME sample3_bz2_build COMMAND sh -c "$<TARGET_FILE:bzip2> -ds < ${PROJECT_SOURCE_DIR}/sample3.bz2 > ${CMAKE_CURRENT_BINARY_DIR}/sample3.tst"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    add_test(NAME sample3_bz2_cmp COMMAND ${CMAKE_COMMAND} -E compare_files
    ${CMAKE_CURRENT_BINARY_DIR}/sample3.tst ${PROJECT_SOURCE_DIR}/sample3.ref)

endif()

# Export package ===============================================================

# Add path to includes to build-tree export
target_include_directories(${LIB_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)

# Add all targets to the build-tree export set
export(TARGETS ${LIB_NAME}
    FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

if(REGISTER_PACKAGE)
    # Export the package for use from the build-tree
    # (this registers the build-tree with a global CMake-registry)
    export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the ZLIBConfig.cmake file
configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    # Install the <Package>Config.cmake
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
