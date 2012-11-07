# Defines the CMake commands/policies
cmake_minimum_required( VERSION 2.8.5 )

# Set the project name
project( pyne )

# Make the scripts available in the 'cmake' directory available for the
# 'include()' command, 'find_package()' command.
set( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_LIST_DIR}/cmake )

# Include the HDF5 library and c++ headers
find_package( HDF5 COMPONENTS C )
include_directories(${HDF5_INCLUDE_DIRS})
if(WIN32)
    # FindHDF5 finds the includes but not the libraries on Windows (MSYS).  Annoying!
    get_filename_component(_hdf5libdir ${HDF5_INCLUDE_DIRS} PATH)
    list(APPEND HDF5_LIBRARY_DIRS "${_hdf5libdir}/bin")
endif(WIN32)
link_directories(${HDF5_LIBRARY_DIRS})
add_definitions(${HDF5_DEFINITIONS})
set(LIBS ${LIBS} ${HDF5_C_LIBRARIES})

# Include the CMake script UseCython.cmake.  This defines add_cython_module().
# Instruction for use can be found at the top of cmake/UseCython.cmake.
include( UseCython )

# Also, there are some custom pyne macros to make for less verbosity
include( PyneMacros )

# Beware of dragons
print_logo()

# This makes all the libraries build as SHARED
set(BUILD_SHARED_LIBS true)

# find numpy and include the numpy headers
find_package(Numpy REQUIRED)
include_directories(${NUMPY_INCLUDE_DIR})

# Add JsonCpp Flag
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DJSON_IS_AMALGAMATION")

# With CMake, a clean separation can be made between the source tree and the
# build tree.  When all source is compiled, as with pure C/C++, the source is
# no-longer needed in the build tree.  However, with pure *.py source, the
# source is processed directly.  To handle this, we reproduce the availability
# of the source files in the build tree.
add_custom_target( ReplicatePythonSourceTree ALL ${CMAKE_COMMAND} -P
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/ReplicatePythonSourceTree.cmake
  ${CMAKE_CURRENT_BINARY_DIR}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} )

# Add cython version info to source tree
message("-- Making Cython Version.")
execute_process(COMMAND python ${CMAKE_CURRENT_SOURCE_DIR}/configure.py cython_version)
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/pyne/_includes/
     DESTINATION ${CMAKE_BINARY_DIR}/pyne/_includes/)

add_subdirectory(cpp)
add_subdirectory(pyne)
add_subdirectory(pyne/xs)

# Print include dir
get_property(inc_dirs DIRECTORY PROPERTY INCLUDE_DIRECTORIES)
message("-- C_INCLUDE_PATH for ${CMAKE_CURRENT_SOURCE_DIR}: ${inc_dirs}")

message("-- Copying C/C++ header files.")
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/cpp/ DESTINATION
    ${CMAKE_BINARY_DIR}/pyne/include/ FILES_MATCHING PATTERN "*.h")

message("-- Copying Cython header files.")
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/pyne/ 
    DESTINATION ${CMAKE_BINARY_DIR}/pyne/include/pyne/ 
    FILES_MATCHING PATTERN "*.pxd"
                   PATTERN "lib" EXCLUDE
                   PATTERN "include" EXCLUDE)

message("-- Copying scripts.")
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/scripts DESTINATION ${CMAKE_BINARY_DIR})

# Add cython version info to source tree
message("-- Making Metadata.")
execute_process(COMMAND python ${CMAKE_CURRENT_SOURCE_DIR}/configure.py metadata)
