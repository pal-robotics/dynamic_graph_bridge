# Copyright (C) 2008-2013 LAAS-CNRS, JRL AIST-CNRS.
#
# Author: Florent Lamiraux, Nirmal Giftsun
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.

# Catkin part


CMAKE_MINIMUM_REQUIRED(VERSION 2.8.3)

project(dynamic_graph_bridge)

find_package(Bullet REQUIRED)
find_package(catkin REQUIRED COMPONENTS sot_cmake sot-core roscpp rospy std_msgs message_generation std_srvs geometry_msgs sensor_msgs tf realtime_tools jrl_dynamics_urdf dynamic-graph dynamic-graph-python sot-dynamic)
find_package(Boost REQUIRED COMPONENTS program_options)


add_message_files(FILES Matrix.msg Vector.msg)

add_service_files(FILES  RunCommand.srv RunPythonFile.srv)

generate_messages(
  DEPENDENCIES std_msgs
)

catkin_package(
	INCLUDE_DIRS include
	LIBRARIES ros_bridge ros_interpreter
	CATKIN_DEPENDS message_runtime
    DEPENDS dynamic-graph dynamic-graph-python sot-dynamic
)

include_directories(${catkin_INCLUDE_DIRS} ${Bullet_INCLUDE_DIRS})
link_directories(${DYNAMIC_GRAPH_PLUGINDIR})

## LAAS cmake submodule part
set(PROJECT_DESCRIPTION "Dynamic graph bridge library")
set(PROJECT_NAME dynamic_graph_bridge)
set(PROJECT_URL "abc")

set(CXX_DISABLE_WERROR False)
set(CUSTOM_HEADER_DIR dynamic_graph_bridge)
set(${PROJECT_NAME}_HEADERS
  include/dynamic_graph_bridge/ros_init.hh
  include/dynamic_graph_bridge/ros_interpreter.hh
  )
include(${SOT_CMAKE_DIR}/base.cmake)
include(${SOT_CMAKE_DIR}/GNUInstallDirs.cmake)
include(${SOT_CMAKE_DIR}/python.cmake)

SETUP_PROJECT()

set(LIBRARY_OUTPUT_PATH ${CATKIN_DEVEL_PREFIX}/lib)
set(CMAKE_INSTALL_RPATH "${LIBRARY_OUTPUT_PATH}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${LIBRARY_OUTPUT_PATH}")
set(PKG_CONFIG_ADDITIONAL_VARIABLES
  ${PKG_CONFIG_ADDITIONAL_VARIABLES}
  plugindirname
  plugindir
  )
 
add_library(ros_bridge
  src/converter.hh
  include/dynamic_graph_bridge/ros_init.hh src/ros_init.cpp
  src/sot_to_ros.hh src/sot_to_ros.cpp
  )
target_link_libraries(ros_bridge ${catkin_LIBRARIES})

install(TARGETS ros_bridge DESTINATION lib)

# Make sure rpath are preserved during the install as ROS dependencies
# are not installed.
set_target_properties(ros_bridge PROPERTIES BUILD_WITH_INSTALL_RPATH True)

macro(compile_plugin NAME)
  message(lib path ${LIBRARY_OUTPUT_PATH})
  file(MAKE_DIRECTORY "${LIBRARY_OUTPUT_PATH}/dynamic_graph/ros/${NAME}")
  add_library(${NAME} src/${NAME}.cpp src/${NAME}.hh)
  add_dependencies(${NAME} ros_bridge)
  target_link_libraries(${NAME} ros_bridge ${catkin_LIBRARIES})
#  set_target_properties(${NAME} PROPERTIES BUILD_WITH_INSTALL_RPATH True)
  install(TARGETS ${NAME} DESTINATION lib)
  

  dynamic_graph_python_module("ros/${NAME}"
    ""
    ros/${NAME}/wrap
    )
  target_link_libraries(ros/${NAME}/wrap ${NAME})


endmacro()

compile_plugin(ros_publish)
compile_plugin(ros_subscribe)
compile_plugin(ros_time)
compile_plugin(ros_joint_state)

target_link_libraries(ros_joint_state dynamic)

compile_plugin(robot_model)
target_link_libraries(robot_model ${catkin_LIBRARIES})

# ros_interperter library.
add_library(ros_interpreter src/ros_interpreter.cpp)

add_dependencies(ros_interpreter ros_bridge)
target_link_libraries(ros_interpreter ros_bridge)
set_target_properties(ros_interpreter PROPERTIES BUILD_WITH_INSTALL_RPATH True)
message(cmakeinstalllibdir ${CMAKE_INSTALL_LIBDIR} )
install(TARGETS ros_interpreter DESTINATION ${CMAKE_INSTALL_LIBDIR})

# Stand alone remote dynamic-graph Python interpreter.
add_executable(interpreter src/interpreter.cpp)
add_dependencies(interpreter ros_interpreter)
target_link_libraries(interpreter ros_interpreter)

# set_target_properties(interpreter PROPERTIES BUILD_WITH_INSTALL_RPATH True)
#install(TARGETS interpreter DESTINATION bin)

# Stand alone embedded intepreter with a robot controller.
add_executable(geometric_simu src/geometric_simu.cpp src/sot_loader.cpp)
target_link_libraries(geometric_simu  ${Boost_LIBRARIES} ${catkin_LIBRARIES})

add_subdirectory(src)

#install ros executables
install(PROGRAMS
  scripts/robot_pose_publisher
  scripts/run_command
  scripts/tf_publisher
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
  )

install(TARGETS ros_bridge ros_interpreter
  DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  )

SETUP_PROJECT_FINALIZE()

