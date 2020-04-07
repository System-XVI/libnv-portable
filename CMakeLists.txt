cmake_minimum_required(VERSION 3.14.6)
cmake_policy(VERSION 3.14.6)

project(libnv-portable VERSION 1.0.0 DESCRIPTION "A cross-platform userspace version of libnv, ported from FreeBSD")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")

find_package(Atf-C)
if(NOT Atf-C_FOUND)
    message("ATF not found; will not build ATF tests")
    set(TEST_ATF FALSE)
else()
    set(TEST_ATF TRUE)
endif()

# Set output directory
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY bin)

# Set options
option(TEST "Enable unit tests" OFF)
option(DEBUG "Build without optimization and include debugging symbols" OFF)

add_library(nvp SHARED)
target_sources(nvp
    PRIVATE
        "src/cnvlist.c"
        "src/dnvlist.c"
        "src/msgio.c"
        "src/nvlist.c"
        "src/nvpair.c"
)

# Set include directories
target_include_directories(nvp PUBLIC include)
target_include_directories(nvp PRIVATE src)

# Set library version
set_target_properties(nvp PROPERTIES VERSION ${PROJECT_VERSION})
set_target_properties(nvp PROPERTIES SOVERSION ${PROJECT_VERSION_MAJOR})

# Set compiler/language options
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_C_FLAGS_RELEASE "-O2 -D_DEFAULT_SOURCE -D_XOPEN_SOURCE=600")
set(CMAKE_C_FLAGS_DEBUG "-O0 -g3 -D_DEFAULT_SOURCE -D_XOPEN_SOURCE=600")

# If we're on Linux, link against libbsd
if(CMAKE_SYSTEM_NAME MATCHES ".*Linux")
    set(LINUX TRUE)
    target_link_libraries(nvp bsd)
endif()

function(build_test target is_atf)
    if (NOT (is_atf AND NOT TEST_ATF))
        add_executable(${target} tests/${target}.c)
        target_include_directories(${target} PRIVATE include)
        target_link_directories(${target} PRIVATE bin)
        target_link_libraries(${target} nvp)
        if(LINUX)
            target_link_libraries(${target} bsd)
        endif()
        if(is_atf)
            target_link_libraries(${target} Atf-C::Atf-C)
        endif()
    #add_custom_command(TARGET ${target} POST_BUILD COMMAND ./bin/test/${target})
    endif()
endfunction()

# If tests are enabled, make tests
if(TEST)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY bin/test)

    build_test(nvlist_add_test FALSE)
    build_test(nvlist_append_test FALSE)
    build_test(nvlist_exists_test FALSE)
    build_test(nvlist_free_test FALSE)
    build_test(nvlist_get_test FALSE)
    build_test(nvlist_move_test FALSE)
    build_test(nvlist_send_recv_test TRUE)
endif()
