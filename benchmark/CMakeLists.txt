############################################################################
# Copyright (c) 2016, Johan Mabille, Sylvain Corlay and Wolf Vollprecht    #
#                                                                          #
# Distributed under the terms of the BSD 3-Clause License.                 #
#                                                                          #
# The full license is in the file LICENSE, distributed with this software. #
############################################################################

cmake_minimum_required(VERSION 3.29)

if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    project(xtensor-benchmark)

    find_package(xtensor REQUIRED CONFIG)
    set(XTENSOR_INCLUDE_DIR ${xtensor_INCLUDE_DIRS})
endif ()

message(STATUS "Forcing tests build type to Release")
set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)

include(CheckCXXCompilerFlag)

string(TOUPPER "${CMAKE_BUILD_TYPE}" U_CMAKE_BUILD_TYPE)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Intel")
    CHECK_CXX_COMPILER_FLAG(-march=native arch_native_supported)
    if(arch_native_supported AND NOT CMAKE_CXX_FLAGS MATCHES "-march")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
    endif()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wunused-parameter -Wextra -Wreorder")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")

    # Enable link time optimization and set the default symbol
    # visibility to hidden (very important to obtain small binaries)
    if (NOT ${U_CMAKE_BUILD_TYPE} MATCHES DEBUG)
        # Default symbol visibility
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden")

        # Check for Link Time Optimization support
        # (GCC/Clang)
        # LTO had to be removed as google benchmark doesn't build with it
        # CHECK_CXX_COMPILER_FLAG("-flto" HAS_LTO_FLAG)
        # if (HAS_LTO_FLAG)
        #     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto")
        # endif()

        # Intel equivalent to LTO is called IPO
        if (CMAKE_CXX_COMPILER_ID MATCHES "Intel")
            CHECK_CXX_COMPILER_FLAG("-ipo" HAS_IPO_FLAG)
            if (HAS_IPO_FLAG)
                set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ipo")
            endif()
        endif()
    endif()
endif()

if(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc /MP /bigobj")
    set(CMAKE_EXE_LINKER_FLAGS /MANIFEST:NO)
    foreach(flag_var
            CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
            CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
        string(REPLACE "/MD" "-MT" ${flag_var} "${${flag_var}}")
    endforeach()
endif()


if(DOWNLOAD_GBENCHMARK OR GBENCHMARK_SRC_DIR)
    if(DOWNLOAD_GBENCHMARK)
        # Download and unpack googlebenchmark at configure time
        configure_file(downloadGBenchmark.cmake.in googlebenchmark-download/CMakeLists.txt)
    else()
        # Copy local source of googlebenchmark at configure time
        configure_file(copyGBenchmark.cmake.in googlebenchmark-download/CMakeLists.txt)
    endif()
    execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
                    RESULT_VARIABLE result
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googlebenchmark-download )
    if(result)
        message(FATAL_ERROR "CMake step for googlebenchmark failed: ${result}")
    endif()
    execute_process(COMMAND ${CMAKE_COMMAND} --build .
                    RESULT_VARIABLE result
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googlebenchmark-download )
    if(result)
        message(FATAL_ERROR "Build step for googlebenchmark failed: ${result}")
    endif()

    # Add googlebenchmark directly to our build. This defines
    # the gtest and gtest_main targets.
    add_subdirectory(${CMAKE_CURRENT_BINARY_DIR}/googlebenchmark-src
                     ${CMAKE_CURRENT_BINARY_DIR}/googlebenchmark-build)

    set(GBENCHMARK_INCLUDE_DIRS "${googlebenchmark_SOURCE_DIR}/include")
    set(GBENCHMARK_LIBRARIES benchmark)
else()
    find_package(benchmark REQUIRED)
endif()


message(STATUS "BLAS VENDOR:    " ${BLA_VENDOR})
message(STATUS "BLAS LIBRARIES: " ${BLAS_LIBRARIES})

if (HAVE_CBLAS)
    if (WIN32)
        find_package(OpenBLAS REQUIRED)
        set(BLAS_LIBRARIES ${CMAKE_INSTALL_PREFIX}${OpenBLAS_LIBRARIES})
    else()
        find_package(BLAS REQUIRED)
        find_package(LAPACK REQUIRED)
    endif()
    add_definitions(-DHAVE_CBLAS=1 -DWITH_OPENBLAS=1)
endif()

include_directories(${XTENSOR_INCLUDE_DIR} ${GBENCHMARK_INCLUDE_DIRS})

set(XTENSOR_BENCHMARK
    main.cpp
    benchmark_blas.hpp
)

set(XTENSOR_BENCHMARK_TARGET benchmark_xtensor)
add_executable(${XTENSOR_BENCHMARK_TARGET} EXCLUDE_FROM_ALL ${XTENSOR_BENCHMARK} ${XTENSOR_HEADERS})
target_link_libraries(${XTENSOR_BENCHMARK_TARGET} ${BLAS_LIBRARIES} ${LAPACK_LIBRARIES} ${GBENCHMARK_LIBRARIES})

add_custom_target(xbenchmark
    COMMAND benchmark_xtensor
    DEPENDS ${XTENSOR_BENCHMARK_TARGET})

add_custom_target(xpowerbench
    COMMAND echo "sudo needed to set cpu power governor to performance"
    COMMAND sudo cpupower frequency-set --governor performance
    COMMAND benchmark_xtensor
    COMMAND sudo cpupower frequency-set --governor powersave
    DEPENDS ${XTENSOR_BENCHMARK_TARGET})
