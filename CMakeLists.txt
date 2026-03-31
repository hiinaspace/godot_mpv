cmake_minimum_required(VERSION 3.17)

# Make sure all the dependencies are satisfied
find_package(Python3 3.4 REQUIRED)
find_program(GIT git REQUIRED)

# Ensure godot-cpp submodule has been initialized
if(NOT EXISTS "${CMAKE_CURRENT_LIST_DIR}/godot-cpp/src")
    message(NOTICE "godot-cpp bindings source not found")
    message(NOTICE "initializing/updating the godot-cpp submodule...")
    execute_process(
        COMMAND git submodule update --init godot-cpp
        WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
        COMMAND_ERROR_IS_FATAL ANY
    )
endif()
add_subdirectory(godot-cpp SYSTEM)

# Add godot-cpp's module path and include the exported functions
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${godot-cpp_SOURCE_DIR}/cmake")
include(GodotCPPModule)

# Retrieve platform and suffix info from godot-cpp target
get_target_property(GODOTCPP_SUFFIX godot::cpp GODOTCPP_SUFFIX)
get_target_property(GODOTCPP_PLATFORM godot::cpp GODOTCPP_PLATFORM)

project(godot_mpv)

# GitVersion integration (keep as-is)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}")
include(GitVersion)
get_version_from_git()
configure_file(
    ${CMAKE_SOURCE_DIR}/native/src/godot_mpv/include/version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/native/src/godot_mpv/include/version.h
)

# GLAD targets — STATIC on Windows, OBJECT on Linux (embedded into main .so)
if(WIN32)
    add_library(glad_gles2 STATIC
        "${CMAKE_SOURCE_DIR}/dependencies/glad/src/gles2.c"
    )
    add_library(glad_egl STATIC
        "${CMAKE_SOURCE_DIR}/dependencies/glad/src/egl.c"
    )
else()
    add_library(glad_gles2 OBJECT
        "${CMAKE_SOURCE_DIR}/dependencies/glad/src/gles2.c"
    )
    add_library(glad_egl OBJECT
        "${CMAKE_SOURCE_DIR}/dependencies/glad/src/egl.c"
    )
    set_target_properties(glad_gles2 glad_egl PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif()

target_include_directories(glad_gles2 PUBLIC
    "${CMAKE_SOURCE_DIR}/dependencies/glad/include"
)
target_include_directories(glad_egl PUBLIC
    "${CMAKE_SOURCE_DIR}/dependencies/glad/include"
)

# Source files
file(GLOB_RECURSE SOURCES CONFIGURE_DEPENDS
    native/src/godot_mpv/*.cpp
    native/src/godot_mpv/*.h
)

# Main shared library
add_library(godot_mpv SHARED ${SOURCES})

target_compile_features(godot_mpv PRIVATE cxx_std_20)

target_link_libraries(godot_mpv PRIVATE
    godot::cpp
    glad_gles2
    glad_egl
)

target_include_directories(godot_mpv PRIVATE
    ${CMAKE_SOURCE_DIR}
    ${CMAKE_SOURCE_DIR}/dependencies/glad
    ${CMAKE_CURRENT_BINARY_DIR}/native/src/godot_mpv/include
)

# Platform-specific libmpv linkage
if(WIN32)
    target_link_libraries(godot_mpv PRIVATE
        "${CMAKE_SOURCE_DIR}/dependencies/mpv-dev/libmpv.dll.a"
        EGL
        GLESv2
    )
    target_include_directories(godot_mpv PRIVATE
        "${CMAKE_SOURCE_DIR}/dependencies/mpv-dev/include"
    )
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    link_directories(${CMAKE_SOURCE_DIR}/dependencies/aarch64/mpv)
    target_link_libraries(godot_mpv PRIVATE mpv EGL GLESv3)
else()
    # Use find_library/find_path to avoid pkg-config transitive dependency issues
    find_library(MPV_LIBRARY NAMES mpv REQUIRED
        HINTS /usr/local/lib /usr/local/lib/x86_64-linux-gnu
              /usr/lib /usr/lib/x86_64-linux-gnu)
    find_path(MPV_INCLUDE_DIR mpv/client.h REQUIRED
        HINTS /usr/local/include /usr/include)
    target_link_libraries(godot_mpv PRIVATE ${MPV_LIBRARY})
    target_include_directories(godot_mpv PRIVATE ${MPV_INCLUDE_DIR})
endif()

# Output naming and placement — matches godot-cpp-template pattern
set_target_properties(godot_mpv PROPERTIES
    PREFIX ""
    OUTPUT_NAME "godot_mpv${GODOTCPP_SUFFIX}"
    # RUNTIME covers Windows DLLs; LIBRARY covers Linux .so
    # $<1:...> prevents MSVC from inserting Debug/Release subdirs
    RUNTIME_OUTPUT_DIRECTORY "$<1:${PROJECT_SOURCE_DIR}/godot_project/bin/${GODOTCPP_PLATFORM}>"
    LIBRARY_OUTPUT_DIRECTORY "$<1:${PROJECT_SOURCE_DIR}/godot_project/bin/${GODOTCPP_PLATFORM}>"
)
