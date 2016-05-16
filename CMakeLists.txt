# The MIT License (MIT)
# 
# Copyright (c) 2016 Fabian Killus
# 
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
# 
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
# 
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.2)


# ------------------------------------------------------------------------------------------------#
# The intention of this script is to provide a simple solution for building all dependencies
# required for Rigs of Rods. It is in fact a super- or meta-build project which makes use of
# the ExternalProject module of CMake to build the individual dependencies.
# ------------------------------------------------------------------------------------------------#

project( ror-dependencies VERSION 27 )


# ------------------------------------------------------------------------------------------------#
# Options to enable/disable building specific dependencies
# ------------------------------------------------------------------------------------------------#

# Options with same default value on all platforms
option( BUILD_OGRE         "Build Ogre3D graphics library"        ON )
option( BUILD_CAELUM       "Build Caelum library"                 ON )


# ------------------------------------------------------------------------------------------------#
# General setup and initialization
# ------------------------------------------------------------------------------------------------#

include(ExternalProject)

# Set the EP_BASE directory property to setup the build directory structure (see the
# ExternalProject documentation for more information)
set_property(DIRECTORY PROPERTY EP_BASE ${CMAKE_BINARY_DIR})


# Determine the name for the output directory where dependencies are going to be installed
if( WIN32 )
  set( DEPENDENCIES_OUTPUT_DIR ${CMAKE_BINARY_DIR}/Dependencies_${CMAKE_SYSTEM_NAME}_${CMAKE_GENERATOR} )
  string( REPLACE " " "-" DEPENDENCIES_OUTPUT_DIR ${DEPENDENCIES_OUTPUT_DIR} )
else()
  set( DEPENDENCIES_OUTPUT_DIR ${CMAKE_BINARY_DIR}/Dependencies_${CMAKE_SYSTEM_NAME} )
endif()

if( MSVC )
  set( DEPENDENCIES_INCLUDE_DIR ${DEPENDENCIES_OUTPUT_DIR}/include )
  set( DEPENDENCIES_LIB_DIR ${DEPENDENCIES_OUTPUT_DIR}/lib )
  set( DEPENDENCIES_BIN_DIR ${DEPENDENCIES_OUTPUT_DIR}/bin )
else()
  set( DEPENDENCIES_INCLUDE_DIR ${DEPENDENCIES_OUTPUT_DIR}/include )
  set( DEPENDENCIES_LIB_DIR ${DEPENDENCIES_OUTPUT_DIR}/lib/${CMAKE_BUILD_TYPE} )
  set( DEPENDENCIES_BIN_DIR ${DEPENDENCIES_OUTPUT_DIR}/bin/${CMAKE_BUILD_TYPE} )
endif()

file( MAKE_DIRECTORY ${DEPENDENCIES_INCLUDE_DIR} )
file( MAKE_DIRECTORY ${DEPENDENCIES_LIB_DIR} )
file( MAKE_DIRECTORY ${DEPENDENCIES_BIN_DIR} )


# Compile flags
if( MSVC )
  set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj" )
endif()

if( UNIX )
  #set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native" )
endif()


# On Linux some packages rely on pkgconfig to be found correctly. Therefore the environment variable
# needs to be set up accordingly. In addition when building depedencies which need to find other
# dependencies (e.g. MyGUI searching for OGRE) PKG_CONFIG_USE_CMAKE_PREFIX_PATH needs to be set to TRUE.
if( UNIX )
  set( ENV{PKG_CONFIG_PATH} "$ENV{PKG_CONFIG_PATH}:${DEPENDENCIES_LIB_DIR}/pkgconfig" )
endif()


# ------------------------------------------------------------------------------------------------#
#  Ogre3D
# ------------------------------------------------------------------------------------------------#
# Patching is needed for the D3D9 plugin when compiling with VS2015.
# See http://www.ogre3d.org/forums/viewtopic.php?f=2&t=84008

if( BUILD_OGRE )

  # First build the required dependencies for Ogre itself. They are provided in a dedicated repository.
  ExternalProject_Add(
    ogredeps
    URL https://bitbucket.org/cabalistic/ogredeps/get/7168c50f9d04.zip
    CMAKE_ARGS
      -DOGREDEPS_BUILD_SDL2=OFF
      -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/OgreDependencies
  )

  if( WIN32 )
    # By default the cg.dll is only installed in the build directory. Therefore manually copy it to the dependency output directory.
    ExternalProject_Add_Step(
      ogredeps copy_cg_dll
      COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/Build/ogredeps/bin/Release/cg.dll ${DEPENDENCIES_BIN_DIR}/cg.dll
      DEPENDEES build
      )
  endif()

  ExternalProject_Add(
    ogre
    DEPENDS ogredeps
    HG_REPOSITORY https://bitbucket.org/sinbad/ogre
    HG_TAG v1-8-0
    UPDATE_COMMAND ""
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
      -DOGRE_DEPENDENCIES_DIR=${CMAKE_BINARY_DIR}/OgreDependencies
      -DOGRE_USE_BOOST=OFF
      -DOGRE_BUILD_SAMPLES=OFF
      -DOGRE_BUILD_TOOLS=OFF
      -DOGRE_BUILD_RENDERSYSTEM_D3D9=ON
      -DOGRE_BUILD_RENDERSYSTEM_D3D11=OFF  # TODO
      -DOGRE_BUILD_RENDERSYSTEM_GL3PLUS=ON
      -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
  )

endif()



# ------------------------------------------------------------------------------------------------#
#  Caelum
# ------------------------------------------------------------------------------------------------#

if( BUILD_CAELUM )

  if( WIN32 )
    # Set paths to help find Ogre on Windows
    set( CAELUM_CMAKE_ARGS
	  -DOgre_INCLUDE_DIRS=${DEPENDENCIES_INCLUDE_DIR}/OGRE
	  -DOgre_LIBRARY_DIRS=${DEPENDENCIES_LIB_DIR}
	  -DOgre_LIBRARIES=${DEPENDENCIES_LIB_DIR}/OgreMain
    )
  elseif( UNIX )
    # Set pkgconfig path on Linux
    set( CAELUM_CMAKE_ARGS
        -DPKG_CONFIG_USE_CMAKE_PREFIX_PATH=ON
    )
  endif()

  ExternalProject_Add(
    caelum
    DEPENDS ogre
    GIT_REPOSITORY https://github.com/mikadou/ogre-caelum.git
    GIT_TAG boostless
    UPDATE_COMMAND ""
    CMAKE_ARGS
      -DPKG_CONFIG_USE_CMAKE_PREFIX_PATH=ON
      -DCMAKE_PREFIX_PATH=${DEPENDENCIES_OUTPUT_DIR}
      -DCMAKE_INSTALL_PREFIX=${DEPENDENCIES_OUTPUT_DIR}
      ${CAELUM_CMAKE_ARGS}
      -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    )
endif()

