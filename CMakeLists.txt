project(CMakeTemplate CXX)
cmake_minimum_required(VERSION 3.10)

set(CMAKE_VERBOSE_MAKEFILE OFF)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR}/cmake-modules)


################################################################
### Set options for compilation                              ###
################################################################
option(DISABLE_TESTING    "Disable  CTest testing"                                                          OFF)
option(BUILD_SHARED_LIBS  "Sets shared linking on/off"                                                      OFF)
option(ENABLE_EIGEN3      "Use Eigen3 Library. Guide with -DEIGEN_INCLUDE_DIR:PATH=<dir-to-eigen3>"         OFF)
option(ENABLE_SPDLOG      "Use SpdLog logging library. Guide with -Dspdlog_DIR:PATH=<dir-to-spdlog>"        OFF)
option(ENABLE_H5PP        "Use h5pp, a C++ front-end to HDF5"                                               OFF)
option(ENABLE_OPENMP      "Enable parallelization with OpenMP"                                              OFF)
option(DOWNLOAD_MISSING   "Download enabled libraries that aren't found"                                    OFF)


#################################################################
### h5pp includes eigen as a dependency                       ###
#################################################################
if(ENABLE_H5PP)
    message(STATUS "Eigen3 is included with h5pp -- disabling standalone Eigen3")
    set(ENABLE_EIGEN3 OFF)
    set(ENABLE_SPDLOG OFF)
endif()

#################################################################
### Set default policies if CMake is new enough               ###
#################################################################
if (CMAKE_VERSION VERSION_LESS 3.12)
    message(STATUS "Not setting policies")
else()
    cmake_policy(SET CMP0074 NEW)
    cmake_policy(SET CMP0075 NEW)
#    # Use cmake_policy CMP0067 to let try_compile honor current CMAKE_CXX_STANDARD setting.
#    # See here https://stackoverflow.com/questions/47213356/cmake-using-corrext-c-standard-when-checking-for-header-files
#    cmake_policy(SET CMP0067 NEW)
endif()


################################################################
### Get git version number                                   ###
### Generates a header gitversion/gitversion.h               ###
### Include it using #include <gitversion.h>                 ###
### Gives a namespace GIT:: with several git version numbers.###
################################################################
include(cmake-modules/gitversion.cmake)



##################################################################
### Print operating system details                          ###
##################################################################
include(cmake-modules/PrintHostInfo.cmake)

##################################################################
### Print compiler details                                     ###
##################################################################
message("Compiling with: ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")



##################################################################
### Create a target with all the options you need to compile   ###
### NOTE: This should not contain 3rd party libraries, only    ###
### features and flags such as -O3, -fopenmp and -std=c++17    ###
##################################################################
add_library(project-settings INTERFACE)

##################################################################
### Setup compiler flags                                       ###
##################################################################
if(NOT MARCH)
    set(MARCH "native")
endif()
message("Using microarchitechture: ${MARCH}")
set(COMMON_OPTIONS            -g -Wall -Wpedantic -Wextra  -march=${MARCH} -mtune=${MARCH})
set(DEBUG_OPTIONS             -O0 -g3 -fstack-protector -D_GLIBCXX_DEBUG -D_GLIBCXX_DEBUG_PEDANTIC -D_FORTIFY_SOURCE=2)
set(RELWITHDEBINFO_OPTIONS    -O1 -g3 -fstack-protector -D_GLIBCXX_DEBUG -D_GLIBCXX_DEBUG_PEDANTIC -D_FORTIFY_SOURCE=2)
set(RELEASE_OPTIONS           -O3 -DNDEBUG )
set(PROFILE_OPTIONS           -O3 -DNDEBUG  -lprofiler -g  -ftime-report)


##################################################################
### OS-dependent compiler flags                                ###
##################################################################
if(${CMAKE_HOST_APPLE})
    list(APPEND COMMON_OPTIONS  -mmacosx-version-min=10.13)
endif()


##################################################################
### Compiler-dependent compiler flags                          ###
##################################################################
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU" )
    list(APPEND PROFILE_OPTIONS -fopt-info -fopt-info-missed)
elseif("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    list(APPEND COMMON_OPTIONS ${GCC_TOOLCHAIN})
    list(APPEND COMMON_OPTIONS -stdlib=libstdc++)
    list(APPEND PROFILE_OPTIONS  -Rpass-missed="loop|vect" )# -Rpass="loop|vect" -Rpass-analysis="loop|vect")
endif()



##################################################################
### Setup static/shared build and library suffixes (.a/.so)    ###
##################################################################
set(COMPILER_LFLAGS)
if(BUILD_SHARED_LIBS)
    set(CUSTOM_SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX})
    set(CMAKE_FIND_LIBRARY_SUFFIXES ${CUSTOM_SUFFIX} ${CMAKE_FIND_LIBRARY_SUFFIXES})
else()
    set(CUSTOM_SUFFIX ${CMAKE_STATIC_LIBRARY_SUFFIX})
    set(CMAKE_FIND_LIBRARY_SUFFIXES ${CUSTOM_SUFFIX} ${CMAKE_FIND_LIBRARY_SUFFIXES})
    list(APPEND COMPILER_LFLAGS -static)        ### Static linkage
endif()


##################################################################
### Compiler-dependent linker flags                            ###
##################################################################
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU" )
    list(APPEND COMPILER_LFLAGS -lstdc++fs)     ### For experimental::filesystem
    list(APPEND COMPILER_LFLAGS -flto)          ### Link time optimization flag
elseif("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    list(APPEND COMPILER_LFLAGS -stdlib=libstdc++)
    list(APPEND COMPILER_LFLAGS -lstdc++fs)        ### For experimental::filesystem
endif()



##################################################################
### Enable c++17 support                                       ###
##################################################################
target_compile_features(project-settings INTERFACE cxx_std_17)
target_compile_options(project-settings INTERFACE -std=c++17)


######################################
###  Check  C++17 compiles         ###
######################################
include(cmake-modules/CheckCXX17Compiles.cmake)


##################################################################
### Adapt pthread for static/dynamic linking                   ###
##################################################################
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
set(THREADS_PREFER_PTHREAD_FLAG FALSE)
find_package(Threads)
if(TARGET Threads::Threads AND NOT BUILD_SHARED_LIBS)
    set_target_properties(Threads::Threads PROPERTIES INTERFACE_LINK_LIBRARIES "-Wl,--whole-archive ${CMAKE_THREAD_LIBS_INIT} -Wl,--no-whole-archive")
endif()



##################################################################
###  Optional OpenMP support                                   ###
###  Note that Clang has some trouble with static openmp       ###
###  We can solve this by trying to use its omp.h header       ###
###  linked together with intel omp library, libiomp5.a        ###
##################################################################
if(ENABLE_OPENMP)
    include(cmake-modules/CheckOMPCompiles.cmake)
    if(TARGET OpenMP)
        message(STATUS "Enabling OpenMP")
        target_link_libraries(project-settings INTERFACE OpenMP)
    else()
        message(STATUS "Disabling OpenMP -- Could not generate target OpenMP")
        set(ENABLE_OPENMP OFF)
        list(APPEND COMMON_OPTIONS -Wno-unknown-pragmas)
    endif()
endif()





##################################################################
### Apply all settings to helper target                        ###
##################################################################
target_compile_options (project-settings INTERFACE
        ${COMMON_OPTIONS}
        "$<$<CONFIG:DEBUG>:${DEBUG_OPTIONS}>"
        "$<$<CONFIG:RELWITHDEBINFO>:${RELWITHDEBINFO_OPTIONS}>"
        "$<$<CONFIG:RELEASE>:${RELEASE_OPTIONS}>"
        "$<$<CONFIG:PROFILE>:${PROFILE_OPTIONS}>"
        )
target_link_libraries(project-settings INTERFACE ${COMPILER_LFLAGS})
target_include_directories(project-settings INTERFACE source)




##################################################################
### Print all flags                                            ###
##################################################################
    message(STATUS "COMMON CXX flags    : ${COMMON_OPTIONS} ${CMAKE_CXX_FLAGS}")
if (CMAKE_BUILD_TYPE STREQUAL "Release")
    message(STATUS "RELEASE CXX flags   : ${RELEASE_OPTIONS}")
elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    message(STATUS "RELWITHDEBINFO  CXX flags: ${RELWITHDEBINFO_OPTIONS}")
elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
    message(STATUS "DEBUG CXX flags     : ${DEBUG_OPTIONS}")
elseif(CMAKE_BUILD_TYPE STREQUAL "Profile")
    message(STATUS "PROFILE CXX flags   : ${PROFILE_OPTIONS}")
endif()
    message(STATUS "LINKER CXX flags    : ${COMPILER_LFLAGS}")

get_target_property(SETTINGS_INFO_LIB   project-settings      INTERFACE_LINK_LIBRARIES)
get_target_property(SETTINGS_INFO_INC   project-settings      INTERFACE_INCLUDE_DIRECTORIES)
get_target_property(SETTINGS_INFO_OPT   project-settings      INTERFACE_COMPILE_OPTIONS)
message(STATUS "Settings -- linker  : ${SETTINGS_INFO_LIB}")
message(STATUS "Settings -- include : ${SETTINGS_INFO_INC}")
message(STATUS "Settings -- flags   : ${SETTINGS_INFO_OPT}")





##################################################################
### Add all source files recursively                           ###
##################################################################
file(GLOB_RECURSE SOURCES "source/*.cpp")
file(GLOB_RECURSE HEADERS "source/*.h")
add_executable(${PROJECT_NAME} ${SOURCES} ${HEADERS})


##################################################################
### Apply all settings to current project                      ###
##################################################################
target_link_libraries(${PROJECT_NAME} PUBLIC project-settings)



##################################################################
### Setup dependency paths                                     ###
##################################################################
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "${PROJECT_SOURCE_DIR}/libs" CACHE PATH "..." FORCE)
endif()

include(GNUInstallDirs)
if(THIRD_PARTY_DIR)
    set(INSTALL_DIR_THIRD_PARTY       ${THIRD_PARTY_DIR})
else()
    set(INSTALL_DIR_THIRD_PARTY       ${CMAKE_INSTALL_PREFIX})
endif()
set(BUILD_DIR_THIRD_PARTY         ${CMAKE_BINARY_DIR}/third-party-build)
set(CONFIG_DIR_THIRD_PARTY        ${CMAKE_BINARY_DIR}/third-party-config)

if (ENABLE_SPDLOG)
    include(cmake-modules/Fetch_spdlog.cmake)
endif()

if (ENABLE_H5PP)
    include(cmake-modules/Fetch_h5pp.cmake)
endif()

if (ENABLE_EIGEN3)
    include(cmake-modules/Fetch_Eigen3.cmake)
endif()



##################################################################
### Link all generated targets for dependencies                ###
##################################################################
target_link_libraries(${PROJECT_NAME} PRIVATE ${COMPILER_LFLAGS})
set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD_REQUIRED ON CXX_EXTENSIONS OFF)

if (TARGET spdlog::spdlog AND ENABLE_SPDLOG)
    target_link_libraries(${PROJECT_NAME} PRIVATE spdlog::spdlog)
endif()

if (TARGET h5pp::h5pp AND TARGET h5pp::deps AND ENABLE_H5PP)
    target_link_libraries(${PROJECT_NAME} PRIVATE h5pp::h5pp h5pp::deps)
endif()


##################################################################
### Print all generated targets for dependencies               ###
##################################################################
if(TARGET OpenMP AND ENABLE_OPENMP)
    get_target_property(OMP_INFO_LIB   OpenMP            INTERFACE_LINK_LIBRARIES)
    get_target_property(OMP_INFO_INC   OpenMP            INTERFACE_INCLUDE_DIRECTORIES)
endif()

if(TARGET spdlog::spdlog AND ENABLE_SPDLOG)
    get_target_property(SPDLOG_INFO         spdlog::spdlog   INTERFACE_INCLUDE_DIRECTORIES)
endif()

if(TARGET h5pp::h5pp AND TARGET h5pp::deps AND ENABLE_H5PP)
    get_target_property(H5PP_INFO_INC         h5pp::h5pp   INTERFACE_INCLUDE_DIRECTORIES)
    get_target_property(H5PP_DEPS_LIB         h5pp::deps   INTERFACE_LINK_LIBRARIES)
endif()

if(TARGET Eigen3::Eigen AND ENABLE_EIGEN3)
    get_target_property(EIGEN3_INCLUDE_DIR    Eigen3::Eigen   INTERFACE_INCLUDE_DIRECTORIES)
endif()


##################################################################
### Print summary of CMake configuration                       ###
##################################################################
message("=============================================================================")
message("=== SUMMARY OF CMAKE CONFIGURATION:  THE FOLLOWING LIBRARIES WILL BE USED ===")
if(ENABLE_SPDLOG)
message("=============================================================================")
message("    SPDLOG         INC:  ${SPDLOG_INFO}")
endif()
if(ENABLE_H5PP)
message("=============================================================================")
message("    H5PP           INC:  ${H5PP_INFO_INC}")
message("                   DEP:  ${H5PP_DEPS_LIB}")
endif()
if(ENABLE_EIGEN3)
message("=============================================================================")
message("    EIGEN3         INC:  ${EIGEN3_INCLUDE_DIR}")
endif()
message("=============================================================================")
message("")


##################################################################
### Print general information about next steps                 ###
##################################################################
message("")
message("=============================================================================")
message("")
message("      To clean cmake files, run './build.sh -c'")
message("      To clean all downloaded libraries, run './build.sh -L '")
message("      To clean specific libraries, run './build.sh -l <libname> '")
message("      For more info, run ./build.sh -h '")
message("")
message("=============================================================================")
message("")




##################################################################
### Unit testing runs before the main project builds           ###
##################################################################
if(NOT DISABLE_TESTING)
    enable_testing()

    # Set up a target "all-tests" which will contain all the tests.
    add_custom_target(all-tests)
    # Add the all the tests to all-tests
    if(ENABLE_H5PP)
        add_subdirectory(tests/h5pp EXCLUDE_FROM_ALL)
        add_dependencies(all-tests test-h5pp)  #Make sure the test gets built after main project
    endif()
    if(ENABLE_SPDLOG OR ENABLE_H5PP)
        add_subdirectory(tests/spdlog EXCLUDE_FROM_ALL)
        add_dependencies(all-tests test-spdlog)  #Make sure the test gets built after main project
    endif()
    if(ENABLE_EIGEN3 OR ENABLE_H5PP)
        add_subdirectory(tests/eigen3 EXCLUDE_FROM_ALL)
        add_dependencies(all-tests test-eigen3)  #Make sure the test gets built after main project
    endif()
    if(ENABLE_OPENMP)
        add_subdirectory(tests/openmp EXCLUDE_FROM_ALL)
        add_dependencies(all-tests test-openmp)  #Make sure the test gets built after main project
    endif()
    #Run all tests as soon as the tests have been built
    add_custom_command(TARGET all-tests
            POST_BUILD
            COMMENT "Running Tests"
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
            COMMAND ${CMAKE_CTEST_COMMAND} -C $<CONFIGURATION> --output-on-failures)

    # Make sure the tests are built and run before building the main project
    add_dependencies(${PROJECT_NAME} all-tests)
endif()





