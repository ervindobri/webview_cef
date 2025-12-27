# The Flutter tooling requires that developers have a version of Visual Studio
# installed that includes CMake 3.14 or later. You should not increase this
# version, as doing so will cause the plugin to fail to compile for some
# customers of the plugin.
cmake_minimum_required(VERSION 3.14)

# Project-level configuration.
set(PROJECT_NAME "webview_cef")
project(${PROJECT_NAME} LANGUAGES CXX)

if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
  message(STATUS "${CMAKE_BUILD_TYPE}")
endif()

include(${CMAKE_CURRENT_SOURCE_DIR}/../third/download.cmake)

# wait for unified version cef source code and prebuilt files
prepare_prebuilt_files(${CMAKE_CURRENT_SOURCE_DIR}/../third/cef)

# CEF configuration
set(CEF_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../third/cef")

# Build libcef_dll_wrapper from source
set(CEF_LIBCEF_DLL_WRAPPER_PATH "${CEF_ROOT}/libcef_dll")

if(EXISTS "${CEF_LIBCEF_DLL_WRAPPER_PATH}")
  message(STATUS "Building libcef_dll_wrapper from source at ${CEF_LIBCEF_DLL_WRAPPER_PATH}")

  # Collect all wrapper source files
  file(GLOB_RECURSE CEF_WRAPPER_SOURCES
    "${CEF_LIBCEF_DLL_WRAPPER_PATH}/*.cc"
    "${CEF_LIBCEF_DLL_WRAPPER_PATH}/*.cpp"
  )

  # Create the wrapper library
  add_library(libcef_dll_wrapper STATIC ${CEF_WRAPPER_SOURCES})

  # Set include directories for wrapper
  target_include_directories(libcef_dll_wrapper PUBLIC "${CEF_ROOT}")
  target_include_directories(libcef_dll_wrapper PUBLIC "${CEF_ROOT}/include")

  # Wrapper compile definitions
  target_compile_definitions(libcef_dll_wrapper PRIVATE
    NOMINMAX
    WIN32_LEAN_AND_MEAN
    WRAPPING_CEF_SHARED
  )

  # C++17 standard
  set_target_properties(libcef_dll_wrapper PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
  )
else()
  message(WARNING "CEF wrapper source not found at ${CEF_LIBCEF_DLL_WRAPPER_PATH}")
endif()

# This value is used when generating builds using this plugin, so it must
# not be changed
set(PLUGIN_NAME "webview_cef_plugin")

# Any new source files that you add to the plugin should be added here.
list(APPEND PLUGIN_SOURCES
  "webview_cef_plugin.cpp"
  "webview_cef_plugin.h"
  "webview_cef_keyevent.h"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_app.cc"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_app.h"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_handler.cc"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_handler.h"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_plugin.cc"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_plugin.h"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_value.cc"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_value.h"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_js_handler.cc"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_js_handler.h"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_cookieVisitor.cc"
  "${CMAKE_CURRENT_LIST_DIR}/../common/webview_cookieVisitor.h"
)

# Define the plugin library target. Its name must not be changed (see comment
# on PLUGIN_NAME above).
add_library(${PLUGIN_NAME} SHARED
  "include/webview_cef/webview_cef_plugin_c_api.h"
  "webview_cef_plugin_c_api.cpp"
  ${PLUGIN_SOURCES}
)

# Apply a standard set of build settings that are configured in the
# application-level CMakeLists.txt. This can be removed for plugins that want
# full control over build settings.
apply_standard_settings(${PLUGIN_NAME})

# Symbols are hidden by default to reduce the chance of accidental conflicts
# between plugins. This should not be removed; any symbols that should be
# exported should be explicitly exported with the FLUTTER_PLUGIN_EXPORT macro.
set_target_properties(${PLUGIN_NAME} PROPERTIES
  CXX_VISIBILITY_PRESET hidden)
target_compile_definitions(${PLUGIN_NAME} PRIVATE FLUTTER_PLUGIN_IMPL NOMINMAX)

# Source include directories and library dependencies. Add any plugin-specific
# dependencies here.
target_include_directories(${PLUGIN_NAME} INTERFACE
  "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_include_directories(${PLUGIN_NAME} PRIVATE "${CEF_ROOT}")
target_include_directories(${PLUGIN_NAME} PRIVATE "${CEF_ROOT}/include")
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../common)

# Link against CEF libraries
target_link_libraries(${PLUGIN_NAME} PRIVATE flutter flutter_wrapper_plugin)

# Link wrapper (built from source)
if(TARGET libcef_dll_wrapper)
  target_link_libraries(${PLUGIN_NAME} PRIVATE libcef_dll_wrapper)
endif()

# Link libcef.lib - Standard CEF distribution structure
target_link_libraries(${PLUGIN_NAME} PRIVATE
  debug "${CEF_ROOT}/Debug/libcef.lib"
  optimized "${CEF_ROOT}/Release/libcef.lib"
)

# List of absolute paths to libraries that should be bundled with the plugin.
# Standard CEF distribution paths
set(webview_cef_bundled_libraries
  "${CEF_ROOT}/Resources/locales"
  "${CEF_ROOT}/Resources/icudtl.dat"
  "${CEF_ROOT}/Resources/resources.pak"
  "${CEF_ROOT}/Resources/chrome_100_percent.pak"
  "${CEF_ROOT}/Resources/chrome_200_percent.pak"
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/chrome_elf.dll>
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/libcef.dll>
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/v8_context_snapshot.bin>
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/d3dcompiler_47.dll>
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/libEGL.dll>
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/libGLESv2.dll>
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/vulkan-1.dll>
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/vk_swiftshader.dll>
  $<$<CONFIG:Debug>:${CEF_ROOT}/Debug/vk_swiftshader_icd.json>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/chrome_elf.dll>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/libcef.dll>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/v8_context_snapshot.bin>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/d3dcompiler_47.dll>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/libEGL.dll>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/libGLESv2.dll>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/vulkan-1.dll>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/vk_swiftshader.dll>
  $<$<CONFIG:Release>:${CEF_ROOT}/Release/vk_swiftshader_icd.json>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/chrome_elf.dll>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/libcef.dll>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/v8_context_snapshot.bin>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/d3dcompiler_47.dll>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/libEGL.dll>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/libGLESv2.dll>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/vulkan-1.dll>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/vk_swiftshader.dll>
  $<$<CONFIG:Profile>:${CEF_ROOT}/Release/vk_swiftshader_icd.json>
  PARENT_SCOPE)
