cmake_minimum_required(VERSION 2.8.3)
project(serial)

if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "No build type selected, default to Release")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
endif()

if (NOT CMAKE_BUILD_TARGET)
    message(STATUS "No target type selected, default to both shared and static libraries")
    set(CMAKE_BUILD_TARGET "Both" CACHE STRING "" FORCE)
endif()

option(BUILD_EXAMPLE "Build example program" OFF)
option(BUILD_TEST "Build test program" OFF)

set(${PROJECT_NAME}_VERSION_MAJOR 1)
set(${PROJECT_NAME}_VERSION_MINOR 2)
set(${PROJECT_NAME}_VERSION_PATCH 2)
set(${PROJECT_NAME}_VERSION ${${PROJECT_NAME}_VERSION_MAJOR}.${${PROJECT_NAME}_VERSION_MINOR}.${${PROJECT_NAME}_VERSION_PATCH})

set(${PROJECT_NAME}_DESCRIPTION "A cross-platform library for serial communication")
set(${PROJECT_NAME}_URL "https://github.com/WHILL/serial")

if(APPLE)
    find_library(IOKIT_LIBRARY IOKit)
    find_library(FOUNDATION_LIBRARY Foundation)
endif()

if(UNIX AND NOT APPLE)
    find_library(rt_LIBRARY rt)
    find_library(pthread_LIBRARY pthread)
else()
    # Otherwise normal call
endif()

message(STATUS "CMAKE_CURRENT_SOURCE_DIR=${CMAKE_CURRENT_SOURCE_DIR}")
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

## Sources
set(serial_SRCS
    src/serial.cc
)

if(APPLE)
    # If OSX
    list(APPEND serial_SRCS src/impl/unix.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_osx.cc)
elseif(UNIX)
    # If unix
    list(APPEND serial_SRCS src/impl/unix.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_linux.cc)
else()
    # If windows
    list(APPEND serial_SRCS src/impl/win.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_win.cc)
endif()

if(${CMAKE_BUILD_TARGET} MATCHES "Shared")
    set(BUILD_SERIAL_SHARED TRUE)
elseif(${CMAKE_BUILD_TARGET} MATCHES "Static")
    set(BUILD_SERIAL_STATIC TRUE)
elseif(${CMAKE_BUILD_TARGET} MATCHES "Both")
    set(BUILD_SERIAL_SHARED TRUE)
    set(BUILD_SERIAL_STATIC TRUE)
else()
    message(FATAL_ERROR "Invalid build type ${CMAKE_BUILD_TYPE}")
endif()

## Add serial library
if(BUILD_SERIAL_SHARED)
    add_library(${PROJECT_NAME} SHARED ${serial_SRCS})
    set_property(TARGET ${PROJECT_NAME} PROPERTY VERSION ${${PROJECT_NAME}_VERSION})
    set_property(TARGET ${PROJECT_NAME} PROPERTY SOVERSION ${${PROJECT_NAME}_VERSION_MAJOR})
    list(APPEND ${PROJECT_NAME}_TARGETS ${PROJECT_NAME})

    if(APPLE)
        target_link_libraries(${PROJECT_NAME} ${FOUNDATION_LIBRARY} ${IOKIT_LIBRARY})
    elseif(UNIX)
        target_link_libraries(${PROJECT_NAME} ${rt_LIBRARY} ${pthread_LIBRARY})
    else()
        target_link_libraries(${PROJECT_NAME} setupapi)
    endif()
endif()

if(BUILD_SERIAL_STATIC)
    add_library(${PROJECT_NAME}_static STATIC ${serial_SRCS})
    set_target_properties(${PROJECT_NAME}_static PROPERTIES OUTPUT_NAME ${PROJECT_NAME})
    list(APPEND ${PROJECT_NAME}_TARGETS ${PROJECT_NAME}_static)

    if(APPLE)
        target_link_libraries(${PROJECT_NAME}_static ${FOUNDATION_LIBRARY} ${IOKIT_LIBRARY})
    elseif(UNIX)
        target_link_libraries(${PROJECT_NAME}_static rt pthread)
    else()
        target_link_libraries(${PROJECT_NAME}_static setupapi)
    endif()
endif()


## Uncomment for example
if(BUILD_EXAMPLE)
add_executable(serial_example examples/serial_example.cc)
add_dependencies(serial_example ${PROJECT_NAME})
target_link_libraries(serial_example ${PROJECT_NAME})
endif()


## Tests
if(BUILD_TEST)
    add_subdirectory(tests)
endif()

set(CONF_INCLUDE_DIR "${CMAKE_INSTALL_PREFIX}/include")
set(CONF_LIBRARY_DIR "${CMAKE_INSTALL_PREFIX}/lib/${CMAKE_LIBRARY_ARCHITECTURE}")
set(CONF_LIBRARIES   "${CONF_LIBRARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}serial${CMAKE_SHARED_LIBRARY_SUFFIX}")
configure_file(${PROJECT_NAME}Config.cmake.in ${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}Config.cmake @ONLY)
configure_file(${PROJECT_NAME}.pc.in ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.pc @ONLY)

message(STATUS "Targets will be installed...")
message(STATUS "CMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}")
message(STATUS "CMAKE_LIBRARY_ARCHITECTURE=${CMAKE_LIBRARY_ARCHITECTURE}")

install(TARGETS ${${PROJECT_NAME}_TARGETS}
        EXPORT ${PROJECT_NAME}-export
        LIBRARY DESTINATION "lib/${CMAKE_LIBRARY_ARCHITECTURE}"
        ARCHIVE DESTINATION "lib/${CMAKE_LIBRARY_ARCHITECTURE}"
        PUBLIC_HEADER DESTINATION "include/${PROJECT_NAME}"
)

install(DIRECTORY include/serial/
    DESTINATION "include/${PROJECT_NAME}"
)

install(FILES "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}Config.cmake"
    DESTINATION "lib/cmake/${PROJECT_NAME}"
)

install(EXPORT ${PROJECT_NAME}-export
    FILE "${PROJECT_NAME}Config-version.cmake"
    DESTINATION "lib/cmake/${PROJECT_NAME}"
)

install(FILES
    "${PROJECT_BINARY_DIR}/${PROJECT_NAME}.pc"
    DESTINATION "lib/pkgconfig"
)


