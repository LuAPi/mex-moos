cmake_minimum_required(VERSION 3.3.2)
project(mex-moos)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

set(CMakeScripts_MODULE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
set(CMAKE_MODULE_PATH ${CMakeScripts_MODULE_DIR})

find_package(MOOS 10 REQUIRED)
find_package( Matlab REQUIRED MEX_COMPILER )

set(LIBRARY_FULL_NAME mex-moos)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})

# Declare the mex target name and associated source file.
set(${LIBRARY_FULL_NAME}_MEX_TARGET mexmoos)
set(${LIBRARY_FULL_NAME}_MEX_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/mex-moos.cpp)

# Explicitly set the output location for the mex file to be build/lib, and set
# the extension to match the mex extension for the current platform.
set(${LIBRARY_FULL_NAME}_MEX_FILE
  ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/${${LIBRARY_FULL_NAME}_MEX_TARGET}.${Matlab_MEX_EXTENSION})

# Add custom command to build the mex file.
#
# Build the -I<DIR> include arguments for mex.  We do _not_ include the MRG
# library dependency includes here, as they are explicitly not in the public
# API and thus not required by mex.
set(MEX_API_PUBLIC_INCLUDE_DIRS
  ${MOOS_INCLUDE_DIRS}
  ${${LIBRARY_FULL_NAME}_EXTERNAL_DEPEND_INCLUDE_DIRS}
  ${${LIBRARY_FULL_NAME}_SELF_INCLUDE_DIRS})
list(REMOVE_DUPLICATES MEX_API_PUBLIC_INCLUDE_DIRS)
foreach (INC_DIR ${MEX_API_PUBLIC_INCLUDE_DIRS})
  list(APPEND MEX_BUILD_INCLUDE_DIRS_CMDS "-I${INC_DIR}")
endforeach()

if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
  get_filename_component( MATLAB_MEX_LIBRARIES_DIR "${Matlab_MEX_LIBRARY}" DIRECTORY )
  # On Linux, the mex compiler does not properly use rpath linking (up to
  # at least 2013b), thus although the mex file will link, it will not save
  # full paths to the required libraries, thus otool -L / ldd with show
  # 'libmx -> not found' or similar.  Thus we have to explicitly set the
  # -rpath list to include BOTH the MATLAB library directory, AND the directory
  # containing our libraries (passed to -L as well).  We have to do this for
  # MATLAB as well as MATLAB **INCORRECTLY** sets the rpath (WTF?).
  #
  # Note that due to an incomprehensible reason, -rpath fails if you have
  # spaces between it and the previous / next argument, see [1].
  #
  # [1]: http://www-01.ibm.com/support/docview.wss?uid=swg21329145
  #
  # The -Wl is important for MATLAB R2014+.
  set(FORCE_AND_FIX_MEX_TO_USE_RPATH_LINKING_ARGS
    LDFLAGS='$$LDFLAGS -Wl,-rpath=${MATLAB_MEX_LIBRARIES_DIR},-rpath=${CMAKE_LIBRARY_OUTPUT_DIRECTORY},-rpath=${MOOS_DIR}/lib')
endif()


# IMPORTANT: DO **NOT** add VERBATIM to the end of this add_custom_command(),
#            it screws up the odd MEX_LD_FLAGS escaping (adds wrapping "" and
#            does crazy things with the $LDFLAGS statement).

# Strip namespace prefixes from library binary names
string(REGEX REPLACE ".+::"
       "" MOOS_LIB_BINARY_NAME
       ${MOOS_LIBRARIES} )

if (POLICY CMP0042)
  cmake_policy(GET CMP0042 OSX_USING_PATH)
endif()

# We only need to _explicitly_ link against libraries that are used _directly_
# in the MEX file.  As such we do not need to explicitly link against glog or
# Ceres as they come in through the linking of -l${LIBRARY_FULL_NAME}.
#
# Note that we explicitly disable ensures in Sophus library when compiling mex
# as the default ensure uses printf which is a macro for mexPrintf in MEX files
# which does not have the same arguments.

if (APPLE AND OSX_USING_PATH)
  # On OS X on newer versions of CMake when CMP0042 (rpath) is enabled by
  # default, mex fails to correctly set the link library paths and so the
  # libraries are not set.  Here we use the install_name_tool app to set
  # the rpaths explicitly s/t the direct dependencies are picked up.
  find_program(INSTALL_NAME_TOOL install_name_tool)
  mark_as_advanced(FORCE INSTALL_NAME_TOOL)
  add_custom_command( OUTPUT ${${LIBRARY_FULL_NAME}_MEX_FILE}
    COMMAND ${Matlab_MEX_COMPILER}
    ARGS -DSOPHUS_DISABLE_ENSURES
    ${MEX_BUILD_INCLUDE_DIRS_CMDS}
    ${FORCE_AND_FIX_MEX_TO_USE_RPATH_LINKING_ARGS}
    -L${MOOS_DIR}/lib
    -l${MOOS_LIB_BINARY_NAME}
    -output ${${LIBRARY_FULL_NAME}_MEX_FILE}
    ${${LIBRARY_FULL_NAME}_MEX_SOURCE}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${${LIBRARY_FULL_NAME}_MEX_SOURCE}
    COMMAND ${INSTALL_NAME_TOOL} 
    ARGS -add_rpath ${MOOS_DIR}/lib
    ${${LIBRARY_FULL_NAME}_MEX_FILE}
    DEPENDS ${${LIBRARY_FULL_NAME}_MEX_SOURCE}
    COMMENT "Building MEX interface for ${LIBRARY_FULL_NAME}.")
else()
  add_custom_command( OUTPUT ${${LIBRARY_FULL_NAME}_MEX_FILE}
    COMMAND ${Matlab_MEX_COMPILER}
    ARGS -DSOPHUS_DISABLE_ENSURES
    ${MEX_BUILD_INCLUDE_DIRS_CMDS}
    ${FORCE_AND_FIX_MEX_TO_USE_RPATH_LINKING_ARGS}
    -L${MOOS_DIR}/lib
    -l${MOOS_LIB_BINARY_NAME}
    -output ${${LIBRARY_FULL_NAME}_MEX_FILE}
    ${${LIBRARY_FULL_NAME}_MEX_SOURCE}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${${LIBRARY_FULL_NAME}_MEX_SOURCE}
    COMMENT "Building MEX interface for ${LIBRARY_FULL_NAME}.")
endif()

# Add custom target which is always built, that depends on the mex file to drive
# it's creation.
add_custom_target( ${${LIBRARY_FULL_NAME}_MEX_TARGET}
  ALL DEPENDS ${${LIBRARY_FULL_NAME}_MEX_FILE}
  SOURCES ${${LIBRARY_FULL_NAME}_MEX_SOURCE}
  VERBATIM )

message(STATUS "==> MRG: IMPORTANT: to use mex-moos with matlab, add the "
  "following to MATLAB's path:")
message(STATUS "==> MRG:   -- ${CMAKE_CURRENT_SOURCE_DIR}")
message(STATUS "==> MRG:   -- ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")
