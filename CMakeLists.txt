cmake_minimum_required(VERSION 3.16.0)

set(MORB_VERSION_MAJOR 0 CACHE STRING "major version" FORCE) 
set(MORB_VERSION_MINOR 0 CACHE STRING "major version" FORCE) 
set(MORB_VERSION_PATCH 1 CACHE STRING "major version" FORCE) 
set(MORB_VERSION_SHORT ${MORB_VERSION_MAJOR}.${MORB_VERSION_MINOR} CACHE STRING "short-version" FORCE)
set(MORB_VERSION ${MORB_VERSION_SHORT}.${MORB_VERSION_PATCH} CACHE STRING "version" FORCE)

find_program(CLANGXX_17_FOUND clang++-17)
if(CLANGXX_17_FOUND)
    set(CMAKE_C_COMPILER "clang-17")
    set(CMAKE_CXX_COMPILER "clang++-17")
endif()

# If we have used vcpkg in the past, use it again
if(EXISTS "${CMAKE_SOURCE_DIR}/vcpkgLoc.txt" AND NOT CMAKE_TOOLCHAIN_FILE)
    file(READ "${CMAKE_SOURCE_DIR}/vcpkgLoc.txt" CMAKE_TOOLCHAIN_FILE)
endif()

if( CMAKE_TOOLCHAIN_FILE )
    # If we want to use vcpkg
    if(EXISTS "${CMAKE_TOOLCHAIN_FILE}")
        file(WRITE "${CMAKE_SOURCE_DIR}/vcpkgLoc.txt" ${CMAKE_TOOLCHAIN_FILE})
        message(STATUS "Using vcpkg toolchain")
        set(Boost_NO_SYSTEM_PATHS ON)
        set(Boost_NO_WARN_NEW_VERSIONS ON)
        set(VCPKG_MANIFEST_FEATURES "examples")

        # set(Python_ROOT_DIR "${CMAKE_SOURCE_DIR}/venv")
        # set(PYTHON_EXECUTABLE "${CMAKE_SOURCE_DIR}/venv/bin/python3")
        # set(Python_FIND_VIRTUALENV ONLY)
        # find_package(Python COMPONENTS Interpreter Development)
    else()
        message(FATAL_ERROR "The cmake toolchain file specified does not exist: ${CMAKE_TOOLCHAIN_FILE}")
    endif()
else()
    message(STATUS "No vcpkg toolchain file specified")
endif()

project(MORB_SLAM 
    VERSION ${MORB_VERSION}
    DESCRIPTION "A library for visual simultanious localization and mapping (vSLAM) for autonomous vehicles"
    HOMEPAGE_URL "https://github.com/DavidPetkovsek/MORB_SLAM"
    LANGUAGES C CXX # if you do not specify C in addition to CXX, GNU compilers will fail to find threads on some computers
)


include(GNUInstallDirs)

# Compiler configuration
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Default to Release if not explicitly set
if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "No build type specified; defaulting to RelWithDebInfo")
    set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Build type" FORCE)
endif()
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -gdwarf-4")
elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -gdwarf-4")
endif()

# https://eigen.tuxfamily.org/dox/group__TopicStructHavingEigenMembers.html
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if(CMAKE_GENERATOR STREQUAL "Ninja")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdiagnostics-color=always")
    endif()
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 7) # required for eigen
        message(FATAL_ERROR "Insufficient g++ version must be at least 7, required for eigen")
    endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    if(CMAKE_GENERATOR STREQUAL "Ninja")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdiagnostics-color=always")
    endif()
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5) # required for eigen
        message(FATAL_ERROR "Insufficient clang++ version must be at least 5, required for eigen")
    endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.12) # required for eigen
        message(FATAL_ERROR "Insufficient msvc version must be at least 19.12, required for eigen")
    endif()
endif()


set(CMAKE_PROJECT_CONFIG_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
set(THREADS_PREFER_PTHREAD_FLAG ON)

include("${PROJECT_SOURCE_DIR}/cmake/FuncLib.cmake")


######################################################################
#                                                                    #
#           Subdirectories, Components, and targets                  #
#                                                                    #
######################################################################


add_subdirectory(Thirdparty/g2o)
add_subdirectory(Thirdparty/DBoW2)
add_subdirectory(Thirdparty/Sophus)

set(TARGET_NAME ${PROJECT_NAME})

find_package(Threads REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(OpenCV REQUIRED)
find_package(Boost REQUIRED COMPONENTS serialization regex)
find_package(Boost REQUIRED)
find_package(OpenSSL REQUIRED) # TODO: Do we need this dependency?
find_package(Pangolin QUIET) # only needed for visualization
find_package(ixwebsocket QUIET) # only needed for getting realsense data through a websocket
find_package(realsense2 QUIET) # realsense only needed for example

file(GLOB CPP_FILES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cc)
file(GLOB CPP_OPTIMIZER_FILES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/Optimizer/*.cpp ${CMAKE_CURRENT_SOURCE_DIR}/src/Optimizer/*.cc)
file(GLOB CPP_CAMERA_MODELS_FILES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/CameraModels/*.cpp)
file(GLOB CPP_SETTINGS_FILES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/Settings/*.cpp)
file(GLOB CPP_INERTIAL_ODOMETRY_FILES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/InertialOdometry/*.cc ${CMAKE_CURRENT_SOURCE_DIR}/src/InertialOdometry/*.cpp ${CMAKE_CURRENT_SOURCE_DIR}/src/InertialOdometry/InertialOptimizer/*.cc)
list(APPEND CPP_FILES ${CPP_CAMERA_MODELS_FILES})
list(APPEND CPP_FILES ${CPP_OPTIMIZER_FILES})
list(APPEND CPP_FILES ${CPP_SETTINGS_FILES})
list(APPEND CPP_FILES ${CPP_INERTIAL_ODOMETRY_FILES})


file(GLOB headers CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/*.hpp ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/*.h)
file(GLOB camera_model_headers CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/CameraModels/*.h ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/CameraModels/*.hpp)
file(GLOB settings_headers CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/Settings/*.h ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/Settings/*.hpp)
file(GLOB inertial_odometry_headers CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/InertialOdometry/*.h ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/InertialOdometry/*.hpp)


# optionally built native viewer
set(NATIVE_VIEWER_BUILT OFF)
set(NATIVE_VIEWER_LIBS "")
set(NATIVE_VIEWER_INCLUDES "")
if(Pangolin_FOUND)
    message(STATUS "** Pangolin found, native viewer will be built.")
    
    set(NATIVE_VIEWER_BUILT ON)

    file(GLOB CPP_PANGOLIN_VIEWER_FILES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/Viewers/Native/*.cc)
    list(APPEND CPP_FILES ${CPP_PANGOLIN_VIEWER_FILES})
    set(NATIVE_VIEWER_LIBS ${Pangolin_LIBRARIES})
    file(GLOB pangolin_viewer_headers CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/Viewers/Native/*.h)
    set(NATIVE_VIEWER_INCLUDES ${pangolin_viewer_headers})
else()
    message(STATUS "** Pangolin not found, the native viewer will not be built.")
endif()

# optionally built external viewer
set(EXTERNAL_VIEWER_BUILT OFF)
set(EXTERNAL_VIEWER_LIBS "")
set(EXTERNAL_VIEWER_INCLUDES "")
if(ixwebsocket_FOUND)
    message(STATUS "** ixwebsocket found, external viewer will be built.")

    set(EXTERNAL_VIEWER_BUILT ON)

    file(GLOB CPP_EXTERNAL_VIEWER_FILES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/Viewers/External/ExternalMapViewer.cc)
    list(APPEND CPP_FILES ${CPP_EXTERNAL_VIEWER_FILES})
    set(EXTERNAL_VIEWER_LIBS ixwebsocket::ixwebsocket)
    file(GLOB external_viewer_headers CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/Viewers/External/ExternalMapViewer.h)
    set(EXTERNAL_VIEWER_INCLUDES ${external_viewer_headers})
else()
    message(STATUS "** ixwebsocket not found, the external viewer will not be built.")
endif()


add_library(${TARGET_NAME} SHARED ${CPP_FILES})
add_library(${PROJECT_NAME}::${TARGET_NAME} ALIAS ${TARGET_NAME})
target_include_directories(${TARGET_NAME} PUBLIC
                       $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> # for headers when building
                       $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}> # for client in install mode
                       )
target_link_libraries( ${TARGET_NAME} PUBLIC ${PROJECT_NAME}::DBoW2 )
target_link_libraries( ${TARGET_NAME} PUBLIC ${PROJECT_NAME}::g2o )
target_link_libraries( ${TARGET_NAME} PUBLIC ${PROJECT_NAME}::sophus )
target_link_libraries( ${TARGET_NAME} PUBLIC Threads::Threads )
target_link_libraries( ${TARGET_NAME} PUBLIC Eigen3::Eigen )
target_link_libraries( ${TARGET_NAME} PUBLIC Boost::serialization Boost::regex )
target_link_libraries( ${TARGET_NAME} PUBLIC ${OpenCV_LIBS} )
target_link_libraries( ${TARGET_NAME} PRIVATE OpenSSL::SSL )
if(NATIVE_VIEWER_LIBS)
    target_link_libraries( ${TARGET_NAME} PRIVATE ${NATIVE_VIEWER_LIBS} )
endif()
if(EXTERNAL_VIEWER_LIBS)
    target_link_libraries( ${TARGET_NAME} PRIVATE ${EXTERNAL_VIEWER_LIBS} )
endif()


target_precompile_headers(${TARGET_NAME} PRIVATE
    <Eigen/Core>
    <Eigen/Dense>
    <Eigen/Geometry>
    <opencv2/core/core.hpp>
)


# https://cmake.org/cmake/help/latest/guide/importing-exporting/index.html#creating-a-package-version-file
# https://cmake.org/cmake/help/latest/manual/cmake-packages.7.html#creating-packages
# https://mail.kde.org/pipermail/kde-buildsystem/2008-April/004544.html
set_property(TARGET ${TARGET_NAME} PROPERTY VERSION ${MORB_VERSION}) # x.x.x used for symlinks
set_property(TARGET ${TARGET_NAME} PROPERTY SOVERSION ${MORB_VERSION_MAJOR}) # ABI/API compatibility at so level must be integer
set_property(TARGET ${TARGET_NAME} PROPERTY "INTERFACE_${TARGET_NAME}_MAJOR_VERSION" ${MORB_VERSION_MAJOR})
set_property(TARGET ${TARGET_NAME} APPEND PROPERTY COMPATIBLE_INTERFACE_STRING "${TARGET_NAME}_MAJOR_VERSION")


set(EXPORT_TARGET_NAME "${TARGET_NAME}Targets")
set(EXPORT_TARGET_FILE_NAME "${PROJECT_NAME}${EXPORT_TARGET_NAME}.cmake")


# install the target and create export-set
install(TARGETS ${TARGET_NAME}
        EXPORT ${EXPORT_TARGET_NAME}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# install header files
install(FILES ${headers} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME})
install(FILES ${camera_model_headers} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}/CameraModels)
install(FILES ${settings_headers} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}/Settings)
install(FILES ${inertial_odometry_headers} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}/InertialOdometry)
if(NATIVE_VIEWER_INCLUDES)
    install(FILES ${NATIVE_VIEWER_INCLUDES} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}/Viewer/Native)
endif()
if(EXTERNAL_VIEWER_INCLUDES)
    install(FILES ${EXTERNAL_VIEWER_INCLUDES} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}/Viewer/External)
endif()


# generate and install export file
install(EXPORT ${EXPORT_TARGET_NAME}
        FILE ${EXPORT_TARGET_FILE_NAME}
        NAMESPACE ${PROJECT_NAME}::
        DESTINATION ${CMAKE_PROJECT_CONFIG_INSTALL_DIR}
)


######################################################################
#                                                                    #
#                           EXAMPLES                                 #
#                                                                    #
######################################################################

# example stereo inertial realsense through websocket
if(ixwebsocket_FOUND AND realsense2_FOUND)
    set(TARGET_NAME realsense_websocket_client)
    add_executable(${TARGET_NAME} Examples/Stereo-Inertial/realsense_websocket_client.cpp)
    target_link_libraries( ${TARGET_NAME} PUBLIC ${PROJECT_NAME}::${PROJECT_NAME} ${realsense2_LIBRARY} )
else()
    message(STATUS "** Realsense websocket client will not compile because ixwebsocket was not found")
endif()

# example stereo inertial euroc
if(NATIVE_VIEWER_BUILT AND EXTERNAL_VIEWER_BUILT)
    add_executable(test_stereo_inertial_euroc Testing/Stereo-Inertial/EuRoC/test_stereo_inertial_euroc.cpp)
    target_link_libraries( test_stereo_inertial_euroc PUBLIC ${PROJECT_NAME}::${PROJECT_NAME} )
endif()

# example stereo inertial recordings
if(NATIVE_VIEWER_BUILT)    
    add_executable(test_stereo_inertial_recordings Testing/Stereo-Inertial/Recordings/test_stereo_inertial_recordings.cpp)
    target_link_libraries( test_stereo_inertial_recordings PUBLIC ${PROJECT_NAME}::${PROJECT_NAME} )
endif()


######################################################################
#                                                                    #
#                CMAKE INSTALL, CONFIG, AND PACKAGING                #
#                                                                    #
######################################################################


set(CONFIG_FILE_NAME_WITHOUT_EXT "${PROJECT_NAME}Config")
set(CMAKE_CONFIG_FILE_BASENAME "${CMAKE_CURRENT_BINARY_DIR}/${CONFIG_FILE_NAME_WITHOUT_EXT}")
set(CMAKE_CONFIG_VERSION_FILE_NAME "${CMAKE_CONFIG_FILE_BASENAME}-version.cmake")
set(CMAKE_CONFIG_FILE_NAME "${CMAKE_CONFIG_FILE_BASENAME}.cmake")

include(CMakePackageConfigHelpers)


# generate the version file for the config file
write_basic_package_version_file(
  "${CMAKE_CONFIG_VERSION_FILE_NAME}"
  COMPATIBILITY SameMajorVersion
)

# create config file
configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/Config.cmake.in
  "${CMAKE_CONFIG_FILE_NAME}"
  INSTALL_DESTINATION ${CMAKE_PROJECT_CONFIG_INSTALL_DIR}
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

# install config files
install(FILES
        "${CMAKE_CONFIG_FILE_NAME}"
        "${CMAKE_CONFIG_VERSION_FILE_NAME}"
        DESTINATION ${CMAKE_PROJECT_CONFIG_INSTALL_DIR}
)

include("${PROJECT_SOURCE_DIR}/cmake/Packing.cmake")
