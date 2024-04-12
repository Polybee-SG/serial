cmake_minimum_required(VERSION 3.5)
project(serial)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to Release as none was specified.")
  set(CMAKE_BUILD_TYPE "Release" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# Default to C++17
if(NOT CMAKE_CXX_STANDARD)
  if ("cxx_std_17" IN_LIST CMAKE_CXX_COMPILE_FEATURES)
    set(CMAKE_CXX_STANDARD 17)
  else()
    message( FATAL_ERROR "cxx_std_17 could not be found.")
  endif()
endif()

# Defaults for Microsoft C++ compiler
if(MSVC)
  # https://blog.kitware.com/create-dlls-on-windows-without-declspec-using-new-cmake-export-all-feature/
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

  # Enable Math Constants
  # https://docs.microsoft.com/en-us/cpp/c-runtime-library/math-constants?view=vs-2019
  add_compile_definitions(
    _USE_MATH_DEFINES
  )
endif()

if(APPLE)
  find_library(IOKIT_LIBRARY IOKit)
  find_library(FOUNDATION_LIBRARY Foundation)
endif()

# Update the policy setting to avoid an error when loading the ament_cmake package
# at the current cmake version level
if(POLICY CMP0057)
  cmake_policy(SET CMP0057 NEW)
endif()

find_package(ament_cmake QUIET)

if ( ament_cmake_FOUND )

  message(STATUS "------------------------------------------")
  message(STATUS "serial is being built using AMENT.")
  message(STATUS "------------------------------------------")

  ament_export_include_directories(include)
  ament_export_libraries(${PROJECT_NAME})
  ament_package()

  set( SERIAL_LIB_DESTINATION   lib )
  set( SERIAL_INC_DESTINATION   include )
  set( SERIAL_BIN_DESTINATION   bin )

elseif( CATKIN_DEVEL_PREFIX OR CATKIN_BUILD_BINARY_PACKAGE)

  set(catkin_FOUND 1)
  add_definitions( -DUSING_ROS )

  message(STATUS "------------------------------------------")
  message(STATUS "serial is being built using CATKIN.")
  message(STATUS "------------------------------------------")

  find_package(catkin REQUIRED)

  if(UNIX AND NOT APPLE)
    # If Linux, add rt and pthread
    set(rt_LIBRARIES rt)
    set(pthread_LIBRARIES pthread)
    catkin_package(
      LIBRARIES ${PROJECT_NAME}
      INCLUDE_DIRS include
      DEPENDS rt pthread
    )
  else()
    # Otherwise normal call
    catkin_package(
      LIBRARIES ${PROJECT_NAME}
      INCLUDE_DIRS include
    )
  endif()

  set( SERIAL_LIB_DESTINATION   ${CATKIN_PACKAGE_LIB_DESTINATION} )
  set( SERIAL_INC_DESTINATION   ${CATKIN_GLOBAL_INCLUDE_DESTINATION} )
  set( SERIAL_BIN_DESTINATION   ${CATKIN_GLOBAL_BIN_DESTINATION} )

endif()

## Sources
set(serial_SRCS
  src/serial.cc
  include/serial/serial.h
  include/serial/v8stdint.h
)
if(APPLE)
  # If OSX
  list(APPEND serial_SRCS src/impl/unix.cc)
  list(APPEND serial_SRCS src/impl/list_ports/list_ports_osx.cc)
elseif(UNIX)
  # If unix
  list(APPEND serial_SRCS src/impl/unix.cc)
  list(APPEND serial_SRCS src/impl/list_ports/list_ports_linux.cc)
else()
  # If windows
  list(APPEND serial_SRCS src/impl/win.cc)
  list(APPEND serial_SRCS src/impl/list_ports/list_ports_win.cc)
endif()

## Add serial library
add_library(${PROJECT_NAME} ${serial_SRCS})
if(APPLE)
  target_link_libraries(${PROJECT_NAME} ${FOUNDATION_LIBRARY} ${IOKIT_LIBRARY})
elseif(UNIX)
  target_link_libraries(${PROJECT_NAME} rt pthread)
else()
  target_link_libraries(${PROJECT_NAME} setupapi)
endif()

## Uncomment for example
add_executable(serial_example examples/serial_example.cc)
add_dependencies(serial_example ${PROJECT_NAME})
target_link_libraries(serial_example ${PROJECT_NAME})

## Include headers
include_directories(include)

## Install executable
INSTALL(TARGETS ${PROJECT_NAME}
  ARCHIVE DESTINATION ${SERIAL_LIB_DESTINATION}
  LIBRARY DESTINATION ${SERIAL_LIB_DESTINATION}
  RUNTIME DESTINATION ${SERIAL_BIN_DESTINATION}
)

## Install headers
INSTALL(FILES include/serial/serial.h include/serial/v8stdint.h
  DESTINATION ${SERIAL_INC_DESTINATION}/serial)

## Tests
if(catkin_FOUND)
  if (CATKIN_ENABLE_TESTING)
    add_subdirectory(tests)
  endif()
endif()
