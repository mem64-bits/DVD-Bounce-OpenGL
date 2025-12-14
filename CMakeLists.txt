cmake_minimum_required(VERSION 3.25)

# ==============================================================================
#  PROJECT CONFIGURATION
# ==============================================================================
project(DVDBounceOpenGL CXX C) # <--- Change Name Here Only

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_C_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Feature: User Option to switch between Static (.lib) and Dynamic (.dll/.so)
option(USE_SHARED_LIBS "Build external libraries as Shared (Dynamic) instead of Static" OFF)

# Fix for Windows headers breaking std::min/max
if(MSVC)
    add_compile_definitions(NOMINMAX)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
endif()

# ==============================================================================
# 1. SETUP PYTHON VENV (Automated)
# ==============================================================================
find_package(Python3 COMPONENTS Interpreter REQUIRED)
set(VENV_DIR "${CMAKE_BINARY_DIR}/venv")

if(WIN32)
    set(VENV_PYTHON_EXE "${VENV_DIR}/Scripts/python.exe")
else()
    set(VENV_PYTHON_EXE "${VENV_DIR}/bin/python3")
endif()

if(NOT EXISTS "${VENV_PYTHON_EXE}")
    message(STATUS "🌱 Creating Python Virtual Environment in: ${VENV_DIR}...")
    execute_process(COMMAND "${Python3_EXECUTABLE}" -m venv "${VENV_DIR}")
    execute_process(COMMAND "${VENV_PYTHON_EXE}" -m pip install --upgrade pip --quiet)
endif()

# ==============================================================================
# 2. GLAD 2 GENERATION & LIBRARY SETUP
# ==============================================================================
set(GLAD_OUT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external/glad")
set(GLAD_HEADER "${GLAD_OUT_DIR}/include/glad/gl.h")

# A. Install 'glad2' if missing
execute_process(COMMAND "${VENV_PYTHON_EXE}" -c "import glad2" RESULT_VARIABLE GLAD_PKG_EXISTS ERROR_QUIET)
if(NOT GLAD_PKG_EXISTS EQUAL 0)
    message(STATUS "📦 Installing GLAD 2 generator...")
    execute_process(COMMAND "${VENV_PYTHON_EXE}" -m pip install glad2 jinja2 --quiet)
endif()

# B. Generate Sources
if(NOT EXISTS "${GLAD_HEADER}")
    message(STATUS "⚙️  Generating OpenGL 4.6 Core files...")
    execute_process(
        COMMAND "${VENV_PYTHON_EXE}" -m glad 
                --api gl:core=4.6
                --out-path "${GLAD_OUT_DIR}"
                --reproducible
                c 
    )
endif()

# C. Define GLAD Target (Static or Shared based on option)
if(USE_SHARED_LIBS)
    add_library(glad SHARED "${GLAD_OUT_DIR}/src/gl.c")
    target_compile_definitions(glad PUBLIC GLAD_GLAPI_EXPORT) # Required for DLL export
else()
    add_library(glad STATIC "${GLAD_OUT_DIR}/src/gl.c")
endif()

target_include_directories(glad PUBLIC "${GLAD_OUT_DIR}/include")

# ==============================================================================
# 3. EXTERNAL DEPENDENCIES
# ==============================================================================

# A. GLFW (Pass the Shared/Static option down to GLFW's build system)
set(GLFW_BUILD_DOCS OFF CACHE BOOL "" FORCE)
set(GLFW_INSTALL OFF CACHE BOOL "" FORCE)
set(BUILD_SHARED_LIBS ${USE_SHARED_LIBS} CACHE BOOL "" FORCE) # <--- Feature Retained

add_subdirectory(external/glfw)

# B. Header-Only Libs
add_library(glm INTERFACE)
target_include_directories(glm SYSTEM INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/external")

add_library(stb INTERFACE)
target_include_directories(stb SYSTEM INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/external/stb")

# ==============================================================================
# 4. HELPER FUNCTIONS
# ==============================================================================
function(silence_target target)
    if(TARGET ${target})
        if(MSVC)
            target_compile_options(${target} PRIVATE /W0)
        else()
            target_compile_options(${target} PRIVATE -w)
        endif()
    endif()
endfunction()

# Silence vendor warnings so they don't clutter your output
silence_target(glad)
silence_target(glfw)

# ==============================================================================
# 5. MAIN EXECUTABLE DEFINITION
# ==============================================================================
add_executable(${PROJECT_NAME} 
    src/main.cpp 
    src/stb_define.cpp
)

# Link everything
target_link_libraries(${PROJECT_NAME} PRIVATE glad glfw glm stb)

# Include Paths
target_include_directories(${PROJECT_NAME} PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/src/include"
    "${CMAKE_CURRENT_SOURCE_DIR}/external"
)

# ==============================================================================
# 6. COMPILER FLAGS & OPTIMIZATIONS (Feature Retained)
# ==============================================================================
if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE 
        /W4 /permissive- /GS /MP
        $<$<CONFIG:Release>:/O2 /Oi /Gy /arch:AVX2>
    )
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    target_compile_options(${PROJECT_NAME} PRIVATE 
        -Wall -Wextra -Wpedantic
        $<$<CONFIG:Release>:-O2 -march=native>
    )
endif()

# Enable Interprocedural Optimization (LTO) for Release builds
set_property(TARGET ${PROJECT_NAME} PROPERTY INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE)

# ==============================================================================
# 7. POST-BUILD: ASSET COPYING
# ==============================================================================
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/assets")
    file(GLOB_RECURSE ASSET_FILES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/assets/*")
    foreach(ASSET_PATH ${ASSET_FILES})
        file(RELATIVE_PATH ASSET_REL "${CMAKE_CURRENT_SOURCE_DIR}" "${ASSET_PATH}")
        add_custom_command(
            TARGET ${PROJECT_NAME} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different 
                "${ASSET_PATH}" 
                "$<TARGET_FILE_DIR:${PROJECT_NAME}>/${ASSET_REL}"
            COMMENT "📦 Updating asset: ${ASSET_REL}"
        )
    endforeach()
endif()

# ==============================================================================
# 8. POST-BUILD: DLL COPYING (Feature Retained)
# ==============================================================================
# If we built with USE_SHARED_LIBS=ON, we must copy glfw3.dll / glad.dll to the exe folder
if(WIN32 AND USE_SHARED_LIBS)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        $<TARGET_RUNTIME_DLLS:${PROJECT_NAME}>
        $<TARGET_FILE_DIR:${PROJECT_NAME}>
        COMMAND_EXPAND_LISTS
        COMMENT "📚 Copying Dynamic Libraries..."
    )
endif()

# ==============================================================================
# 9. DEBUGGER SETUP
# ==============================================================================
# Sets Visual Studio to run the exe from the build folder (so it finds assets)
set_target_properties(${PROJECT_NAME} PROPERTIES 
    VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
)