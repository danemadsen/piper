cmake_minimum_required(VERSION 3.13)

file(READ "${CMAKE_CURRENT_LIST_DIR}/VERSION" piper_version)

project(
  piper
  VERSION 1.2.0
  DESCRIPTION "Piper neural text to speech system"
  HOMEPAGE_URL "https://github.com/rhasspy/piper"
  LANGUAGES CXX
)

option(BUILD_SHARED_LIBS "Build using shared libraries" ON)

if(MSVC)
  # Force compiler to use UTF-8 for IPA constants
  add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
  add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
elseif(NOT APPLE)
  # Linux flags
  string(APPEND CMAKE_CXX_FLAGS " -Wall -Wextra -Wl,-rpath,'$ORIGIN'")
  string(APPEND CMAKE_C_FLAGS " -Wall -Wextra")
endif()

# Conditionally set the library type based on the option
if(BUILD_SHARED_LIBS)
  set(LIB_TYPE SHARED)
else()
  set(LIB_TYPE STATIC)
endif()

# Use the LIB_TYPE variable to determine the type of library to build
add_library(piper ${LIB_TYPE} src/cpp/piper.cpp)

# ---- fmt ---

if(NOT DEFINED FMT_DIR)
  set(FMT_VERSION "10.0.0")
  set(FMT_DIR "${CMAKE_CURRENT_BINARY_DIR}/fi")

  include(ExternalProject)
  ExternalProject_Add(
    fmt_external
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/f"
    URL "https://github.com/fmtlib/fmt/archive/refs/tags/${FMT_VERSION}.zip"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${FMT_DIR}
    CMAKE_ARGS -DFMT_TEST:BOOL=OFF  # Don't build all the tests
  )
  add_dependencies(piper fmt_external)
endif()

# ---- spdlog ---

if(NOT DEFINED SPDLOG_DIR)
  set(SPDLOG_DIR "${CMAKE_CURRENT_BINARY_DIR}/si")
  set(SPDLOG_VERSION "1.12.0")
  ExternalProject_Add(
    spdlog_external
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/s"
    URL "https://github.com/gabime/spdlog/archive/refs/tags/v${SPDLOG_VERSION}.zip"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${SPDLOG_DIR}
  )
  add_dependencies(piper spdlog_external)
endif()

# ---- piper-phonemize ---

if(NOT DEFINED PIPER_PHONEMIZE_DIR)
  set(PIPER_PHONEMIZE_DIR "${CMAKE_CURRENT_BINARY_DIR}/pi")
  ExternalProject_Add(
    piper_phonemize_external
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/p"
    URL "https://github.com/rhasspy/piper-phonemize/archive/refs/heads/master.zip"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${PIPER_PHONEMIZE_DIR}
  )
  add_dependencies(piper piper_phonemize_external)
endif()

if((NOT MSVC) AND (NOT APPLE))
  # Linux flags
  string(APPEND CMAKE_CXX_FLAGS " -Wall -Wextra -Wl,-rpath,'$ORIGIN'")
  string(APPEND CMAKE_C_FLAGS " -Wall -Wextra")
endif()

target_link_libraries(piper
  fmt
  spdlog
  espeak-ng
  piper_phonemize
  onnxruntime
)

target_link_directories(piper PUBLIC
  ${FMT_DIR}/lib
  ${SPDLOG_DIR}/lib
  ${PIPER_PHONEMIZE_DIR}/lib
)

target_include_directories(piper PUBLIC
  ${FMT_DIR}/include
  ${SPDLOG_DIR}/include
  ${PIPER_PHONEMIZE_DIR}/include
)

target_compile_definitions(piper PUBLIC _PIPER_VERSION=${piper_version})

# ---- Declare install targets ----

install(
  TARGETS piper
  LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX})

# Dependencies
install(
  DIRECTORY ${PIPER_PHONEMIZE_DIR}/bin/
  DESTINATION ${CMAKE_INSTALL_PREFIX}
  USE_SOURCE_PERMISSIONS  # keep +x
  FILES_MATCHING
  PATTERN "piper_phonemize"
  PATTERN "espeak-ng"
  PATTERN "*.dll"
)

install(
  DIRECTORY ${PIPER_PHONEMIZE_DIR}/lib/
  DESTINATION ${CMAKE_INSTALL_PREFIX}
  FILES_MATCHING
  PATTERN "*.dll"
  PATTERN "*.so*"
)

install(
  DIRECTORY ${PIPER_PHONEMIZE_DIR}/share/espeak-ng-data
  DESTINATION ${CMAKE_INSTALL_PREFIX}
)

install(
  FILES ${PIPER_PHONEMIZE_DIR}/share/libtashkeel_model.ort
  DESTINATION ${CMAKE_INSTALL_PREFIX}
)
