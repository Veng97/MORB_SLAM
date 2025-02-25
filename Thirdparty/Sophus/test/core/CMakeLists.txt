# Tests to run
SET( TEST_SOURCES test_common test_so2 test_se2 test_rxso2 test_sim2 test_so3 test_se3 test_rxso3 test_sim3 test_velocities test_geometry)
find_package( Ceres 1.6.0 QUIET )

# Set compiler standard to C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 7) # required for eigen
    message(FATAL_ERROR "Insufficient gcc version must be at least 7")
  endif()
endif()

FOREACH(test_src ${TEST_SOURCES})
  ADD_EXECUTABLE( ${test_src} ${test_src}.cpp tests.hpp ../../sophus/test_macros.hpp)
  TARGET_LINK_LIBRARIES( ${test_src} sophus )
  if( Ceres_FOUND )
      TARGET_INCLUDE_DIRECTORIES( ${test_src} SYSTEM PRIVATE ${CERES_INCLUDE_DIRS})
      ADD_DEFINITIONS(-DSOPHUS_CERES)
  endif(Ceres_FOUND)
  ADD_TEST( ${test_src} ${test_src} )
ENDFOREACH(test_src)


