# --------------------------------------------------------------------------- #
#    Main CMake file for SMS++ BundleSolver                                   #
#                                                                             #
#    This file allows one to build the library using CMake.                   #
#    To do so, you can use the following commands:                            #
#                                                                             #
#        $ cmake -S <source-path> -B <build-path>                             #
#        $ cmake --build <build-path>                                         #
#                                                                             #
#    The following command also installs the library in the system:           #
#                                                                             #
#        $ cmake --build <build-path> --target install                        #
#                                                                             #
#    Note: this file tries to respect the principles of Modern CMake,         #
#          keep it in mind when editing it.                                   #
#                                                                             #
#                              Niccolo' Iardella                              #
#                                Donato Meoli                                 #
#                         Dipartimento di Informatica                         #
#                             Universita' di Pisa                             #
# --------------------------------------------------------------------------- #
cmake_minimum_required(VERSION 3.19)
cmake_policy(VERSION 3.12)
project(BundleSolver
        VERSION 0.4.3
        DESCRIPTION "SMS++ BundleSolver module"
        HOMEPAGE_URL https://gitlab.com/smspp/bundlesolver
        LANGUAGES C CXX)

# These variables make the code harder to read but easier to change.
set(modName ${PROJECT_NAME})
set(modNamespace "SMS++")

# Find out if it's being called by the umbrella.
get_directory_property(hasParent PARENT_DIRECTORY)

# This adds the cmake directory to the module search paths,
# allowing us to use our modules.
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# ----- Settings ------------------------------------------------------------ #
# This creates an ENABLE_TESTING option (default: ON) and enables the testing
# using the ctest executable.
# See: https://cmake.org/cmake/help/latest/manual/ctest.1.html
#include(CTest)

# Sets the default build type (if none was specified).
# See: https://cmake.org/cmake/help/latest/variable/CMAKE_BUILD_TYPE.html
include(BuildType)

# ----- Requirements -------------------------------------------------------- #
# OsiCpx and/or OsiGrb needed for BundleSolver
find_package(Osi OPTIONAL_COMPONENTS OsiCpx OsiGrb)
if ((NOT Osi_OsiCpx_FOUND) AND (NOT Osi_OsiGrb_FOUND))
    set(NDOFiOracle_USE_OSI OFF)
endif ()

# OsiClp needed for BundleSolver
find_package(Clp COMPONENTS OsiClp)
if (NOT Clp_FOUND)
    set(NDOFiOracle_USE_CLP OFF)
endif ()

# Set the WHICH_OSI_QP macro according to these following priorities:
# - use the given WHICH_OSI_QP in input as CMake option, if any
# - use Gurobi, i.e., 1, if available
# - use Cplex, i.e., 2, if available
# - otherwise, use Clp

# if WHICH_OSI_QP is *not* given in input as CMake option...
# (`NOT DEFINED WHICH_OSI_QP` tests `set( )`, not `add_definitions( )`)
if (NOT DEFINED WHICH_OSI_QP)
    # ... then use OsiGrb in BundleSolver by default (as in BundleSolver.cpp:70)
    add_definitions(-DWHICH_OSI_QP=2)
else () # else if WHICH_OSI_QP is given in input as CMake option...
    if (WHICH_OSI_QP MATCHES "^(CPLEX|cplex|CPX|cpx|1)$")
        add_definitions(-DWHICH_OSI_QP=1) # ... then use OsiCpx in BundleSolver
    elseif (WHICH_OSI_QP MATCHES "^(GUROBI|gurobi|GRB|grb|2)$")
        add_definitions(-DWHICH_OSI_QP=2) # ... then use OsiGrb in BundleSolver
    endif ()
endif ()

# if Cplex is found...
if (Osi_OsiCpx_FOUND)
    # ... and if WHICH_OSI_QP is *not* given in input as CMake option...
    # (`NOT DEFINED WHICH_OSI_QP` tests `set( )`, not `add_definitions( )`)
    if (NOT DEFINED WHICH_OSI_QP)
        # ... then use OsiCpx in BundleSolver
        add_definitions(-DWHICH_OSI_QP=1)
    endif ()
endif ()

# but, if also Gurobi is found...
if (Osi_OsiGrb_FOUND)
    # ... and if WHICH_OSI_QP is *not* given in input as CMake option...
    # (`NOT DEFINED WHICH_OSI_QP` tests `set( )`, not `add_definitions( )`)
    if (NOT DEFINED WHICH_OSI_QP)
        # ... then use OsiGrb in BundleSolver
        add_definitions(-DWHICH_OSI_QP=2)
    endif ()
endif ()

if (NOT hasParent)
    # If it's not being called by the umbrella, we need to
    # look for the system-installed libraries.
    find_package(MILPSolver REQUIRED)
    find_package(NDOFiOracle REQUIRED)
else ()
    # NDOFiOracle library, required by BundleSolver.
    # Requires CoinUtils, Osi and CPLEX.
    set(NDOFiOracle_READ_PATHS OFF CACHE BOOL
                               "Whether NdoFiOracle will read locations for
		                       dependencies or not." FORCE)
    add_subdirectory(NdoFiOracle)

    # Add the binary directory to RPATH, so executables that use
    # dynamic loading will look into it.
    set(SMSPP_BUILD_RPATH
        "${SMSPP_BUILD_RPATH};${CMAKE_CURRENT_BINARY_DIR}/NdoFiOracle")
endif ()

# ----- Configuration header ------------------------------------------------ #
# This will generate a *Config.h header in the build directory.
configure_file(cmake/${modName}Config.h.in ${modName}Config.h)

# ----- Library ------------------------------------------------------------- #
# With the BUILD_SHARED_LIBS variable we can specify if the library will
# be STATIC or SHARED, so no reason to do it now.
add_library(${modName})
target_compile_features(${modName} PUBLIC cxx_std_17)

# When adding source files with target_sources(), PRIVATE means that the files
# should only be added to this library, whereas PUBLIC means they should be
# added to this library and to any target that links to it.
# INTERFACE can be used for sources that should not be added to this library
# but should be added to anything that links to it.
# Note: do not GLOB files here.
target_sources(${modName} PRIVATE
               BundleSolver.cpp
               ParallelBundleSolver.cpp)

# When using target_include_directories(), PUBLIC means that any targets
# that link to this target also need that include directory.
# Other options are PRIVATE (only affect the current target, not dependencies),
# and INTERFACE (only needed for dependencies).

# Different INSTALL_INTERFACE and BUILD_INTERFACE paths are used when
# generating the target import file (***Targets.cmake).
# This means that if a target finds this library in its build directory
# will look into the BUILD_INTERFACE path for its headers, if it finds it
# installed in the system will look into the INSTALL_INTERFACE path.
target_include_directories(
        ${modName} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include/${modNamespace}>)

# When linking other targets to the library with target_link_libraries(),
# PRIVATE means that the libraries will be linked only to this library,
# PUBLIC means they will be linked also to the targets that depend on this
# library, INTERFACE means they will be linked only to the targets that depend
# on this library.
target_link_libraries(${modName} PUBLIC
                      ${modNamespace}::MILPSolver
                      NDOFiOracle::NDOFiOracle)

# if NDOFiOracle library is *not* given as subdirectory since it is
# in the system-installed libraries, we need to link the Coin::Osi* targets here
if (NOT hasParent)
    message(STATUS "${modName}: OsiClp found, configuring BundleSolver")
    target_link_libraries(${modName} PUBLIC Coin::OsiClp)

    if (Osi_OsiCpx_FOUND)
        message(STATUS "${modName}: OsiCpx found, configuring BundleSolver")
        target_link_libraries(${modName} PUBLIC Coin::OsiCpx)
    endif ()

    if (Osi_OsiGrb_FOUND)
        message(STATUS "${modName}: OsiGrb found, configuring BundleSolver")
        target_link_libraries(${modName} PUBLIC Coin::OsiGrb)
    endif ()
endif ()

# This alias is defined so that executables in this same project can use
# the library with this notation.
add_library(${modNamespace}::${modName} ALIAS ${modName})

# ----- Subdirectories ------------------------------------------------------ #

if (BUILD_TESTING)
#    add_subdirectory(test)
endif ()

# ----- Install instructions ------------------------------------------------ #
# The following commands are used when installing the library
# and its CMake configuration files on the system.
# They are not required for local builds (see below).

include(GNUInstallDirs)

# Install the library
install(TARGETS ${modName}
        EXPORT ${modName}Targets
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

# Install the headers
install(FILES
        ${CMAKE_CURRENT_SOURCE_DIR}/BundleSolver.h
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${modNamespace})

# Install the auto-generated configuration header (see above).
install(FILES ${PROJECT_BINARY_DIR}/${modName}Config.h
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${modNamespace})

# (Generate and) install the target import file, that allows other
# CMake projects to import the target.
install(EXPORT ${modName}Targets
        NAMESPACE ${modNamespace}::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${modName})

# Generate the package version file, that allows other
# CMake projects to know the version.
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/${modName}ConfigVersion.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY AnyNewerVersion)

# Generate the package configuration file, that allows other
# CMake projects to find the library with find_package().
configure_package_config_file(
        ${CMAKE_CURRENT_LIST_DIR}/cmake/${modName}Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${modName}Config.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${modName})

# Install the package version and configuration files.
install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/${modName}Config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/${modName}ConfigVersion.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${modName})

# Install the README and LICENSE files.
install(FILES
        ${CMAKE_CURRENT_LIST_DIR}/README.md
        ${CMAKE_CURRENT_LIST_DIR}/LICENSE
        DESTINATION ${CMAKE_INSTALL_DATADIR}/${modName})

# ----- Add the build tree to RPATH------------------------------------------ #
# Add the binary directory to RPATH, so executables that use
# dynamic loading will look into it.
if (hasParent)
    set(SMSPP_BUILD_RPATH
        "${SMSPP_BUILD_RPATH};${CMAKE_CURRENT_BINARY_DIR}"
        PARENT_SCOPE)
endif ()

# ----- Packaging support --------------------------------------------------- #
# The following commands allow one to build a proper package using CPack.
# See: https://cmake.org/cmake/help/latest/manual/cpack.1.html

# Using this with the umbrella project is too messy.
if (NOT hasParent)
    set(CPACK_PACKAGE_VENDOR "SMS++ Team")
    set(CPACK_PACKAGE_CONTACT "Niccolo' Iardella")
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY ${PROJECT_DESCRIPTION})
    set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_LIST_DIR}/LICENSE")
    set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_LIST_DIR}/README.md")
    set(CPACK_SOURCE_IGNORE_FILES
        \\\\.git/
        \\\\.gitignore$
        \\\\.gitattributes$)
    include(CPack)
endif ()

# --------------------------------------------------------------------------- #
# Remove from the search paths the cmake directory we added at the beginning.
list(REMOVE_AT CMAKE_MODULE_PATH -1)

# --------------------------------------------------------------------------- #
