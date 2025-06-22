cmake_minimum_required(VERSION 3.16)
project(FirewallProject VERSION 0.1.0 LANGUAGES C)

# Set C standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

# Compiler flags for SIMD and performance
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror -O3 -march=native -mtune=native")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse -msse2 -msse4.2 -mavx -mavx2")
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g -O0 -DDEBUG")

# Optional AVX-512 (disabled by default, enabled if supported)
option(ENABLE_AVX512 "Enable AVX-512 instructions if supported" OFF)
if(ENABLE_AVX512)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mavx512f -mavx512cd -mavx512bw -mavx512dq -mavx512vl")
endif()

# Find required libraries
find_package(Threads REQUIRED)

# Define directories
set(SOURCE_DIR "${CMAKE_SOURCE_DIR}/src")
set(INCLUDE_DIR "${CMAKE_SOURCE_DIR}/include")
set(CONFIG_DIR "${CMAKE_SOURCE_DIR}/config")
set(LOG_DIR "${CMAKE_SOURCE_DIR}/logs")

# Collect source files
set(SOURCES
    ${SOURCE_DIR}/main.c
    ${SOURCE_DIR}/packet_capture.c
    ${SOURCE_DIR}/packet_processing.c
    ${SOURCE_DIR}/rule_engine.c
    ${SOURCE_DIR}/ips_ids.c
    ${SOURCE_DIR}/dpi.c
    ${SOURCE_DIR}/ml_module.c
    ${SOURCE_DIR}/logging.c
    ${SOURCE_DIR}/management.c
)

# Create executable
add_executable(firewall ${SOURCES})

# Include directories
target_include_directories(firewall PRIVATE ${INCLUDE_DIR})

# Link libraries
target_link_libraries(firewall PRIVATE Threads::Threads)

# Check for SIMD support at compile time
include(CheckCCompilerFlag)
check_c_compiler_flag("-mavx512f" HAS_AVX512)
if(HAS_AVX512 AND ENABLE_AVX512)
    target_compile_definitions(firewall PRIVATE USE_AVX512)
endif()
check_c_compiler_flag("-mavx2" HAS_AVX2)
if(HAS_AVX2)
    target_compile_definitions(firewall PRIVATE USE_AVX2)
endif()
check_c_compiler_flag("-mavx" HAS_AVX)
if(HAS_AVX)
    target_compile_definitions(firewall PRIVATE USE_AVX)
endif()
check_c_compiler_flag("-msse4.2" HAS_SSE42)
if(HAS_SSE42)
    target_compile_definitions(firewall PRIVATE USE_SSE42)
endif()

# Installation rules
install(TARGETS firewall
        RUNTIME DESTINATION bin
        PERMISSIONS OWNER_EXECUTE OWNER_READ GROUP_EXECUTE GROUP_READ WORLD_EXECUTE WORLD_READ)

install(DIRECTORY ${CONFIG_DIR}/
        DESTINATION /etc/firewall-project
        FILES_MATCHING PATTERN "*.conf"
        PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ)

install(DIRECTORY
        DESTINATION /var/log/firewall-project
        DIRECTORY_PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ GROUP_WRITE WORLD_READ)

# Custom target for cleaning logs
add_custom_target(clean-logs
    COMMAND ${CMAKE_COMMAND} -E remove_directory ${LOG_DIR}/*
    COMMAND ${CMAKE_COMMAND} -E touch ${LOG_DIR}/.gitkeep
    COMMENT "Cleaning log files"
)

# Enable testing
enable_testing()
add_test(NAME VersionTest COMMAND firewall --version)

# Build type configuration
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")

# Status messages
message(STATUS "Configuring FirewallProject v${PROJECT_VERSION} for ${CMAKE_SYSTEM_NAME}")
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "SIMD flags: ${CMAKE_C_FLAGS}")
if(ENABLE_AVX512 AND HAS_AVX512)
    message(STATUS "AVX-512 enabled")
endif()
message(STATUS "Install prefix: ${CMAKE_INSTALL_PREFIX}")