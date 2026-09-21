cmake_minimum_required(VERSION 3.16)
project(urg_node2)

# Default to C99
if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

# Default to C++17
# Lyrical and later build the ROS 2 core as C++20, but this package only
# requires C++17. Leaving CMAKE_CXX_STANDARD unset by the caller means the
# distribution's own default (C++20 from ament_ros_defaults on Lyrical) wins.
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_components REQUIRED)
find_package(rclcpp_lifecycle REQUIRED)
find_package(lifecycle_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(diagnostic_msgs REQUIRED)
find_package(diagnostic_updater REQUIRED)
find_package(laser_proc REQUIRED)

# Dependencies linked into urg_node2 and its test.
# Modern CMake targets are used instead of ament_target_dependencies(), which
# is deprecated from Kilted onwards.
# laser_proc still exports the old-style ament variables only (it defines no
# laser_proc_TARGETS), so it is linked through laser_proc_LIBRARIES.
set(URG_NODE2_DEPENDENCIES
  ${rclcpp_TARGETS}
  ${rclcpp_components_TARGETS}
  ${rclcpp_lifecycle_TARGETS}
  ${lifecycle_msgs_TARGETS}
  ${sensor_msgs_TARGETS}
  ${diagnostic_msgs_TARGETS}
  ${diagnostic_updater_TARGETS}
  ${laser_proc_LIBRARIES}
)

set(URG_LIBRARY_DIR urg_library/current)
set(URG_LIBRARY_C_INCLUDE_DIR ${URG_LIBRARY_DIR}/include/c)
set(URG_LIBRARY_SRC_DIR ${URG_LIBRARY_DIR}/src)

include_directories(include ${URG_LIBRARY_C_INCLUDE_DIR} ${laser_proc_INCLUDE_DIRS})

add_library(urg_c SHARED
  ${URG_LIBRARY_SRC_DIR}/urg_sensor.c
  ${URG_LIBRARY_SRC_DIR}/urg_utils.c
  ${URG_LIBRARY_SRC_DIR}/urg_debug.c
  ${URG_LIBRARY_SRC_DIR}/urg_connection.c
  ${URG_LIBRARY_SRC_DIR}/urg_ring_buffer.c
  ${URG_LIBRARY_SRC_DIR}/urg_serial.c
  ${URG_LIBRARY_SRC_DIR}/urg_serial_utils.c
  ${URG_LIBRARY_SRC_DIR}/urg_tcpclient.c
)
# urg_library relies on POSIX/GNU declarations (M_PI, fd_set, ...) that a
# strict -std=c99 does not expose.
set_target_properties(urg_c PROPERTIES C_EXTENSIONS ON)

add_library(urg_node2 SHARED src/urg_node2.cpp)
target_link_libraries(urg_node2 PUBLIC ${URG_NODE2_DEPENDENCIES})
target_link_libraries(urg_node2 PRIVATE urg_c)
rclcpp_components_register_node(urg_node2
  PLUGIN "urg_node2::UrgNode2"
  EXECUTABLE urg_node2_node)

install(TARGETS
  urg_node2 urg_c
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY launch config
  DESTINATION share/${PROJECT_NAME}
)

if(BUILD_TESTING)
  find_package(ament_cmake_gtest)
  ament_add_gtest(urg_node2_test src/urg_node2.cpp test/urg_node2_test.cpp TIMEOUT 200)
  target_link_libraries(urg_node2_test ${URG_NODE2_DEPENDENCIES} urg_c)
endif()

# disable tool tests, because a lot of errors occur in urg_library
#if(BUILD_TESTING)
#  find_package(ament_lint_auto REQUIRED)
#  # the following line skips the linter which checks for copyrights
#  # uncomment the line when a copyright and license is not present in all source files
#  #set(ament_cmake_copyright_FOUND TRUE)
#  # the following line skips cpplint (only works in a git repo)
#  # uncomment the line when this package is not in a git repo
#  #set(ament_cmake_cpplint_FOUND TRUE)
#  ament_lint_auto_find_test_dependencies()
#endif()

ament_package()
