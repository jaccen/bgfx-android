cmake_minimum_required(VERSION 3.1)
cmake_policy(SET CMP0042 NEW)
project(Engine)

set(CMAKE_CXX_STANDARD 14)


option (Engine_BUILD_WITH_INSTALLER_EXTRACTOR "Compile with library to extract some installers of Gothic" OFF)


# Vim - You Complete Me
set(CMAKE_EXPORT_COMPILE_COMMANDS "ON")

set(BGFX_DEBUG 1)
add_definitions(-D__STDC_CONSTANT_MACROS)
add_definitions(-D__STDC_LIMIT_MACROS)

# clang-format
include(cmake/clang-format.cmake)

if(EMSCRIPTEN)

    # cmake -DCMAKE_TOOLCHAIN_FILE="/usr/lib/emscripten/cmake/Modules/Platform/Emscripten.cmake" ..

    SET(CMAKE_EXECUTABLE_SUFFIX ".html")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -g")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s LZ4=1")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --profiling")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s ASSERTIONS=0")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s USE_GLFW=3")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s USE_PTHREADS=2")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s TOTAL_MEMORY=924288000")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --preload-file ${CMAKE_CURRENT_SOURCE_DIR}/../content@/")
endif()

if(ANDROID)
    set(BGFX_DEBUG 1)
    set(Engine_BUILD_WITH_INSTALLER_EXTRACTOR ON CACHE STRING "" FORCE)

    # Usage: cmake -DANDROID_NDK=$NDK -DCMAKE_TOOLCHAIN_FILE=$NDK/cmake/toolchain.cmake -DANDROID_ABI=armeabi-v7a -DANDROID_NATIVE_API_LEVEL=23 -DLIBRARY_OUTPUT_PATH=. ..

    add_definitions(-D__gnu_linux__) # BGFX file reader needs to know this
    add_definitions(-D__ANDROID__) # Some like "ANDROID", like in the toolhain file, some expect __ANDROID__
    add_definitions(-Dalloca=__builtin_alloca) # imgui doesn't seem to take the crystax header from this
    add_definitions(-DBGFX_CONFIG_RENDERER_OPENGLES=30) # Need GLES3 for instancing

    # Fix other libraries
    set(BUILD_SQUISH_WITH_SSE2 OFF CACHE STRING "" FORCE) # No SSE2 on android

    # Make Apk.cmake compatible
    set(ARM_TARGET ${ANDROID_ABI})
    set(ANDROID_API_LEVEL ${ANDROID_NATIVE_API_LEVEL})

    set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR})

    include_directories(${ANDROID_NDK}/sources/android/native_app_glue)
    #include_directories(src/forandroid/compat)

    # TODO: Add a dummy file to supress a warning here
    file(GLOB ANDROID_SRC
        #${ANDROID_NDK}/sources/android/native_app_glue/android_native_app_glue.c
        #src/android/compat/*.c
    )

    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mno-unaligned-access") # Unaligned access not supported on ARM
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -ftree-vectorize")
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0 -g")
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x -ffor-scope -fno-rtti -pipe -ffunction-sections -fdata-sections -ffast-math -Wnon-virtual-dtor -Wreorder -Wsign-promo -fvisibility=hidden -fvisibility-inlines-hidden -Wstrict-null-sentinel -Os -funroll-all-loops -fpeel-loops -ftree-vectorize")
    set(LINKER_FLAGS "${LINKER_FLAGS} -nostartfiles, -Wl,--as-needed -Wl,--gc-sections -Wl,--no-undefined -Wl,--strip-all -Wl,-rpath-link=${ANDROID_NDK_SYSROOT}/usr/lib/ -L${ANDROID_NDK_SYSROOT}/usr/lib/")

    add_library(android_glue STATIC ${ANDROID_SRC})

    # Stuff all android libraries in here
    target_link_libraries(android_glue log android bgfx_common EGL GLESv2)
    set_target_properties(android_glue PROPERTIES COMPILE_DEFINITIONS "ANDROID")

endif()

if (APPLE)
    ADD_DEFINITIONS(-DBGFX_CONFIG_MULTITHREADED=0)
endif()

if (CMAKE_COMPILER_IS_GNUCC)
    # add warning flags
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-strict-aliasing -Wall -Wextra")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-reorder -Wno-unused-variable -Wno-unused-parameter -Wno-unused-function")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror=return-type") # missing return is error

    # Fix up C++ standard
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.4)
        message(WARNING "Compilation wasn't tested on GCC versions prior to 5.4 and might not be successful!")
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++1y")
    endif()
endif()

if(NOT ANDROID)
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
endif()

set(GAME_CONTENT_DIR ../content CACHE STRING "Content-folder, containing VDF-Archives and other files.")
set(GAME_STARTUP_WORLD addonworld.zen CACHE STRING "[Testing] World to load at program start")

# Setup BGFX
add_definitions(-DBGFX_CONFIG_MAX_MATRIX_CACHE=131072) # Need to up this since everything got it's own matrix right now
add_definitions(-DBGFX_CONFIG_MAX_COMMAND_BUFFER_SIZE=231072) # FIXME: Flush some times at loading, so we don't exhaust the default setting of 64k
add_definitions(-DBGFX_CONFIG_MAX_INDEX_BUFFERS=65535)
add_definitions(-DBGFX_CONFIG_MAX_VERTEX_BUFFERS=65535) # TODO: Don't mess around with memory so much

# Set up GLM
add_definitions(-DGLM_FORCE_PURE) # FIXME: glm crashes using SSE because of some alignment issues
add_definitions(-DGLM_ENABLE_EXPERIMENTAL) # Quaternions

add_definitions(-D__STDC_LIMIT_MACROS)
add_definitions(-D__STDC_CONSTANT_MACROS)
add_definitions(-D__STDC_FORMAT_MACROS)

# Temp
if(NOT MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")
endif()

if(MINGW)
    set(COMPILER_PREFIX "i686-mingw32")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s")
endif()

if(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc /MP /bigobj")
    add_definitions(-DNOMINMAX)
endif()

# Setup OpenMP
FIND_PACKAGE(OpenMP)
if(OPENMP_FOUND)
    message("OPENMP FOUND")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
endif()

# Setup bullet
if(MSVC)
    set(USE_MSVC_RUNTIME_LIBRARY_DLL ON)
endif()

# ------------------ Config-Header ------------------

configure_file(src/rgconfig.h.in ${PROJECT_BINARY_DIR}/src/rgconfig.h)
include_directories(${PROJECT_BINARY_DIR}/src)

add_definitions(-DBGFX_CONFIG_DEBUG=${BGFX_DEBUG})
add_subdirectory(lib/bgfx-cmake)


# ------------------ Engine lib ------------------
file(GLOB ENGINE_SRC
      src/*.cpp
        )

add_library(engine STATIC ${ENGINE_SRC})
#set_target_properties(engine PROPERTIES LINKER_LANGUAGE C)
target_link_libraries(engine     bgfx bgfx_common bgfx3rdParty )

# ------------------ Other ------------------

include_directories(src)
include_directories(lib)
include_directories(lib/glm)


if(NOT ANDROID)


    include_directories(lib/glfw/include)

    if(NOT EMSCRIPTEN)
        add_definitions(-DRE_USE_SOUND)
        add_subdirectory(lib/glfw)
        target_link_libraries(engine glfw)
    endif()
else()
    add_definitions(-DRE_USE_SOUND)
    target_link_libraries(engine android_glue)
endif()

set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT Engine)
