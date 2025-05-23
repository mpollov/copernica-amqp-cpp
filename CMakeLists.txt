# Builds AMQP-CPP
#
# Options:
#
# - AMQP-CPP_BUILD_SHARED (default OFF)
#       ON:  Build shared lib
#       OFF: Build static lib
#
# - AMQP-CPP_LINUX_TCP (default OFF)
#       ON:  Build posix handler implementation
#       OFF: Don't build posix handler implementation

cmake_minimum_required(VERSION 3.4 FATAL_ERROR)

# project name
project(amqpcpp)
set (VERSION_MAJOR 4)
set (VERSION_MINOR 3)
set (VERSION_PATCH 27)
set (SO_VERSION ${VERSION_MAJOR}.${VERSION_MINOR})

# build options
option(AMQP-CPP_BUILD_SHARED "Build shared library. If off, build will be static." OFF)
option(AMQP-CPP_LINUX_TCP "Build linux sockets implementation." OFF)
option(AMQP-CPP_BUILD_EXAMPLES "Build amqpcpp examples" OFF)

# pass version number to source files as macro
add_compile_definitions(VERSION=${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})

# ensure c++11 on all compilers
set (CMAKE_CXX_STANDARD 17)

# add source files
# ------------------------------------------------------------------------------------------------------

# set include/ as include directory
include_directories(SYSTEM ${CMAKE_CURRENT_SOURCE_DIR}/include)

# macro that adds a list of provided source files to a list called SRCS.
# if variable SRCS does not yet exist, it is created.
macro (add_sources)
    file (RELATIVE_PATH _relPath "${PROJECT_SOURCE_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}")
    foreach (_src ${ARGN})
        if (_relPath)
            list (APPEND SRCS "${_relPath}/${_src}")
        else()
            list (APPEND SRCS "${_src}")
        endif()
    endforeach()
    if (_relPath)
        # propagate SRCS to parent directory
        set (SRCS ${SRCS} PARENT_SCOPE)
    endif()
endmacro()

# add source files
#add_subdirectory(src)
aux_source_directory(src src_MAIN)
if(AMQP-CPP_LINUX_TCP)
    #add_subdirectory(src/linux_tcp)
    aux_source_directory(src/linux_tcp src_LINUX_TCP)
endif()

# potentially build the examples
if(AMQP-CPP_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

# settings for specific compilers
# ------------------------------------------------------------------------------------------------------

# we have to prevent windows from defining the max macro.
if (WIN32)
    add_definitions(-DNOMINMAX)
endif()

# build targets
# ------------------------------------------------------------------------------------------------------

# set output directory
set(LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_BINARY_DIR}/bin)

if(AMQP-CPP_BUILD_SHARED)
    # create shared lib
    #add_library(${PROJECT_NAME} SHARED ${SRCS})
    add_library(${PROJECT_NAME} SHARED ${src_MAIN} ${src_LINUX_TCP})
    set_target_properties(${PROJECT_NAME} PROPERTIES
        # set shared lib version
        SOVERSION ${SO_VERSION}
        # export symbols for Visual Studio as a workaround
        WINDOWS_EXPORT_ALL_SYMBOLS ON
    )
else()
    # create static lib
    #add_library(${PROJECT_NAME} STATIC ${SRCS})
    add_library(${PROJECT_NAME} STATIC ${src_MAIN} ${src_LINUX_TCP})
endif()

if(WIN32)
    target_link_libraries(${PROJECT_NAME} PUBLIC ws2_32)
endif()

# install rules
# ------------------------------------------------------------------------------------------------------

include(GNUInstallDirs)
install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Config
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# copy header files
install(DIRECTORY include/amqpcpp/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/amqpcpp
        FILES_MATCHING PATTERN "*.h")
install(FILES include/amqpcpp.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(EXPORT ${PROJECT_NAME}Config DESTINATION cmake)
export(TARGETS ${PROJECT_NAME} FILE ${PROJECT_NAME}Config.cmake)

set(DEST_DIR "${CMAKE_INSTALL_PREFIX}")
set(PRIVATE_LIBS "-llibamqpcc")
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/amqpcpp.pc.in"
               "${CMAKE_CURRENT_BINARY_DIR}/amqpcpp.pc" @ONLY)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/amqpcpp.pc" DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

# submodule support
# ------------------------------------------------------------------------------------------------------
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/>
    $<INSTALL_INTERFACE:include/>
)

if(AMQP-CPP_LINUX_TCP)
    target_link_libraries(${PROJECT_NAME} ${CMAKE_DL_LIBS})
    # Find OpenSSL and provide include dirs
    find_package(OpenSSL REQUIRED)
    target_include_directories(${PROJECT_NAME} PRIVATE ${OPENSSL_INCLUDE_DIR})
endif()
