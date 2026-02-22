project("ebml2" VERSION 0.21.3 LANGUAGES C)

option(CONFIG_EBML_WRITING "Build libebml2 with write support" ON)

set(ebml2_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlmain.c
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlbinary.c
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlcrc.c
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmldate.c
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlelement.c
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlmaster.c
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlnumber.c
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlstring.c
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlvoid.c
)
set(ebml2_PUBLIC_HEADERS
  ${CMAKE_CURRENT_SOURCE_DIR}/ebml2/ebml.h
  ${CMAKE_CURRENT_SOURCE_DIR}/ebml2/ebml_classes.h
  ${CMAKE_CURRENT_BINARY_DIR}/ebml2/ebml_config.h
)
set(ebml2_INTERNAL_HEADERS
  ${CMAKE_CURRENT_SOURCE_DIR}/internal.h
  ${CMAKE_CURRENT_SOURCE_DIR}/ebmlcrc.h
)
add_library("ebml2")
add_library(ebml2::ebml2 ALIAS ebml2)
target_sources("ebml2" PRIVATE
  ${ebml2_SOURCES} 
  ${ebml2_PUBLIC_HEADERS} 
  ${ebml2_INTERNAL_HEADERS}
)
target_compile_definitions("ebml2" PRIVATE EBML2_UGLY_HACKS_API)
target_compile_definitions("ebml2" PUBLIC HAS_EBML2)
configure_file(ebml_config.h.in ebml2/ebml_config.h)
set_target_properties("ebml2" PROPERTIES 
  PUBLIC_HEADER "${ebml2_PUBLIC_HEADERS}"
  C_STANDARD 11
)

# the library needs to be static as it shares constants variables
target_link_libraries("ebml2" PUBLIC corec::corec)

target_include_directories("ebml2"
  PRIVATE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
# keep until the sources are split
target_include_directories("ebml2" PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)


# we can't export EBML as a DLL for now as it doesn't seem possible to use dllimport to initialize global (startup) variables
if (BUILD_SHARED_LIBS)
  target_compile_definitions("ebml2" PUBLIC ebml2_IMPORTS)
endif (BUILD_SHARED_LIBS)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Generate package config file
configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/ebml2Config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/ebml2Config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/ebml2
)

write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/ebml2ConfigVersion.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion
)

install(TARGETS "ebml2"
  EXPORT ebml2Targets
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/ebml2
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

# Install CMake package configuration files
install(EXPORT ebml2Targets
  FILE ebml2Targets.cmake
  NAMESPACE ebml2::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/ebml2)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/ebml2Config.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/ebml2ConfigVersion.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/ebml2)


if (MATROSKA2_TOOLS)
  add_executable("ebmltree" test/ebmltree.c)
  target_link_libraries("ebmltree" PUBLIC "ebml2" "corec")
endif ()
