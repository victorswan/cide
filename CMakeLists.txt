cmake_minimum_required(VERSION 3.0)

cmake_policy(SET CMP0048 NEW)
cmake_policy(SET CMP0054 NEW)
cmake_policy(SET CMP0071 NEW)
cmake_policy(SET CMP0074 NEW)

project(CIDE LANGUAGES CXX)

# Make CMake find the Find<Package>.cmake files in the cmake subdirectory.
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)


if (NOT ${CMAKE_MAKE_PROGRAM} MATCHES "ninja")
  message(WARNING "Not using the Ninja Generator may lead to build failure.")
endif()

# --- Dependencies ---

# Cross-platform threading. See:
# https://cmake.org/cmake/help/latest/module/FindThreads.html
find_package(Threads REQUIRED)

# OpenMP
find_package(OpenMP)

# External dependency: libclang
find_package(Clang REQUIRED)

# External dependency: Qt 5
set(CMAKE_INCLUDE_CURRENT_DIR ON)
# Instruct CMake to run moc automatically when needed.
set(CMAKE_AUTOMOC ON)
# Instruct CMake to run rcc (resource compiler) automatically when needed.
set(CMAKE_AUTORCC ON)
find_package(Qt5 REQUIRED COMPONENTS Widgets Help Svg)

# External dependency: libgit2
find_package(Libgit2 REQUIRED)

# Packaged dependency: glslang
add_subdirectory(third_party/glslang)

# Packaged dependency: YAML-CPP
# Override YAML-CPP's option defaults to not build contrib and tools
option(YAML_CPP_BUILD_CONTRIB "Enable yaml-cpp contrib in library" OFF)
option(YAML_CPP_BUILD_TOOLS "Enable parse tools" OFF)
option(YAML_BUILD_SHARED_LIBS "Build yaml-cpp shared library" ON)
add_subdirectory(third_party/yaml-cpp)

# Packaged dependency: googletest
add_subdirectory(third_party/googletest)
set_target_properties(gtest gtest_main PROPERTIES FOLDER "third_party")


# --- Git version tool ---

add_executable(GitVersionTool
  src/git_version_tool/main.cc
)
target_include_directories(GitVersionTool PUBLIC
  src
  ${LIBGIT2_INCLUDE_DIR}
)
target_link_libraries(GitVersionTool
  ${LIBGIT2_LIBRARIES}
)
if (NOT CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  target_compile_options(GitVersionTool PUBLIC
    "$<$<COMPILE_LANGUAGE:CXX>:-Wall>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-Wextra>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-O2>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-msse2>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-msse3>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-Wno-sign-compare>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-Wno-strict-overflow>"
  )
endif()
target_compile_options(GitVersionTool PUBLIC
  ${LIBGIT2_DEFINITIONS}
)


# --- CIDE base library, used by both the executable and the test ---

# NOTE:
# Windows echo example: COMMAND ${CMAKE_COMMAND} -E echo "\#include <gtk/gtk.h>" > images.c
# Linux echo example: COMMAND echo "\"""#include""<gtk/gtk.h>" "\"" > images.c

# Ugly workaround to force running this custom command on every build:
# The idea is that the ${CMAKE_BINARY_DIR}/always_rebuild file should always be
# missing, which is listed as a dependency here. However, for make (but not
# ninja) it seems that we must generate this file here, otherwise the other file
# listed as output, CIDE_git_version.h, will get deleted. So, this does not work
# with make.
add_custom_command(
  OUTPUT ${CMAKE_BINARY_DIR}/CIDE_git_version.h ${CMAKE_BINARY_DIR}/always_rebuild
  COMMAND $<TARGET_FILE:GitVersionTool> ${CMAKE_SOURCE_DIR} ${CMAKE_BINARY_DIR}/CIDE_git_version.h
  DEPENDS GitVersionTool
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
)

set(CIDEBaseLib_SRC
  ${CMAKE_BINARY_DIR}/CIDE_git_version.h
  
  src/cide/about_dialog.cc
  src/cide/argument_hint_widget.cc
  src/cide/build_target_list_widget.cc
  src/cide/build_target_selector.cc
  src/cide/clang_highlighting.cc
  src/cide/clang_index.cc
  src/cide/clang_tu_pool.cc
  src/cide/clang_utils.cc
  src/cide/code_completion_widget.cc
  src/cide/code_info.cc
  src/cide/code_info_code_completion.cc
  src/cide/code_info_get_info.cc
  src/cide/code_info_get_right_click_info.cc
  src/cide/code_info_goto_referenced_cursor.cc
  src/cide/cpp_utils.cc
  src/cide/crash_backup.cc
  src/cide/create_class.cc
  src/cide/document.cc
  src/cide/document_range.cc
  src/cide/document_widget.cc
  src/cide/document_widget_container.cc
  src/cide/find_and_replace_in_files.cc
  src/cide/git_diff.cc
  src/cide/glsl_highlighting.cc
  src/cide/glsl_parser.cc
  src/cide/main_window.cc
  src/cide/new_project_dialog.cc
  src/cide/parse_thread_pool.cc
  src/cide/clang_parser.cc
  src/cide/problem.cc
  src/cide/project.cc
  src/cide/project_settings.cc
  src/cide/project_tree_view.cc
  src/cide/qt_help.cc
  src/cide/qt_thread.cc
  src/cide/rename_dialog.cc
  src/cide/run_gdb.cc
  src/cide/scroll_bar_minimap.cc
  src/cide/search_bar.cc
  src/cide/search_list_widget.cc
  src/cide/settings.cc
  src/cide/startup_dialog.cc
  src/cide/tab_bar.cc
  src/cide/text_block.cc
  src/cide/text_utils.cc
  src/cide/util.cc
)
# Not sure why this bad workaround is needed on Windows, maybe the functions
# in this part of glslang lack the dllexport pragma?
if (WIN32)
  set(CIDEBaseLib_SRC
    ${CIDEBaseLib_SRC}
    third_party/glslang/glslang/MachineIndependent/glslang.m4
    third_party/glslang/glslang/MachineIndependent/glslang.y
    third_party/glslang/glslang/MachineIndependent/glslang_tab.cpp
    third_party/glslang/glslang/MachineIndependent/attribute.cpp
    third_party/glslang/glslang/MachineIndependent/Constant.cpp
    third_party/glslang/glslang/MachineIndependent/iomapper.cpp
    third_party/glslang/glslang/MachineIndependent/InfoSink.cpp
    third_party/glslang/glslang/MachineIndependent/Initialize.cpp
    third_party/glslang/glslang/MachineIndependent/IntermTraverse.cpp
    third_party/glslang/glslang/MachineIndependent/Intermediate.cpp
    third_party/glslang/glslang/MachineIndependent/ParseContextBase.cpp
    third_party/glslang/glslang/MachineIndependent/ParseHelper.cpp
    third_party/glslang/glslang/MachineIndependent/PoolAlloc.cpp
    third_party/glslang/glslang/MachineIndependent/RemoveTree.cpp
    third_party/glslang/glslang/MachineIndependent/Scan.cpp
    third_party/glslang/glslang/MachineIndependent/ShaderLang.cpp
    third_party/glslang/glslang/MachineIndependent/SymbolTable.cpp
    third_party/glslang/glslang/MachineIndependent/Versions.cpp
    third_party/glslang/glslang/MachineIndependent/intermOut.cpp
    third_party/glslang/glslang/MachineIndependent/limits.cpp
    third_party/glslang/glslang/MachineIndependent/linkValidate.cpp
    third_party/glslang/glslang/MachineIndependent/parseConst.cpp
    third_party/glslang/glslang/MachineIndependent/reflection.cpp
    third_party/glslang/glslang/MachineIndependent/preprocessor/Pp.cpp
    third_party/glslang/glslang/MachineIndependent/preprocessor/PpAtom.cpp
    third_party/glslang/glslang/MachineIndependent/preprocessor/PpContext.cpp
    third_party/glslang/glslang/MachineIndependent/preprocessor/PpScanner.cpp
    third_party/glslang/glslang/MachineIndependent/preprocessor/PpTokens.cpp
    third_party/glslang/glslang/MachineIndependent/propagateNoContraction.cpp
  )
endif()
add_library(CIDEBaseLib SHARED
  ${CIDEBaseLib_SRC}
)
target_include_directories(CIDEBaseLib PUBLIC
  src
  third_party/yaml-cpp/include
  ${gtest_SOURCE_DIR}/include
  ${CLANG_INCLUDE_DIRS}
  ${LIBGIT2_INCLUDE_DIR}
)
target_link_libraries(CIDEBaseLib PUBLIC
  Qt5::Widgets
  Qt5::Help
  Qt5::Svg
  yaml-cpp
  ${CLANG_CLANG_LIB}
  glslang
  ${LIBGIT2_LIBRARIES}
  Threads::Threads
)
if (OpenMP_CXX_FOUND)
  target_link_libraries(CIDEBaseLib PUBLIC OpenMP::OpenMP_CXX)
endif()
if (NOT WIN32)  # TODO: Is this dependent on windows, or on using clang?
  target_compile_options(CIDEBaseLib PUBLIC
    "$<$<COMPILE_LANGUAGE:CXX>:-Wall>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-Wextra>"
  )
endif()
if (NOT CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  target_compile_options(CIDEBaseLib PUBLIC
    ";$<$<COMPILE_LANGUAGE:CXX>:-O2>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-msse2>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-msse3>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-Wno-sign-compare>"
    ";$<$<COMPILE_LANGUAGE:CXX>:-Wno-strict-overflow>"
  )
else()
  target_compile_options(CIDEBaseLib PUBLIC
    ";$<$<COMPILE_LANGUAGE:CXX>:/wd4267>"
    ";$<$<COMPILE_LANGUAGE:CXX>:/wd4018>"
  )
endif()
target_compile_options(CIDEBaseLib PUBLIC
  ${LIBGIT2_DEFINITIONS}
)


# --- CIDE executable ---

set(CIDE_SRCS
  src/cide/main.cc
  resources_packed/resources.qrc
)
if (WIN32)
  set(CIDE_SRCS
    ${CIDE_SRCS}
    resources_packed/cide.rc
  )
endif()
add_executable(CIDE WIN32
  ${CIDE_SRCS}
)
target_link_libraries(CIDE
  CIDEBaseLib
)
if (APPLE)
  set_target_properties(CIDE PROPERTIES
      MACOSX_BUNDLE ON
      MACOSX_BUNDLE_BUNDLE_NAME "CIDE"
      MACOSX_BUNDLE_BUNDLE_IDENTIFIER "net.puzzlepaint.cide")
  file(
    COPY ${CMAKE_CURRENT_SOURCE_DIR}/resources
    DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/CIDE.app/Contents/MacOS
  )
else()
  file(
    COPY ${CMAKE_CURRENT_SOURCE_DIR}/resources
    DESTINATION ${CMAKE_CURRENT_BINARY_DIR}
  )
endif()


# --- CIDE Test executable ---

add_executable(CIDETest
  src/cide/test.cc
)
target_link_libraries(CIDETest
  CIDEBaseLib
  gtest
)
add_test(CIDETest
  CIDETest
)
