cmake_minimum_required(VERSION 3.5)
project(orbslam3)

# (optional) keep if you need it
set(ENV{PYTHONPATH} "/opt/ros/humble/lib/python3.10/site-packages/")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules)

# C++ standard
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# -------- Find packages --------
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(cv_bridge REQUIRED)
find_package(message_filters REQUIRED)
find_package(Sophus REQUIRED)
find_package(Pangolin REQUIRED)
find_package(ORB_SLAM3 REQUIRED)    # via your CMakeModules/FindORB_SLAM3.cmake
find_package(OpenCV 4 REQUIRED COMPONENTS core imgproc features2d calib3d highgui)
find_package(OpenSSL REQUIRED)      # ORB-SLAM3 uses OpenSSL (MD5) on Ubuntu 22.04

# Prefer include dir/library vars if the Find script exports them
include_directories(
  include
  ${ORB_SLAM3_ROOT_DIR}   
  ${ORB_SLAM3_INCLUDE_DIR}            # if defined by FindORB_SLAM3
  ${ORB_SLAM3_ROOT_DIR}/include       # fallback
  ${ORB_SLAM3_ROOT_DIR}/include/CameraModels
  ${OpenCV_INCLUDE_DIRS}
  ${ORB_SLAM3_ROOT_DIR}/Thirdparty/DBoW2
  ${ORB_SLAM3_ROOT_DIR}/Thirdparty/g2o
)

# Help linker find libORB_SLAM3.so if you didn't 'make install'
link_directories(
  ${ORB_SLAM3_ROOT_DIR}/lib
  ${ORB_SLAM3_ROOT_DIR}/build/lib
  /usr/local/lib
)

# Choose Pangolin target or variable, depending on how it’s exported
if (TARGET Pangolin::pangolin)
  set(PANGO_LIB Pangolin::pangolin)
else()
  set(PANGO_LIB ${Pangolin_LIBRARIES})
endif()

# Choose ORB_SLAM3 libs as exported by your Find script
if (DEFINED ORB_SLAM3_LIBRARIES)
  set(ORB3_LIBS ${ORB_SLAM3_LIBRARIES})
else()
  set(ORB3_LIBS ORB_SLAM3)           # fallback target name
endif()

# Keep indirect libs from being dropped by --as-needed
set(_LDFLAGS -Wl,--no-as-needed)

# ------------- Executables -------------

add_executable(mono
  src/monocular/mono.cpp
  src/monocular/monocular-slam-node.cpp
)
ament_target_dependencies(mono rclcpp sensor_msgs cv_bridge)
target_link_libraries(mono
  ${ORB3_LIBS}
  ${OpenCV_LIBS}         # includes opencv_calib3d
  ${PANGO_LIB}
  OpenSSL::Crypto
  pthread
)
target_link_options(mono PRIVATE ${_LDFLAGS})

add_executable(rgbd
  src/rgbd/rgbd.cpp
  src/rgbd/rgbd-slam-node.cpp
)
ament_target_dependencies(rgbd rclcpp sensor_msgs cv_bridge message_filters)
target_link_libraries(rgbd
  ${ORB3_LIBS}
  ${OpenCV_LIBS}
  ${PANGO_LIB}
  OpenSSL::Crypto
  pthread
)
target_link_options(rgbd PRIVATE ${_LDFLAGS})

add_executable(stereo
  src/stereo/stereo.cpp
  src/stereo/stereo-slam-node.cpp
)
ament_target_dependencies(stereo rclcpp sensor_msgs cv_bridge message_filters)
target_link_libraries(stereo
  ${ORB3_LIBS}
  ${OpenCV_LIBS}
  ${PANGO_LIB}
  OpenSSL::Crypto
  pthread
)
target_link_options(stereo PRIVATE ${_LDFLAGS})

add_executable(stereo-inertial
  src/stereo-inertial/stereo-inertial.cpp
  src/stereo-inertial/stereo-inertial-node.cpp
)
ament_target_dependencies(stereo-inertial rclcpp sensor_msgs cv_bridge)
target_link_libraries(stereo-inertial
  ${ORB3_LIBS}
  ${OpenCV_LIBS}
  ${PANGO_LIB}
  OpenSSL::Crypto
  pthread
)
target_link_options(stereo-inertial PRIVATE ${_LDFLAGS})

install(TARGETS mono rgbd stereo stereo-inertial
  DESTINATION lib/${PROJECT_NAME})

ament_package()

