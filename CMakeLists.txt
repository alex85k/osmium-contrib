#----------------------------------------------------------------------
#
#  Osmium Tool CMakeLists.txt
#
#----------------------------------------------------------------------

cmake_minimum_required(VERSION 2.8.5)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")


#----------------------------------------------------------------------
#
#  Project version
#
#----------------------------------------------------------------------

project(osmium)

set(OSMIUM_VERSION_MAJOR 0)
set(OSMIUM_VERSION_MINOR 0)
set(OSMIUM_VERSION_PATCH 1)

set(OSMIUM_VERSION ${OSMIUM_VERSION_MAJOR}.${OSMIUM_VERSION_MINOR}.${OSMIUM_VERSION_PATCH})


#----------------------------------------------------------------------
#
#  Find external dependencies
#
#----------------------------------------------------------------------
find_package(Threads REQUIRED)
list(APPEND LIBRARIES ${CMAKE_THREAD_LIBS_INIT})
if(NOT THREADS_FOUND)
    message(WARNING "Threads library not found!\n")
    set(MISSING_LIB 1)
endif()

find_package(OSMPBF REQUIRED)
include_directories(${OSMPBF_INCLUDE_DIRS})
list(APPEND LIBRARIES ${OSMPBF_LIBRARIES})
if(NOT OSMPBF_FOUND)
    message(WARNING "OSMPBF library not found!\n")
    set(MISSING_LIB 1)
endif()

find_package(Protobuf REQUIRED)
include_directories(${PROTOBUF_INCLUDE_DIRS})
list(APPEND LIBRARIES ${PROTOBUF_LITE_LIBRARIES})
if(NOT PROTOBUF_FOUND)
    message(WARNING "Protobuf library not found!\n")
    set(MISSING_LIB 1)
endif()

find_package(Boost COMPONENTS program_options)
include_directories(${Boost_INCLUDE_DIRS})
list(APPEND LIBRARIES ${Boost_LIBRARIES})
if(NOT Boost_FOUND)
    message(WARNING "Boost program_options not found!\n")
endif()

find_package(Osmium REQUIRED)
include_directories(${OSMIUM_INCLUDE_DIRS})
if(NOT OSMIUM_FOUND)
    message(WARNING "Libosmium not found!\n")
    set(MISSING_LIB 1)
endif()

find_package(GDAL)
include_directories(${GDAL_INCLUDE_DIR})
list(APPEND LIBRARIES ${GDAL_LIBRARY})
if(NOT GDAL_FOUND)
   message(WARNING "GDAL not found, will not build node_density and mapolution!\n")
endif()

##### Find Proj.4 Library
find_path(PROJ_INCLUDE_DIR proj_api.h)
find_library(PROJ_LIBRARY NAMES proj)
if(PROJ_INCLUDE_DIR AND PROJ_LIBRARY)
    message(STATUS "PROJ library found")
    set(PROJ_FOUND 1)
    include_directories(${PROJ_INCLUDE_DIR})
    list(APPEND LIBRARIES ${PROJ_LIBRARY})
else()
    message(WARNING "PROJ not found, will not build node_density and mapolution!\n")
endif()

find_package(ZLIB REQUIRED)
include_directories(${ZLIB_INCLUDE_DIRS})
list(APPEND LIBRARIES ${ZLIB_LIBRARIES})
if(NOT ZLIB_FOUND)
    message(WARNING "Zlib not found!\n")
    set(MISSING_LIB 1)
endif()

find_package(BZip2 REQUIRED)
include_directories(${BZIP2_INCLUDE_DIRS})
list(APPEND LIBRARIES ${BZIP2_LIBRARIES})
if(NOT BZIP2_FOUND)
    message(WARNING "BZip2 library not found!\n")
    set(MISSING_LIB 1)
endif()

find_package(EXPAT REQUIRED)
include_directories(${EXPAT_INCLUDE_DIRS})
list(APPEND LIBRARIES ${EXPAT_LIBRARIES})
if(NOT EXPAT_FOUND)
    message(WARNING "Expat library not found!\n")
    set(MISSING_LIB 1)
endif()

#----------------------------------------------------------------------
#
#  Check that all required libraries are available
#
#----------------------------------------------------------------------
if(MISSING_LIB)
    message(FATAL_ERROR "Required library or libraries missing. Aborting...")
endif(MISSING_LIB)


#----------------------------------------------------------------------
#
#  Optional "cppcheck" target that checks C++ code
#
#----------------------------------------------------------------------
message(STATUS "Looking for cppcheck")
find_program(CPPCHECK cppcheck)

if(CPPCHECK)
    message(STATUS "Looking for cppcheck - found")
    set(CPPCHECK_OPTIONS --enable=warning,style,performance,portability,information,missingInclude)

    # cpp doesn't find system includes for some reason, suppress that report
    set(CPPCHECK_OPTIONS ${CPPCHECK_OPTIONS} --suppress=missingIncludeSystem)

    add_custom_target(cppcheck ${CPPCHECK} --std=c++11 ${CPPCHECK_OPTIONS} ${CMAKE_SOURCE_DIR}/src/*pp)
else()
    message(STATUS "Looking for cppcheck - not found")
    message(STATUS "  Make target cppcheck not available")
endif(CPPCHECK)

#----------------------------------------------------------------------
#
#  C++ compilation options
#
#----------------------------------------------------------------------
add_definitions(-D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64)

if(MSVC)
    add_definitions(-wd4996 -DNOMINMAX -DWIN32_LEAN_AND_MEAN -D_CRT_SECURE_NO_WARNINGS)
else()
    add_compile_options(-std=c++11)
    add_compile_options(
        -Wall
        -Wextra
        -pedantic
        -Wredundant-decls
        -Wdisabled-optimization
        -Wctor-dtor-privacy
        -Wnon-virtual-dtor
        -Woverloaded-virtual
        -Wsign-promo
        -Wold-style-cast
    )
    #always compile with optimization, even in debug mode
    set(CMAKE_CXX_FLAGS_DEBUG -O3)
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O3 -g")
endif()

if (WIN32)
    list(APPEND LIBRARIES ws2_32)
endif()

if(APPLE)
    add_compile_options(-stdlib=libc++)
    set(LDFLAGS ${LDFLAGS} -stdlib=libc++)
endif(APPLE)

macro(COMPILE_PROGRAM PROG)
  file(GLOB SOURCES ${PROG}/*.cpp ${PROG}/*.hpp)
  add_executable(${PROG} ${SOURCES})
  target_link_libraries(${PROG} ${LIBRARIES})
endmacro()


COMPILE_PROGRAM(pub_names)
COMPILE_PROGRAM(road_length)
COMPILE_PROGRAM(export_to_wkt)
if (Boost_FOUND AND GDAL_FOUND AND PROJ_FOUND)
  COMPILE_PROGRAM(node_density)
  COMPILE_PROGRAM(mapolution)
endif()

