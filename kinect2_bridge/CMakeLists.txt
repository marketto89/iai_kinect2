cmake_minimum_required(VERSION 2.8.3)
project(kinect2_bridge)
SET(CMAKE_BUILD_TYPE Release)

option(K2_USE_OPENCL_REGISTRATION "Use OpenCL implementation of the depth registration"       OFF)
option(K2_USE_OPENCL_PIPELINE     "Use OpenCL implementation of libfreenect2 packet pipeline" OFF)
option(K2_USE_TIFF_COMRESSION     "Use TIFF for depth/ir compression instead of PNG."         OFF)
set(K2_MIN_WORKER_THREADS 2 CACHE STRING "Minimum number of worker threads")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DBT_USE_DOUBLE_PRECISION -Wall")
# Unused warnings
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wuninitialized -Winit-self -Wunused-function -Wunused-label -Wunused-variable -Wunused-but-set-variable -Wunused-but-set-parameter")
# Additional warnings
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Warray-bounds -Wtype-limits -Wreturn-type -Wsequence-point -Wparentheses -Wmissing-braces -Wchar-subscripts -Wswitch -Wwrite-strings -Wenum-compare -Wempty-body -Wlogical-op")

#Jetson patched libusb
LINK_DIRECTORIES("lib")

# Check for c++11 support
INCLUDE(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
IF(COMPILER_SUPPORTS_CXX11)
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
ELSEIF(COMPILER_SUPPORTS_CXX0X)
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
ELSE()
  MESSAGE(ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
ENDIF()

find_package(freenect2 CONFIG)

find_package(catkin REQUIRED COMPONENTS roscpp rostime std_msgs sensor_msgs cv_bridge compressed_depth_image_transport depth_registration)
find_package(cmake_modules QUIET)

## System dependencies are found with CMake's conventions
find_package(OpenCV REQUIRED)
find_package(OpenMP)
find_package(PCL REQUIRED)

if(OPENMP_FOUND)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

catkin_package(
  INCLUDE_DIRS include
#  LIBRARIES kinect2_bridge
#  CATKIN_DEPENDS other_catkin_pkg
#  DEPENDS system_lib
)

###########
## Build ##
###########

# Display additional files in qtcreator
execute_process(COMMAND find ${PROJECT_SOURCE_DIR} -type f OUTPUT_VARIABLE FILES_STRING)
string(REPLACE "\n" ";" FILES_LIST ${FILES_STRING})
add_custom_target(additional_files_${PROJECT_NAME}
  SOURCES
  ${FILES_LIST}
  ${PROJECT_SOURCE_DIR}/../README.md
)

include_directories(include
  ${freenect2_INCLUDE_DIRS}
  ${catkin_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIR}
  ${PCL_INCLUDE_DIRS}
)

if(K2_USE_OPENCL_REGISTRATION)
  add_definitions(-DUSE_OPENCL_REGISTRATION)
endif()
if(K2_USE_OPENCL_PIPELINE)
  add_definitions(-DUSE_OPENCL_PIPELINE)
endif()
if(K2_USE_TIFF_COMRESSION)
  add_definitions(-DUSE_TIFF_COMRESSION)
endif()
add_definitions(-DMIN_WORKER_THREADS=${K2_MIN_WORKER_THREADS})
add_definitions(-DK2_CALIB_PATH="${PROJECT_SOURCE_DIR}/data/")

add_executable(kinect2_bridge src/kinect2_bridge.cpp)
target_link_libraries(kinect2_bridge
  /usr/lib/arm-linux-gnueabihf/tegra/libjpeg.so
  usb-1.0
  ${freenect2_LIBRARY}
  ${catkin_LIBRARIES}
  ${OpenCV_LIBRARIES}
  ${PCL_LIBRARIES}
)

