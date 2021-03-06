cmake_minimum_required(VERSION 2.8.3)
project(ati_force_torque)

find_package(catkin REQUIRED COMPONENTS
  cmake_modules
  cob_generic_can
  dynamic_reconfigure 
  gazebo_ros
  geometry_msgs
  iirob_filters
  message_generation
  roscpp
  roslaunch
  rosparam_handler
  rospy
  std_msgs
  std_srvs
  teleop_twist_joy
  tf2
  tf2_geometry_msgs
  tf2_ros
  trajectory_msgs
  visualization_msgs
  realtime_tools
  filters
)

find_package(Eigen3)
if(NOT EIGEN3_FOUND)
  # Fallback to cmake_modules
  find_package(cmake_modules REQUIRED)
  find_package(Eigen REQUIRED)
else()
  set(Eigen_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIR})
  set(Eigen_LIBRARIES ${EIGEN3_LIBRARIES})
endif()

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" Cpp11CompilerFlag)
if (Cpp11CompilerFlag)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif()

generate_ros_parameter_files(
    cfg/CoordinateSystemCalibration.params
    cfg/CanConfiguration.params
    cfg/FTSConfiguration.params
    cfg/NodeConfiguration.params
    cfg/PublishConfiguration.params
    cfg/Calibration.params
)

add_service_files(
  FILES
  CalculateAverageMasurement.srv
  CalculateSensorOffset.srv
  DiagnosticVoltages.srv
)

generate_messages(
  DEPENDENCIES
  geometry_msgs
  std_msgs
)

###################################
## catkin specific configuration ##
###################################
catkin_package(
 INCLUDE_DIRS common/include ros/include
 CATKIN_DEPENDS cob_generic_can geometry_msgs iirob_filters message_runtime roscpp std_msgs std_srvs rospy tf2 tf2_ros tf2_geometry_msgs trajectory_msgs realtime_tools filters
 DEPENDS Eigen
 LIBRARIES ${PROJECT_NAME}
)

###########
## Build ##
###########

include_directories(
  common/include
  ros/include
  ${catkin_INCLUDE_DIRS}
  ${Eigen_INCLUDE_DIRS}
)

add_library(${PROJECT_NAME} common/src/ForceTorqueCtrl.cpp ros/src/force_torque_sensor.cpp ros/src/force_torque_sensor_sim.cpp ros/src/force_torque_sensor_handle.cpp ros/src/force_torque_sensor_handle_sim.cpp)
add_dependencies(${PROJECT_NAME}
  ${PROJECT_NAME}_gencfg  # For dynamic_reconfigure
  ${PROJECT_NAME}_generate_messages_cpp
  ${PROJECT_NAME}_genparam # For rosparam_handler
  )
target_link_libraries(${PROJECT_NAME} ${catkin_LIBRARIES} ${Eigen_LIBRARIES})

add_executable(${PROJECT_NAME}_node ros/src/force_torque_sensor_node.cpp)
add_dependencies(${PROJECT_NAME}_node ${PROJECT_NAME}_generate_messages_cpp)
target_link_libraries(${PROJECT_NAME}_node ${PROJECT_NAME} ${catkin_LIBRARIES} ${Eigen_LIBRARIES})

add_executable(${PROJECT_NAME}_config ros/src/force_torque_sensor_config.cpp)
target_link_libraries(${PROJECT_NAME}_config ${PROJECT_NAME} ${catkin_LIBRARIES} ${Eigen_LIBRARIES})
add_dependencies(${PROJECT_NAME}_config ${PROJECT_NAME} ${catkin_LIBRARIES} ${Eigen_LIBRARIES} ${PROJECT_NAME}_generate_messages_cpp)

#############
## Install ##
#############
install(TARGETS ${PROJECT_NAME} ${PROJECT_NAME}_node ${PROJECT_NAME}_config
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

install(DIRECTORY common/include ros/include
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  FILES_MATCHING PATTERN "*.h"
  PATTERN ".svn" EXCLUDE
)

install(DIRECTORY ros/launch config description
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
)

#############
## Tests ##
#############

roslaunch_add_file_check(ros/launch)
