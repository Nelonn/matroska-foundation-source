cmake_minimum_required(VERSION 3.20)

project("corec" VERSION 1.0.0 LANGUAGES C)

include(CheckIncludeFile)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Options
option(CONFIG_STDIO "Build Core-C file API with stdio support" ON)
option(CONFIG_DEBUGCHECKS "Add extra checks in Core-C objects/helpers in debug builds" ON)
option(CONFIG_DEBUG_LEAKS "Debug leaks in Core-C objects in debug builds" OFF)

set(CONFIG_UNICODE "best" CACHE STRING "Select the type of tchar_t (best, wchar, utf8)")
set_property(CACHE CONFIG_UNICODE PROPERTY STRINGS "best;utf8;wchar")

# Unicode configuration
if(CONFIG_UNICODE STREQUAL "utf8")
    set(CONFIG_UNICODE_MODE CONFIG_UNICODE_UTF8)
elseif(CONFIG_UNICODE STREQUAL "wchar")
    set(CONFIG_UNICODE_MODE CONFIG_UNICODE_WCHAR)
else()
    set(CONFIG_UNICODE_MODE CONFIG_UNICODE_BEST)
endif()

# Generate config header
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/corec_config.h.in ${CMAKE_CURRENT_BINARY_DIR}/corec_config.h)

# Collect all source files (platform-independent)
set(corec_SOURCES
    # Node API
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/node/node.c
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/node/nodetree.c
    # Array API
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/array/array.c
    # String API (base)
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/str/str.c
    # Parser API
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/parser/parser2.c
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/parser/strtab.c
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/parser/strtypes.c
    # File API (base)
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/bufstream.c
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/memstream.c
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/streams.c
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/tools.c
)

# Platform-specific sources
if(WIN32)
    list(APPEND corec_SOURCES
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/str/str_win32.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/file_win32.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/date/date_win32.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/charconvert/charconvert_win32.c
    )
elseif(APPLE)
    list(APPEND corec_SOURCES
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/str/str_osx.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/file_libc.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/date/date_libc.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/charconvert/charconvert_osx.c
    )
elseif(UNIX)
    list(APPEND corec_SOURCES
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/str/str_linux.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/file_libc.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/date/date_libc.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/charconvert/charconvert_linux.c
    )
else()
    # Fallback for other platforms
    list(APPEND corec_SOURCES
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/date/date_libc.c
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/charconvert/charconvert_utf8.c
    )
endif()

# stdio support
if(CONFIG_STDIO)
    list(APPEND corec_SOURCES
        ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/stream_stdio.c
    )
endif()

# Public headers
set(corec_PUBLIC_HEADERS
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/corec.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/err.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helper.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/memheap.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/portab.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/confhelper.h
    ${CMAKE_CURRENT_BINARY_DIR}/corec_config.h
    # Node
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/node/node.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/node/nodebase.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/node/nodetree.h
    # Array
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/array/array.h
    # String
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/str/str.h
    # Parser
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/parser/parser.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/parser/strtypes.h
    # File
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/file.h
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/file/streams.h
    # Date
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/date/date.h
    # Charconvert
    ${CMAKE_CURRENT_SOURCE_DIR}/corec/helpers/charconvert/charconvert.h
)

# Create the library
add_library(corec ${corec_SOURCES} ${corec_PUBLIC_HEADERS})
add_library(corec::corec ALIAS corec)

set_target_properties(corec PROPERTIES
    C_STANDARD 11
    PUBLIC_HEADER "${corec_PUBLIC_HEADERS}"
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
)

# Include directories
target_include_directories(corec
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Platform-specific compile definitions and link libraries
if(WIN32)
    target_link_libraries(corec PRIVATE shell32)
elseif(UNIX)
    check_include_file(sys/vfs.h HAVE_SYS_VFS_H)
    check_include_file(sys/statvfs.h HAVE_SYS_STATVFS_H)
    check_include_file(sys/mount.h HAVE_SYS_MOUNT_H)
    
    if(HAVE_SYS_MOUNT_H)
        target_compile_definitions(corec PRIVATE HAVE_SYS_MOUNT_H)
    endif()
    if(HAVE_SYS_VFS_H)
        target_compile_definitions(corec PRIVATE HAVE_SYS_VFS_H)
    endif()
    if(HAVE_SYS_STATVFS_H)
        target_compile_definitions(corec PRIVATE HAVE_SYS_STATVFS_H)
    endif()
endif()

if(APPLE)
    target_link_libraries(corec PRIVATE "-framework CoreFoundation")
endif()

if(CYGWIN)
    target_link_libraries(corec PRIVATE iconv)
endif()

# Installation
install(TARGETS corec
    EXPORT corecTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/corec
)

# Install subdirectory headers
install(DIRECTORY corec/node/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/corec/node
    FILES_MATCHING PATTERN "*.h"
)
install(DIRECTORY corec/array/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/corec/array
    FILES_MATCHING PATTERN "*.h"
)
install(DIRECTORY corec/str/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/corec/str
    FILES_MATCHING PATTERN "*.h"
)
install(DIRECTORY corec/helpers/parser/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/corec/helpers/parser
    FILES_MATCHING PATTERN "*.h"
)
install(DIRECTORY corec/helpers/file/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/corec/helpers/file
    FILES_MATCHING PATTERN "*.h"
)
install(DIRECTORY corec/helpers/date/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/corec/helpers/date
    FILES_MATCHING PATTERN "*.h"
)
install(DIRECTORY corec/helpers/charconvert/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/corec/helpers/charconvert
    FILES_MATCHING PATTERN "*.h"
)

# Install documentation
install(FILES corec/node/corec.html
    DESTINATION ${CMAKE_INSTALL_DOCDIR}
    OPTIONAL
)

# Generate and install CMake package configuration files
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/corecConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/corecConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/corecConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/corec
)

install(EXPORT corecTargets
    FILE corecTargets.cmake
    NAMESPACE corec::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/corec
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/corecConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/corecConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/corec
)
