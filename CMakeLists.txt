cmake_minimum_required(VERSION 3.5)
project(astra_camera)

# Default to C++14
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(std_msgs REQUIRED)
find_package(std_srvs REQUIRED)
find_package(cv_bridge REQUIRED)
find_package(image_geometry REQUIRED)
find_package(image_transport REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(message_filters REQUIRED)
find_package(camera_info_manager REQUIRED)
find_package(OpenCV REQUIRED)
find_package(Boost REQUIRED)
find_package(Threads REQUIRED)

# ROS IDL interface generation
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/DeviceInfo.msg"
  "msg/Extrinsics.msg"
  "msg/Metadata.msg"
  "srv/GetBool.srv"
  "srv/GetCameraInfo.srv"
  "srv/GetCameraParams.srv"
  "srv/GetDeviceInfo.srv"
  "srv/GetDouble.srv"
  "srv/GetInt32.srv"
  "srv/GetString.srv"
  "srv/SetInt32.srv"
  "srv/SetString.srv"
  DEPENDENCIES
  sensor_msgs
  std_msgs
)

# Platform detection
execute_process(COMMAND uname -m OUTPUT_VARIABLE MACHINES)
execute_process(COMMAND getconf LONG_BIT OUTPUT_VARIABLE MACHINES_BIT)
string(STRIP ${MACHINES} MACHINES)
string(STRIP ${MACHINES_BIT} MACHINES_BIT)

if(MACHINES STREQUAL "x86_64" AND MACHINES_BIT STREQUAL "64")
  set(HOST_PLATFORM "x64")
elseif(MACHINES MATCHES "arm" OR MACHINES MATCHES "aarch64")
  if(MACHINES_BIT STREQUAL "64")
    set(HOST_PLATFORM "arm64")
  else()
    set(HOST_PLATFORM "arm")
  endif()
endif()

set(ORBBEC_OPENNI2_REDIST "${CMAKE_CURRENT_SOURCE_DIR}/include/openni2_redist/${HOST_PLATFORM}")

# Library configuration - 修改目标名称避免冲突
add_library(${PROJECT_NAME}_lib SHARED
  src/point_cloud_proc/point_cloud_xyz.cpp
  src/point_cloud_proc/point_cloud_xyzrgb.cpp
  src/d2c_viewer.cpp
  src/ob_context.cpp
  src/ob_camera_info.cpp
  src/ob_camera_node_factory.cpp
  src/ob_camera_node.cpp
  src/ros_service.cpp
  src/utils.cpp
  src/uvc_camera_driver.cpp
)

target_include_directories(${PROJECT_NAME}_lib PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
  ${Boost_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
)

target_link_libraries(${PROJECT_NAME}_lib
  -lOpenNI2
  -ldw
  -L${ORBBEC_OPENNI2_REDIST}
  ${OpenCV_LIBS}
  ${Boost_LIBRARIES}
)

# 添加完整的依赖项
ament_target_dependencies(${PROJECT_NAME}_lib
  ament_cmake
  rclcpp
  sensor_msgs
  std_msgs
  std_srvs
  cv_bridge
  image_geometry
  image_transport
  tf2
  tf2_ros
  message_filters
  camera_info_manager
  OpenCV
  Boost
  Threads
)

# 安装配置需要同步修改
install(TARGETS
  ${PROJECT_NAME}_lib
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

ament_export_include_directories(include)
ament_export_libraries(${PROJECT_NAME}_lib)
ament_package()
