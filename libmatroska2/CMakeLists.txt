project("matroska2" VERSION 0.22.3 LANGUAGES C)

option(CONFIG_ZLIB "Enable zlib (de)compression" ON)
option(CONFIG_BZLIB "Enable bzlib decompression in libmatroska2" ON)
option(CONFIG_LZO1X "Enable lzo decompression in libmatroska2" ON)
option(CONFIG_CODEC_HELPER "Enable Vorbis frame durations in libmatroska2" ON)

if (CONFIG_ZLIB AND NOT TARGET ZLIB::ZLIB)
  include(FindZLIB)

  if(NOT ZLIB_FOUND)
    include(FetchContent)

    FetchContent_Declare(
        zlib
        GIT_REPOSITORY https://github.com/Nelonn/zlib-ng
        GIT_TAG b583a325937227d7834ae541cf88c1f273da113e
    )

    FetchContent_MakeAvailable(zlib)
    target_compile_definitions(zlib-ng PRIVATE NO_GZIP)
  endif()
endif()

set(matroska2_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/matroskamain.c
  ${CMAKE_CURRENT_SOURCE_DIR}/matroskablock.c
  ${CMAKE_CURRENT_SOURCE_DIR}/matroska_sem.c
)
set(matroska2_PUBLIC_HEADERS
  ${CMAKE_CURRENT_SOURCE_DIR}/matroska2/matroska.h
  ${CMAKE_CURRENT_SOURCE_DIR}/matroska2/matroska_classes.h
  ${CMAKE_CURRENT_SOURCE_DIR}/matroska2/matroska_sem.h
  ${CMAKE_CURRENT_BINARY_DIR}/matroska2/matroska_config.h
)

add_library("matroska2")
add_library(matroska2::matroska2 ALIAS matroska2)
target_sources("matroska2" PRIVATE ${matroska2_SOURCES} ${matroska2_PUBLIC_HEADERS})

target_include_directories(matroska2
        PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
# keep until the sources are split
target_include_directories("matroska2" PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)

if (CONFIG_ZLIB)
  target_link_libraries("matroska2" PRIVATE ZLIB::ZLIB)
endif(CONFIG_ZLIB)

if (CONFIG_LZO1X)
  add_subdirectory("lzokay")
  target_link_libraries("matroska2" PRIVATE $<BUILD_INTERFACE:lzokay>)
endif(CONFIG_LZO1X)

if (CONFIG_BZLIB)
  find_package (BZip2)
  if(BZIP2_FOUND)
    target_include_directories("matroska2" PRIVATE ${BZIP2_INCLUDE_DIRS})
    target_link_libraries("matroska2" PRIVATE ${BZIP2_LIBRARIES})
  else()
    add_subdirectory("bzip2")
    target_link_libraries("matroska2" PRIVATE $<BUILD_INTERFACE:bzlib>)
  endif()
endif(CONFIG_BZLIB)

if (CONFIG_CODEC_HELPER)
  add_subdirectory("tremor" EXCLUDE_FROM_ALL)
  target_link_libraries("matroska2" PRIVATE $<BUILD_INTERFACE:tremor>)
endif(CONFIG_CODEC_HELPER)

set_target_properties("matroska2" PROPERTIES
  PUBLIC_HEADER "${matroska2_PUBLIC_HEADERS}"
  C_STANDARD 11
)
configure_file(matroska_config.h.in matroska2/matroska_config.h)

# the library needs to be static as it shares constants variables
target_link_libraries("matroska2" PUBLIC ebml2::ebml2)
if (BUILD_SHARED_LIBS)
  target_compile_definitions("matroska2" INTERFACE matroska2_IMPORTS)
endif (BUILD_SHARED_LIBS)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

configure_package_config_file(
        matroska2Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/matroska2Config.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/matroska2
)

write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/matroska2ConfigVersion.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
)

install(TARGETS matroska2
        EXPORT matroska2Targets
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/matroska2
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(EXPORT matroska2Targets
        FILE matroska2Targets.cmake
        NAMESPACE matroska2::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/matroska2
)

install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/matroska2Config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/matroska2ConfigVersion.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/matroska2
)

if (MATROSKA2_TOOLS)
  add_executable("mkvtree" test/mkvtree.c)
  target_link_libraries("mkvtree" PRIVATE "matroska2" "ebml2" "corec")
endif ()
