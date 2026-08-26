
cmake_minimum_required(VERSION 3.10)
project(mujoco_vendor)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)

# Handle the case in which mujoco should not vendored, for example in conda-forge/robostack where a recent enough
# mujoco package is available in the distribution, and there is no need to vendor it
# We use the AMENT_VENDOR_POLICY option proposed in https://github.com/ament/ament_cmake/pull/592 and that will
# be introduced at some point in ament_vendor for this purpouse
if(DEFINED AMENT_VENDOR_POLICY AND (AMENT_VENDOR_POLICY STREQUAL "NEVER_VENDOR" OR AMENT_VENDOR_POLICY STREQUAL "NEVER_VENDOR_IGNORE_SATISFIED_CHECK"))
  ament_export_dependencies(mujoco)
  ament_package()
  return()
endif()

include(FetchContent)

set(MUJOCO_VERSION "3.4.0")
set(MUJOCO_DOWNLOAD_URL "https://github.com/google-deepmind/mujoco/releases/download/${MUJOCO_VERSION}")

# Detect system and architecture for appropriate asset selection
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|ARM64")
    set(MUJOCO_ASSET_NAME "mujoco-${MUJOCO_VERSION}-linux-aarch64.tar.gz")
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64")
    set(MUJOCO_ASSET_NAME "mujoco-${MUJOCO_VERSION}-linux-x86_64.tar.gz")
  else()
    message(FATAL_ERROR "Unsupported Linux architecture: ${CMAKE_SYSTEM_PROCESSOR}")
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  # macOS ships a .dmg universal2 framework (verified upstream); handled below.
  set(MUJOCO_ASSET_NAME "mujoco-${MUJOCO_VERSION}-macos-universal2.dmg")
  set(MUJOCO_IS_DMG TRUE)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  set(MUJOCO_ASSET_NAME "mujoco-${MUJOCO_VERSION}-windows-x86_64.zip")
else()
  message(FATAL_ERROR "Unsupported operating system: ${CMAKE_SYSTEM_NAME}")
endif()

if(APPLE)
  set(MUJOCO_LIBNAME "libmujoco.dylib")
else()
  set(MUJOCO_LIBNAME "libmujoco.so")
endif()
set(MUJOCO_ASSET_URL "${MUJOCO_DOWNLOAD_URL}/${MUJOCO_ASSET_NAME}")

message(STATUS "Downloading MuJoCo prebuilt binary: ${MUJOCO_ASSET_NAME}")

if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.24")
  set(_mujoco_timestamp DOWNLOAD_EXTRACT_TIMESTAMP TRUE)
endif()

set(MUJOCO_ROOT_DIR "${CMAKE_BINARY_DIR}/mujoco_prebuilt_${MUJOCO_VERSION}")
if(MUJOCO_IS_DMG)
  # macOS: download the .dmg, mount it, and extract the framework's Headers +
  # dylib into the same include/lib layout the tar.gz path produces; then rewrite
  # the dylib install name (framework path -> @rpath) so it is self-consistent in
  # lib/. Locally validated end-to-end against the 3.4.0 universal2 DMG.
  if(NOT EXISTS "${MUJOCO_ROOT_DIR}/lib/${MUJOCO_LIBNAME}")
    set(_mj_dmg "${CMAKE_BINARY_DIR}/mujoco.dmg")
    file(DOWNLOAD "${MUJOCO_ASSET_URL}" "${_mj_dmg}" SHOW_PROGRESS)
    set(_mj_mnt "${CMAKE_BINARY_DIR}/mjmnt")
    file(MAKE_DIRECTORY "${MUJOCO_ROOT_DIR}/include/mujoco" "${MUJOCO_ROOT_DIR}/lib")
    execute_process(COMMAND hdiutil attach "${_mj_dmg}" -nobrowse -mountpoint "${_mj_mnt}" RESULT_VARIABLE _mj_att)
    if(NOT _mj_att EQUAL 0)
      message(FATAL_ERROR "mujoco_vendor: failed to mount ${_mj_dmg}")
    endif()
    file(GLOB _mj_hdrs "${_mj_mnt}/mujoco.framework/Headers/*.h")
    file(COPY ${_mj_hdrs} DESTINATION "${MUJOCO_ROOT_DIR}/include/mujoco")
    file(GLOB _mj_dylib "${_mj_mnt}/mujoco.framework/Versions/A/libmujoco.*.dylib")
    file(COPY ${_mj_dylib} DESTINATION "${MUJOCO_ROOT_DIR}/lib")
    if(EXISTS "${_mj_mnt}/model")
      file(COPY "${_mj_mnt}/model" DESTINATION "${MUJOCO_ROOT_DIR}")
    endif()
    execute_process(COMMAND hdiutil detach "${_mj_mnt}")
    file(GLOB _mj_lib "${MUJOCO_ROOT_DIR}/lib/libmujoco.*.dylib")
    get_filename_component(_mj_libfile "${_mj_lib}" NAME)
    execute_process(COMMAND install_name_tool -id "@rpath/${_mj_libfile}" "${_mj_lib}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "${_mj_libfile}" "${MUJOCO_ROOT_DIR}/lib/libmujoco.dylib")
  endif()
else()
  FetchContent_Declare(
    mujoco_binary
    URL "${MUJOCO_ASSET_URL}"
    ${_mujoco_timestamp}
  )
  FetchContent_MakeAvailable(mujoco_binary)
endif()

# Copy the FetchContent download to a stable, version-named directory in the
# build tree so that cmake_install.cmake does not reference the
# FetchContent-internal _deps/ path, which may not exist when cmake re-checks
# the build system during the Debian dh_auto_install step.
# The version suffix ensures a fresh copy whenever MUJOCO_VERSION changes.
set(MUJOCO_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/opt/${PROJECT_NAME}")

# Install prebuilt binaries (copy actual files, resolve symlinks)
install(DIRECTORY ${MUJOCO_ROOT_DIR}/
  DESTINATION opt/${PROJECT_NAME}
  USE_SOURCE_PERMISSIONS
  FILES_MATCHING
  PATTERN "*"
)

# The MuJoCo plugins ship with a hardcoded build-time RPATH
# (/tmpfs/src/git/mujoco_internal/build/lib) that does not exist on target
# machines. Rewrite it to a $ORIGIN-relative path so that:
#   1. dpkg-shlibdeps can locate libmujoco.so.3.4.0 when analysing the plugins
#      in the Debian staging area (it expands $ORIGIN to the directory of the
#      file under analysis, landing in opt/mujoco_vendor/lib/ where
#      libmujoco.so.3.4.0 is installed by this same package), and
#   2. plugins can find libmujoco.so at runtime without relying solely on
#      LD_LIBRARY_PATH.
find_program(PATCHELF_EXECUTABLE patchelf)
if(NOT PATCHELF_EXECUTABLE)
  message(WARNING
    "patchelf not found. MuJoCo plugin RPATHs will not be fixed; "
    "Debian packaging will fail with 'cannot find libmujoco.so.3.4.0'.")
else()
  install(CODE "
    file(GLOB _mujoco_plugins
      \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/opt/${PROJECT_NAME}/bin/mujoco_plugin/*.so\")
    foreach(_plugin IN LISTS _mujoco_plugins)
      execute_process(
        COMMAND \"${PATCHELF_EXECUTABLE}\" --set-rpath \"\$ORIGIN/../../lib\" \"\${_plugin}\"
        RESULT_VARIABLE _rc
      )
      if(NOT _rc EQUAL 0)
        message(WARNING \"patchelf failed for \${_plugin}\")
      endif()
    endforeach()
  ")
endif()

# Create symlinks in lib/${PROJECT_NAME}/ pointing to the real binaries in
# opt/${PROJECT_NAME}/bin/ so that `ros2 run mujoco_vendor <exe>` works
# without duplicating files on disk.
install(CODE "
  file(GLOB _mujoco_exes
    \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/opt/${PROJECT_NAME}/bin/*\")
  file(MAKE_DIRECTORY \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}\")
  foreach(_exe IN LISTS _mujoco_exes)
    if(NOT IS_DIRECTORY \"\${_exe}\")
      get_filename_component(_name \"\${_exe}\" NAME)
      execute_process(
        COMMAND \${CMAKE_COMMAND} -E create_symlink
          \"../../opt/${PROJECT_NAME}/bin/\${_name}\"
          \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/\${_name}\"
      )
    endif()
  endforeach()
")

add_library(mujoco INTERFACE)
# add alias mujoco::mujoco mujoco
add_library(mujoco::mujoco ALIAS mujoco)
if(NOT MUJOCO_IS_DMG)
  add_dependencies(mujoco mujoco_binary)
endif()
target_include_directories(mujoco INTERFACE
  $<BUILD_INTERFACE:${MUJOCO_ROOT_DIR}/include>
  $<INSTALL_INTERFACE:opt/${PROJECT_NAME}/include>
)
target_link_libraries(mujoco INTERFACE
  $<BUILD_INTERFACE:${MUJOCO_ROOT_DIR}/lib/${MUJOCO_LIBNAME}>
  $<INSTALL_INTERFACE:opt/${PROJECT_NAME}/lib/${MUJOCO_LIBNAME}>
)

install(TARGETS mujoco
  EXPORT export_${PROJECT_NAME}
  LIBRARY DESTINATION opt/${PROJECT_NAME}/lib
  ARCHIVE DESTINATION opt/${PROJECT_NAME}/lib
  RUNTIME DESTINATION opt/${PROJECT_NAME}/bin
)

# Pinned lodepng version from:
# https://github.com/google-deepmind/mujoco/blob/3.4.0/simulate/CMakeLists.txt#L26
set(MUJOCO_DEP_VERSION_lodepng "b4ed2cd7ecf61d29076169b49199371456d4f90b")
FetchContent_Declare(lodepng
  GIT_REPOSITORY https://github.com/lvandeve/lodepng.git
  GIT_TAG ${MUJOCO_DEP_VERSION_lodepng}
)
FetchContent_GetProperties(lodepng)
if(NOT lodepng_POPULATED)
  FetchContent_Populate(lodepng)
  add_library(lodepng STATIC ${lodepng_SOURCE_DIR}/lodepng.cpp)
  set_target_properties(lodepng PROPERTIES POSITION_INDEPENDENT_CODE ON)
  target_include_directories(lodepng PUBLIC $<BUILD_INTERFACE:${lodepng_SOURCE_DIR}>)
endif()
install(TARGETS lodepng ARCHIVE DESTINATION opt/${PROJECT_NAME}/lib)
install(FILES ${lodepng_SOURCE_DIR}/lodepng.h DESTINATION opt/${PROJECT_NAME}/include)

# Setup the simulate library for mujoco_ros2_control
find_package(glfw3 REQUIRED)

# The Linux/Windows release tarballs ship the `simulate/` sample sources, but the
# macOS framework .dmg ships only the compiled framework (lib + headers). Fetch the
# matching-version simulate sources from the MuJoCo source so mujoco_simulate can be
# built on macOS as well.
if(NOT EXISTS "${MUJOCO_ROOT_DIR}/simulate/simulate.cc"
   AND NOT EXISTS "${MUJOCO_ROOT_DIR}/include/simulate/simulate.cc")
  FetchContent_Declare(mujoco_simulate_src
    URL "https://github.com/google-deepmind/mujoco/archive/refs/tags/${MUJOCO_VERSION}.tar.gz"
    ${_mujoco_timestamp}
  )
  FetchContent_GetProperties(mujoco_simulate_src)
  if(NOT mujoco_simulate_src_POPULATED)
    FetchContent_Populate(mujoco_simulate_src)
  endif()
  file(COPY "${mujoco_simulate_src_SOURCE_DIR}/simulate/"
       DESTINATION "${MUJOCO_ROOT_DIR}/simulate")
endif()

set(_SIM_DIR "${MUJOCO_ROOT_DIR}/simulate")
if(NOT EXISTS "${_SIM_DIR}/simulate.h")
  set(_SIM_DIR "${MUJOCO_ROOT_DIR}/include/simulate")
endif()

add_library(mujoco_simulate STATIC
  ${_SIM_DIR}/platform_ui_adapter.cc
  ${_SIM_DIR}/glfw_adapter.cc
  ${_SIM_DIR}/glfw_dispatch.cc
  ${_SIM_DIR}/simulate.cc
)
# macOS: the simulate GUI uses Objective-C++ backends (CoreVideo display link +
# Cocoa window glue) and links those frameworks — mirrors upstream
# simulate/CMakeLists.txt's APPLE branch. Without these the .cc above under-link.
if(APPLE)
  target_sources(mujoco_simulate PRIVATE
    ${_SIM_DIR}/glfw_corevideo.mm
    ${_SIM_DIR}/macos_gui.mm
  )
  set_source_files_properties(${_SIM_DIR}/glfw_corevideo.mm
    PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)
  target_link_libraries(mujoco_simulate PUBLIC "-framework CoreVideo" "-framework Cocoa")
endif()
set_target_properties(mujoco_simulate PROPERTIES POSITION_INDEPENDENT_CODE ON)
target_include_directories(mujoco_simulate PUBLIC
  $<BUILD_INTERFACE:${_SIM_DIR}>
  $<BUILD_INTERFACE:${MUJOCO_ROOT_DIR}/include>
)
target_link_libraries(mujoco_simulate PUBLIC mujoco glfw PRIVATE lodepng)
target_compile_options(mujoco_simulate PRIVATE
  -Wno-missing-field-initializers -Wno-unused-parameter -Wno-sign-compare -Wno-psabi
)
install(TARGETS mujoco_simulate ARCHIVE DESTINATION opt/${PROJECT_NAME}/lib)
install(DIRECTORY ${_SIM_DIR}/ DESTINATION opt/${PROJECT_NAME}/include/simulate
  FILES_MATCHING PATTERN "*.h"
)

# Export includes and library path
ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)
ament_export_include_directories(opt/${PROJECT_NAME}/include)

# Install environment hook to set LD_LIBRARY_PATH
ament_environment_hooks("${CMAKE_CURRENT_SOURCE_DIR}/env-hooks/${PROJECT_NAME}.dsv")

ament_package(
  CONFIG_EXTRAS_POST "mujoco_vendor-extras.cmake"
)
