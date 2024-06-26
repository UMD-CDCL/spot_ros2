cmake_minimum_required(VERSION 3.22)

list(APPEND CMAKE_PREFIX_PATH /opt/spot-cpp-sdk)

project(spot_driver)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(THIS_PACKAGE_INCLUDE_ROS_DEPENDS
  cv_bridge
  geometry_msgs
  rclcpp
  rclcpp_components
  sensor_msgs
  tf2_ros
  tl_expected
  spot_msgs
)

find_package(ament_cmake REQUIRED)
find_package(ament_cmake_python REQUIRED)
find_package(bosdyn REQUIRED)
foreach(Dependency IN ITEMS ${THIS_PACKAGE_INCLUDE_ROS_DEPENDS})
  find_package(${Dependency} REQUIRED)
endforeach()

add_library(spot_api
  src/api/default_kinematic_api.cpp
  src/api/default_image_client.cpp
  src/api/default_spot_api.cpp
  src/api/default_time_sync_api.cpp
  src/api/spot_image_sources.cpp
  src/interfaces/rclcpp_logger_interface.cpp
  src/interfaces/rclcpp_parameter_interface.cpp
  src/interfaces/rclcpp_tf_interface.cpp
  src/interfaces/rclcpp_wall_timer_interface.cpp
  src/conversions/common_conversions.cpp
  src/conversions/kinematic_conversions.cpp
  src/kinematic/kinematic_node.cpp
  src/kinematic/kinematic_service.cpp
  src/kinematic/kinematic_middleware_handle.cpp
  src/images/spot_image_publisher.cpp
  src/images/images_middleware_handle.cpp
  src/images/spot_image_publisher_node.cpp
)
target_include_directories(spot_api PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)
target_link_libraries(spot_api PUBLIC bosdyn::bosdyn_client_static)
set_property(TARGET spot_api PROPERTY POSITION_INDEPENDENT_CODE ON)
ament_target_dependencies(spot_api PUBLIC ${THIS_PACKAGE_INCLUDE_ROS_DEPENDS})

# Create executable to allow running SpotImagePublisherNode directly as a ROS 2 node
add_executable(spot_image_publisher_node src/images/spot_image_publisher_node_main.cpp)
target_include_directories(spot_image_publisher_node
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(spot_image_publisher_node PUBLIC spot_api)

# Register a composable node to allow loading SpotImagePublisherNode in a component container
add_library(spot_image_publisher_component SHARED src/images/spot_image_publisher_component.cpp)
target_include_directories(spot_image_publisher_component
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(spot_image_publisher_component PUBLIC spot_api)

rclcpp_components_register_node(
  spot_image_publisher_component
  PLUGIN "spot_ros2::images::SpotImagePublisherNode"
  EXECUTABLE spot_image_publisher_node_component)

# Create executable to allow running KinematicNode directly as a ROS 2 node.
add_executable(kinematic_node src/kinematic/kinematic_node_main.cpp)
target_include_directories(kinematic_node
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(kinematic_node PUBLIC spot_api)

# Register a composable node to allow loading KinematicNode in a component container
add_library(kinematic_component SHARED src/kinematic/kinematic_component.cpp)
target_include_directories(kinematic_component
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(kinematic_component PUBLIC spot_api)

rclcpp_components_register_node(
  kinematic_component
  PLUGIN "spot_ros2::kinematic::KinematicNode"
  EXECUTABLE kinematic_node_component)

ament_python_install_package(${PROJECT_NAME})
install(
  PROGRAMS
  spot_driver/command_spot_driver.py
  DESTINATION lib/${PROJECT_NAME}
  RENAME command_spot
  )
install(
  PROGRAMS
  spot_driver/publish_cameras.py
  DESTINATION lib/${PROJECT_NAME}
  RENAME spot_publish_cameras
  )
install(
  PROGRAMS
  spot_driver/spot_ros2.py
  DESTINATION lib/${PROJECT_NAME}
  RENAME spot_ros2
  )
install(
  PROGRAMS
  spot_driver/spot_alerts.py
  DESTINATION lib/${PROJECT_NAME}
  RENAME spot_alerts
  )
# Install Libraries
install(
  TARGETS
    spot_api
    spot_image_publisher_component
    kinematic_component
  EXPORT ${PROJECT_NAME}Targets
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES DESTINATION include
)

# Install Binaries
install(
  TARGETS 
    spot_image_publisher_node
    spot_image_publisher_node_component
    kinematic_node
    kinematic_node_component
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION lib/${PROJECT_NAME}
)

# Install include directory
install(DIRECTORY include/ DESTINATION include)

# Install launch and configurations
install(DIRECTORY config DESTINATION share/${PROJECT_NAME})
install(DIRECTORY launch DESTINATION share/${PROJECT_NAME})
install(DIRECTORY rviz DESTINATION share/${PROJECT_NAME})

ament_export_targets(${PROJECT_NAME}Targets HAS_LIBRARY_TARGET)
ament_export_dependencies(${THIS_PACKAGE_INCLUDE_DEPENDS})

if(BUILD_TESTING)
  add_subdirectory(test)
endif()

ament_package()
