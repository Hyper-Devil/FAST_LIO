cmake_minimum_required(VERSION 3.0.0)
project(fast_lio VERSION 1.0.0)

# 构建类型设置
# 使用 Debug 模式进行断点调试: catkin_make -DCMAKE_BUILD_TYPE=Debug
# 使用 Release 模式获得更佳性能: catkin_make -DCMAKE_BUILD_TYPE=Release
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif()
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

# 统一设置 C++ 标准
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 设置编译选项
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fexceptions")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread -fexceptions")

# 添加定义
add_definitions(-DROOT_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/\")

# 多处理器支持设置
message(STATUS "Current CPU architecture: ${CMAKE_SYSTEM_PROCESSOR}")
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)" )
  include(ProcessorCount)
  ProcessorCount(N)
  message(STATUS "Processor number: ${N}")
  if(N GREATER 4)
    add_definitions(-DMP_EN)
    add_definitions(-DMP_PROC_NUM=3)
    message(STATUS "Core for MP: 3")
  elseif(N GREATER 3)
    add_definitions(-DMP_EN)
    add_definitions(-DMP_PROC_NUM=2)
    message(STATUS "Core for MP: 2")
  else()
    add_definitions(-DMP_PROC_NUM=1)
  endif()
else()
  add_definitions(-DMP_PROC_NUM=1)
endif()

# 查找 OpenMP 并配置
find_package(OpenMP QUIET)
if(OpenMP_FOUND)
  message(STATUS "Found OpenMP")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
endif()

# 查找依赖包
find_package(PythonLibs REQUIRED)
find_path(MATPLOTLIB_CPP_INCLUDE_DIRS "matplotlibcpp.h")

find_package(catkin REQUIRED COMPONENTS
  geometry_msgs
  nav_msgs
  sensor_msgs
  roscpp
  rospy
  std_msgs
  pcl_ros
  tf
  livox_ros_driver
  message_generation
  eigen_conversions
)

find_package(Eigen3 REQUIRED)
find_package(PCL 1.8 REQUIRED)

message(STATUS "Eigen: ${EIGEN3_INCLUDE_DIR}")

# 消息定义
add_message_files(
  FILES
  Pose6D.msg
)

generate_messages(
  DEPENDENCIES
  geometry_msgs
)

# 包配置
catkin_package(
  CATKIN_DEPENDS geometry_msgs nav_msgs roscpp rospy std_msgs message_runtime
  DEPENDS EIGEN3 PCL
  INCLUDE_DIRS include
)

# 设置包含目录
include_directories(
  ${catkin_INCLUDE_DIRS} 
  ${EIGEN3_INCLUDE_DIR}
  ${PCL_INCLUDE_DIRS}
  ${PYTHON_INCLUDE_DIRS}
  include
)

# 构建目标
add_executable(fastlio_mapping 
  src/laserMapping.cpp 
  include/ikd-Tree/ikd_Tree.cpp 
  src/preprocess.cpp
)

# 目标链接库
target_link_libraries(fastlio_mapping 
  ${catkin_LIBRARIES} 
  ${PCL_LIBRARIES} 
  ${PYTHON_LIBRARIES}
)

# 目标特定包含目录
target_include_directories(fastlio_mapping PRIVATE 
  ${PYTHON_INCLUDE_DIRS}
)

# 根据构建类型设置不同的编译选项
if(CMAKE_BUILD_TYPE STREQUAL "Release")
  message(STATUS "Configuring for Release mode with optimizations")
  target_compile_options(fastlio_mapping PRIVATE -O3)
elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
  message(STATUS "Configuring for Debug mode with debug symbols")
  target_compile_options(fastlio_mapping PRIVATE -g -O0)
endif()