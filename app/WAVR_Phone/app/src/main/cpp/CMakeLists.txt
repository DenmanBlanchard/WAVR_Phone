# CMakeLists.txt for app/WAVR_Phone/app/src/main/cpp
#
# Builds libwavr_phone.so (loaded by MainActivity via System.loadLibrary("wavr_phone")).
#
# Phase 1 scope: GameActivity app that
#   - draws with GL ES 3 (EGL window surface)
#   - reads Cardboard head pose and logs it
#   - links the prebuilt ALVR client core (libalvr_client_core.so)
#   - decodes video with NDK MediaCodec into a GL texture
#   - shows one eye full screen, no distortion
#   - sends head pose to the ALVR server (networking is done by ALVR core)

cmake_minimum_required(VERSION 3.22.1)

project("wavr_phone" LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ---------------------------------------------------------------------------
# Options
# ---------------------------------------------------------------------------
# ALVR server version under test. 20.8.0 first, then 20.14.1; not final yet.
# This only labels the build. The client core is built from the matching
# third_party/alvr checkout by tools/build_alvr.sh.
set(WAVR_ALVR_VERSION "20.8.0" CACHE STRING "ALVR server version this build targets")

# Phase 1 needs Cardboard for head pose. Set OFF only for early GL-only bring-up.
option(WAVR_USE_CARDBOARD "Link the Google Cardboard SDK (head pose)" ON)

# ---------------------------------------------------------------------------
# Paths
# cpp -> main -> src -> app -> WAVR_Phone -> app -> repo root (6 levels up)
# ---------------------------------------------------------------------------
set(REPO_ROOT       "${CMAKE_CURRENT_SOURCE_DIR}/../../../../../..")
set(THIRD_PARTY_DIR "${REPO_ROOT}/third_party")
set(JNILIBS_DIR     "${CMAKE_CURRENT_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}")

# ALVR client core: C API header and prebuilt library.
set(ALVR_INCLUDE_DIR "${THIRD_PARTY_DIR}/alvr/alvr/client_core/c")
set(ALVR_LIBRARY     "${JNILIBS_DIR}/libalvr_client_core.so")

# Google Cardboard SDK: headers from the submodule (adjust if the layout differs).
set(CARDBOARD_INCLUDE_DIR "${THIRD_PARTY_DIR}/cardboard/libcardboard/include")
set(CARDBOARD_LIBRARY     "${JNILIBS_DIR}/libcardboard_api.so")

# ---------------------------------------------------------------------------
# Sanity checks
# ---------------------------------------------------------------------------
if(NOT EXISTS "${ALVR_LIBRARY}")
    message(FATAL_ERROR
            "libalvr_client_core.so not found at ${ALVR_LIBRARY}.\n"
            "Run tools/build_alvr.sh first (it writes into app/src/main/jniLibs).")
endif()

if(NOT EXISTS "${ALVR_INCLUDE_DIR}/alvr_client_core.h")
    message(FATAL_ERROR
            "alvr_client_core.h not found under ${ALVR_INCLUDE_DIR}. "
            "Check the third_party/alvr submodule checkout.")
endif()

if(WAVR_USE_CARDBOARD)
    if(NOT EXISTS "${CARDBOARD_INCLUDE_DIR}/cardboard.h")
        message(FATAL_ERROR
                "cardboard.h not found under ${CARDBOARD_INCLUDE_DIR}. "
                "Check the third_party/cardboard submodule, or configure with -DWAVR_USE_CARDBOARD=OFF.")
    endif()
    if(NOT EXISTS "${CARDBOARD_LIBRARY}")
        message(FATAL_ERROR
                "libcardboard_api.so not found at ${CARDBOARD_LIBRARY}. "
                "Copy it into app/src/main/jniLibs/${ANDROID_ABI}, or configure with -DWAVR_USE_CARDBOARD=OFF.")
    endif()
endif()

message(STATUS "wavr_phone: ALVR server ${WAVR_ALVR_VERSION}, Cardboard ${WAVR_USE_CARDBOARD}")

# ---------------------------------------------------------------------------
# Prebuilt libraries
# ---------------------------------------------------------------------------
add_library(alvr_client_core SHARED IMPORTED)
set_target_properties(alvr_client_core PROPERTIES IMPORTED_LOCATION "${ALVR_LIBRARY}")

if(WAVR_USE_CARDBOARD)
    add_library(cardboard_api SHARED IMPORTED)
    set_target_properties(cardboard_api PROPERTIES IMPORTED_LOCATION "${CARDBOARD_LIBRARY}")
endif()

# ---------------------------------------------------------------------------
# Game activity (prefab package from androidx.games)
# ---------------------------------------------------------------------------
find_package(game-activity REQUIRED CONFIG)

# ---------------------------------------------------------------------------
# Main shared library
# ---------------------------------------------------------------------------
add_library(wavr_phone SHARED
        main.cpp
        AndroidOut.cpp
        Renderer.cpp
        Shader.cpp
        TextureAsset.cpp
        Utility.cpp)

target_include_directories(wavr_phone PRIVATE
        "${ALVR_INCLUDE_DIR}"
        "${CMAKE_CURRENT_SOURCE_DIR}")

target_compile_definitions(wavr_phone PRIVATE
        WAVR_ALVR_VERSION="${WAVR_ALVR_VERSION}")

if(WAVR_USE_CARDBOARD)
    target_include_directories(wavr_phone PRIVATE "${CARDBOARD_INCLUDE_DIR}")
    target_compile_definitions(wavr_phone PRIVATE WAVR_HAS_CARDBOARD=1)
endif()

target_link_libraries(wavr_phone
        game-activity::game-activity
        alvr_client_core

        # Rendering
        EGL
        GLESv3
        jnigraphics

        # Video decode (AMediaCodec, AMediaFormat)
        mediandk

        # Android system and logging
        android
        log)

if(WAVR_USE_CARDBOARD)
    target_link_libraries(wavr_phone cardboard_api)
endif()

# Gradle packages src/main/jniLibs automatically, so the .so files are
# linked here but not copied again.