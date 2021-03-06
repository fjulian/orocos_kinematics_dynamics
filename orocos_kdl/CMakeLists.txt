#
# Test CMake version
#
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.12)
IF(POLICY CMP0048)
  CMAKE_POLICY(SET CMP0048 NEW)
ENDIF()
#MARK_AS_ADVANCED( FORCE CMAKE_BACKWARDS_COMPATIBILITY )


###################################################
#                                                 #
#       Start project customization section       #
#                                                 #
###################################################

PROJECT(orocos_kdl)
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()

SET( KDL_VERSION 1.4.0)
STRING( REGEX MATCHALL "[0-9]+" KDL_VERSIONS ${KDL_VERSION} )
LIST( GET KDL_VERSIONS 0 KDL_VERSION_MAJOR)
LIST( GET KDL_VERSIONS 1 KDL_VERSION_MINOR)
LIST( GET KDL_VERSIONS 2 KDL_VERSION_PATCH)

MESSAGE( STATUS "Orocos KDL version ${VERSION} (${KDL_VERSION_MAJOR}.${KDL_VERSION_MINOR}.${KDL_VERSION_PATCH})" )

SET( PROJ_SOURCE_DIR ${orocos_kdl_SOURCE_DIR} )
SET( PROJ_BINARY_DIR ${orocos_kdl_BINARY_DIR} )


IF(NOT CMAKE_INSTALL_PREFIX)
  SET( CMAKE_INSTALL_PREFIX /usr/local/ CACHE PATH "Installation directory" FORCE)
  MESSAGE( STATUS "Setting installation directory to ${CMAKE_INSTALL_PREFIX}" )
ENDIF(NOT CMAKE_INSTALL_PREFIX)

SET(CMAKE_VERSION "${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}")


IF ( NOT CMAKE_BUILD_TYPE )
  SET(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None(CMAKE_CXX_FLAGS or CMAKE_C_FLAGS used) Debug Release RelWithDebInfo MinSizeRel." FORCE)
  MESSAGE( STATUS "Setting build type to '${CMAKE_BUILD_TYPE}'" )
ELSE ( NOT CMAKE_BUILD_TYPE )
  MESSAGE( STATUS "Build type set to '${CMAKE_BUILD_TYPE}' by user." )
ENDIF ( NOT CMAKE_BUILD_TYPE )

SET( KDL_CFLAGS "")

find_package(eigen3_cmake_module QUIET)
find_package(Eigen3 QUIET)
message(STATUS "Using Eigen3 include dirs: ${Eigen3_INCLUDE_DIRS}")
include_directories(${Eigen3_INCLUDE_DIRS})
SET(KDL_CFLAGS "${KDL_CFLAGS} -I${Eigen3_INCLUDE_DIRS}")

# Check the platform STL containers capabilities
include(config/CheckSTLContainers.cmake)
CHECK_STL_CONTAINERS()

# Set the default option appropriately
if(HAVE_STL_CONTAINER_INCOMPLETE_TYPES)
    set(KDL_USE_NEW_TREE_INTERFACE_DEFAULT Off)
else(HAVE_STL_CONTAINER_INCOMPLETE_TYPES)
    set(KDL_USE_NEW_TREE_INTERFACE_DEFAULT On)
endif(HAVE_STL_CONTAINER_INCOMPLETE_TYPES)

# Allow the user to select the Tree API version to use
set(KDL_USE_NEW_TREE_INTERFACE ${KDL_USE_NEW_TREE_INTERFACE_DEFAULT} CACHE BOOL "Use the new KDL Tree interface")

# The new interface requires the use of shared pointers
if(KDL_USE_NEW_TREE_INTERFACE)
    # We need shared_ptr from boost since not all compilers are c++11 capable
    find_package(Boost REQUIRED)
    include_directories(${Boost_INCLUDE_DIRS})
    set(KDL_INCLUDE_DIRS ${Boost_INCLUDE_DIRS})
endif(KDL_USE_NEW_TREE_INTERFACE)

INCLUDE (${PROJ_SOURCE_DIR}/config/DependentOption.cmake)

OPTION(ENABLE_TESTS OFF "Enable building of tests")
IF( ENABLE_TESTS )
  # If not in standard paths, set CMAKE_xxx_PATH's in environment, eg.
  # export CMAKE_INCLUDE_PATH=/opt/local/include
  # export CMAKE_LIBRARY_PATH=/opt/local/lib
  FIND_LIBRARY(CPPUNIT cppunit)
  SET(CPPUNIT ${CPPUNIT} "dl")
  FIND_PATH(CPPUNIT_HEADERS cppunit/TestRunner.h)
  IF ( CPPUNIT AND CPPUNIT_HEADERS)
    MESSAGE( STATUS "-- Looking for Cppunit - found")
  ELSE ( CPPUNIT AND CPPUNIT_HEADERS )
    MESSAGE( FATAL_ERROR "-- Looking for Cppunit - not found")
  ENDIF ( CPPUNIT AND CPPUNIT_HEADERS )
ENDIF(ENABLE_TESTS )

# Disable Windows Compiler warnings
# see https://msdn.microsoft.com/en-us/library/thxezb7y.aspx
IF(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
  add_definitions(
    "/W3"
    "/wd4244"
    "/wd4267"
    "/wd4800"
    "/wd4996")
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  # https://github.com/orocos/orocos_kinematics_dynamics/issues/104
  add_definitions(
    "-Wno-deprecated-declarations"
  )
endif()

OPTION(ENABLE_EXAMPLES OFF "Enable building of examples")

ADD_SUBDIRECTORY( doc )
ADD_SUBDIRECTORY( src )
ADD_SUBDIRECTORY( tests )
ADD_SUBDIRECTORY( models )
ADD_SUBDIRECTORY( examples )


export(TARGETS orocos-kdl
  FILE "${PROJECT_BINARY_DIR}/OrocosKDLTargets.cmake")

export(PACKAGE orocos_kdl)

set(KDL_INCLUDE_DIRS ${KDL_INCLUDE_DIRS} ${Eigen3_INCLUDE_DIRS})

CONFIGURE_FILE(KDLConfig.cmake.in
  ${PROJECT_BINARY_DIR}/orocos_kdl-config.cmake @ONLY)
CONFIGURE_FILE(KDLConfigVersion.cmake.in
  ${PROJECT_BINARY_DIR}/orocos_kdl-config-version.cmake @ONLY)

INSTALL(FILES ${PROJECT_BINARY_DIR}/orocos_kdl-config.cmake DESTINATION share/orocos_kdl/cmake)
INSTALL(FILES ${PROJECT_BINARY_DIR}/orocos_kdl-config-version.cmake DESTINATION share/orocos_kdl/cmake)
INSTALL(EXPORT OrocosKDLTargets DESTINATION share/orocos_kdl/cmake)

INSTALL(FILES package.xml DESTINATION share/orocos_kdl)
