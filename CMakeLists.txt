cmake_minimum_required(VERSION 3.12)

# 定义项目名称
project(anroid_ros-superbuild)

include(ExternalProject)# 引入 ExternalProject 模块，用于外部项目的管理

# 设置 Android 目标 SDK 版本和构建工具版本
set(minimum_target_sdk 30)
set(build_tool_version 35.0.0)
set(ndk_version 25.1.8937393)

# 检查 ANDROID_HOME 环境变量是否已设置
if(NOT ANDROID_HOME)
  # 可以手动指定 Android SDK 位置，避免错误
  set(ANDROID_HOME "/home/xxj/Android/Sdk")
endif()

if(NOT ANDROID_HOME)
  message(FATAL_ERROR "ANDROID_HOME must be given")
endif()

# 设置默认的 keystore 路径（用于 APK 签名）
if(NOT PATH_TO_KEYSTORE)
  set(PATH_TO_KEYSTORE "$ENV{HOME}/.android/debug.keystore")
endif()
message(STATUS "Given keystore ${PATH_TO_KEYSTORE}")
# 查找 Android 工具链文件
file(GLOB android_toolchain LIST_DIRECTORIES false "${ANDROID_HOME}/ndk/${ndk_version}/build/cmake/android.toolchain.cmake")
if(NOT android_toolchain)
  message(FATAL_ERROR "Failed to find android toolchain file - is ANDROID_HOME correct?")
endif()

# 查找 aapt2 工具（用于 Android 资源编译）
file(GLOB android_aapt2 LIST_DIRECTORIES false "${ANDROID_HOME}/build-tools/${build_tool_version}/aapt2")
if(NOT android_aapt2)
  message(FATAL_ERROR "Failed to find aapt2 - is ANDROID_HOME correct?")
endif()

# 查找 android.jar 文件（Android SDK 平台库）
file(GLOB android_jar LIST_DIRECTORIES false "${ANDROID_HOME}/platforms/android-${minimum_target_sdk}/android.jar")
if(NOT android_jar)
  message(FATAL_ERROR "Failed to find android.jar - is ANDROID_HOME correct?")
endif()

# 查找 zipalign（用于 APK 对齐）
file(GLOB android_zipalign LIST_DIRECTORIES false "${ANDROID_HOME}/build-tools/${build_tool_version}/zipalign")
if(NOT android_zipalign)
  message(FATAL_ERROR "Failed to find zipalign - is ANDROID_HOME correct?")
endif()

# 查找 apksigner（用于 APK 签名）
file(GLOB android_apksigner LIST_DIRECTORIES false "${ANDROID_HOME}/build-tools/${build_tool_version}/apksigner")
if(NOT android_apksigner)
  message(FATAL_ERROR "Failed to find apksigner - is ANDROID_HOME correct?")
endif()

# 设置 Android ABI（架构）
set(ANDROID_ABI arm64-v8a)

# 定义 CMake 交叉编译参数
set(android_cmake_args 
  -DCMAKE_TOOLCHAIN_FILE=${android_toolchain}
  -DANDROID_ABI=${ANDROID_ABI}
  -DANDROID_PLATFORM=android-${minimum_target_sdk}
  )

  # 创建依赖项 Python 安装目录
# Make place to install Python packages we depend on
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/deps/_python_)

include(dependencies.cmake) # 包含依赖项配置文件

build_native_dependencies()  # 构建本地依赖
build_crosscompile_dependencies()# 构建交叉编译依赖

##################################
# Android specific projects
##################################

# 设置环境变量
set(cmake_with_env "${CMAKE_COMMAND}" -E
  env
  "PYTHONPATH=${CMAKE_CURRENT_BINARY_DIR}/deps/_python_"
  "AMENT_PREFIX_PATH=${CMAKE_CURRENT_BINARY_DIR}/deps"
  "${CMAKE_COMMAND}")

# 定义 ExternalProject 用于构建 ros_for_android
ExternalProject_Add(ros_for_android
  BUILD_ALWAYS ON
  DOWNLOAD_COMMAND ""
  CMAKE_COMMAND ${cmake_with_env} -DCMAKE_BUILD_TYPE=RELEASE
  BUILD_COMMAND ${cmake_with_env} --build .
  SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src"
  DEPENDS deps-rclcpp deps-rclcpp_lifecycle deps-std_msgs deps-sensor_msgs
  CMAKE_ARGS ${android_cmake_args} "-DCMAKE_FIND_ROOT_PATH=${CMAKE_CURRENT_BINARY_DIR}/deps"
  -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/ros_for_android
  )

# 配置 AndroidManifest.xml
set(NAMESPACE "com.github.sloretz.sensors_for_ros")
set(MIN_SDK_VERSION 30)
set(TARGET_SDK_VERSION 30)
set(LIB_NAME "android-ros")
configure_file(AndroidManifest.xml.in AndroidManifest.xml @ONLY)

  # 资源编译
# Make a place for compiled resources to go
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/compiled_resources)
# Make a list of compiled resources for copying
set(aapt2_compiled_resources)
macro(aapt2_compile input output)
add_custom_command(
  OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/compiled_resources/${output}"
  COMMENT "compiling ${input} -> ${output}"
  COMMAND ${android_aapt2} compile
  "${CMAKE_CURRENT_SOURCE_DIR}/${input}"
  -o "${CMAKE_CURRENT_BINARY_DIR}/compiled_resources"
  )
  list(APPEND aapt2_compiled_resources "${CMAKE_CURRENT_BINARY_DIR}/compiled_resources/${output}")
endmacro()

# 资源编译命令
aapt2_compile(res/values/strings.xml values_strings.arsc.flat)
aapt2_compile(res/values/colors.xml values_colors.arsc.flat)
aapt2_compile(res/drawable/ic_launcher_foreground.xml drawable_ic_launcher_foreground.xml.flat)
aapt2_compile(res/drawable/ic_launcher.xml drawable_ic_launcher.xml.flat)

# Link resources into an APK that doesn't have the libs yet
add_custom_command(
  OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-nolibs.zip"
  COMMENT "linking sensors_for_ros-nolibs.zip"
  DEPENDS
  ${aapt2_compiled_resources}
  COMMAND ${android_aapt2} link
  -o "${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-nolibs.zip"
  -I "${android_jar}"
  --manifest "${CMAKE_CURRENT_BINARY_DIR}/AndroidManifest.xml"
  -R
  ${aapt2_compiled_resources}
  --auto-add-overlay
  )

##############################
# Add compiled libs to the APK

file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib/${ANDROID_ABI})

# Copy cross-compiled libraries to a folder to include in the zip
set(copied_android_libs)
macro(copy_android_lib from_target lib_path)
  if(NOT TARGET ${from_target})
    message(FATAL_ERROR "${from_target} isn't a target. There must be a typo")
  endif()

  get_filename_component(libname "${lib_path}" NAME)
  add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/lib/${ANDROID_ABI}/${libname}"
    COMMENT "Copying ${lib_path} to lib/${ANDROID_ABI}/${libname}"
    DEPENDS
    "${from_target}"
    "${lib_path}"
    COMMAND
    ${CMAKE_COMMAND} -E copy_if_different
    "${lib_path}"
    "${CMAKE_CURRENT_BINARY_DIR}/lib/${ANDROID_ABI}/"
  )
  list(APPEND copied_android_libs "${CMAKE_CURRENT_BINARY_DIR}/lib/${ANDROID_ABI}/${libname}")
endmacro()

copy_android_lib(ros_for_android "ros_for_android/lib/libandroid-ros.so")
copy_android_lib(deps-cyclonedds "deps/lib/libddsc.so")
copy_android_lib(deps-rmw_dds_common "deps/lib/librmw_dds_common.so")
copy_android_lib(deps-libyaml_vendor "deps/lib/libyaml.so")

# Add all shared libraries to APK
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-unaligned-unsigned.zip
  COMMENT "Adding libs to sensors_for_ros-nolibs.zip to make sensors_for_ros-unaligned-unsigned.zip"
  DEPENDS
  ${copied_android_libs}
  ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-nolibs.zip
  WORKING_DIRECTORY
  "${CMAKE_CURRENT_BINARY_DIR}"
  COMMAND zip -D0ru 
  ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-nolibs.zip
  --output-file ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-unaligned-unsigned.zip
  lib/
  )

#############################
# Align to 4KiB page boundary
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-unsigned.zip
  COMMENT "Aligning make sensors_for_ros-unaligned-unsigned.zip to produce sensors_for_ros-unsigned.zip"
  DEPENDS
  ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-unaligned-unsigned.zip
  COMMAND ${android_zipalign} -p -f -v 4
  ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-unaligned-unsigned.zip
  ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-unsigned.zip
  )

# Sign the APK
if(NOT DEFINED $ENV{ANDROID_KEY_PASS})
  set(ENV{ANDROID_KEY_PASS} android)
endif()
if(NOT DEFINED $ENV{ANDROID_KEYSTORE_PASS})
  set(ENV{ANDROID_KEYSTORE_PASS} android)
endif()

add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros.apk
  COMMENT "Signing sensors_for_ros-unsigned.zip to make sensors_for_ros.apk"
  DEPENDS
  ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-unsigned.zip
  COMMAND "${CMAKE_COMMAND}" -E env
  "ANDROID_KEY_PASS=$ENV{ANDROID_KEY_PASS}"
  "ANDROID_KEYSTORE_PASS=$ENV{ANDROID_KEYSTORE_PASS}"
  "${android_apksigner}" sign
  --ks "${PATH_TO_KEYSTORE}"
  --ks-pass env:ANDROID_KEYSTORE_PASS
  --key-pass env:ANDROID_KEY_PASS
  --out "${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros.apk"
  "${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros-unsigned.zip"
  )

add_custom_target(apk ALL
  DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/sensors_for_ros.apk)
