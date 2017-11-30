# Enforce some CMake policies
cmake_minimum_required(VERSION 3.3)
if (POLICY CMP0054)
  # FIXME: This is horrible. With the old behaviour,
  # quoted strings like "MSVC" in if() conditionals
  # get implicitly dereferenced. The NEW behaviour
  # doesn't do this but CMP0054 was only introduced
  # in CMake 3.1 and we support lower versions as the
  # minimum. We could set NEW here but it would be very
  # confusing to use NEW for some builds and OLD for others
  # which could lead to some subtle bugs. Instead when the
  # minimum version is 3.1 change this policy to NEW and remove
  # the hacks in place to work around it.
  cmake_policy(SET CMP0054 OLD)
endif()

set(CMAKE_USER_MAKE_RULES_OVERRIDE_CXX "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cxx_compiler_flags_overrides.cmake")
project(Z3 CXX)

################################################################################
# Project version
################################################################################
set(Z3_VERSION_MAJOR 4)
set(Z3_VERSION_MINOR 5)
set(Z3_VERSION_PATCH 1)
set(Z3_VERSION_TWEAK 0)
set(Z3_VERSION "${Z3_VERSION_MAJOR}.${Z3_VERSION_MINOR}.${Z3_VERSION_PATCH}.${Z3_VERSION_TWEAK}")
set(Z3_FULL_VERSION_STR "${Z3_VERSION}") # Note this might be modified
message(STATUS "Z3 version ${Z3_VERSION}")

################################################################################
# Set various useful variables depending on CMake version
################################################################################
set(ADD_CUSTOM_COMMAND_USES_TERMINAL_ARG "USES_TERMINAL")
set(ADD_CUSTOM_TARGET_USES_TERMINAL_ARG "USES_TERMINAL")

################################################################################
# Message for polluted source tree sanity checks
################################################################################
set(z3_polluted_tree_msg
  " should not exist and is polluting the source tree."
  " It is likely that this file came from the Python build system which"
  " generates files inside the source tree. This is bad practice and the CMake"
  " build system is setup to make sure that the source tree is clean during"
  " its configure step. If you are using git you can remove all untracked files"
  " using ``git clean -fx``. Be careful when doing this. You should probably use"
  " this with ``-n`` first to check which file(s) would be removed."
)

################################################################################
# Sanity check - Disallow building in source
################################################################################
if ("${PROJECT_SOURCE_DIR}" STREQUAL "${PROJECT_BINARY_DIR}")
  message(FATAL_ERROR "In source builds are not allowed. You should invoke "
          "CMake from a different directory.")
endif()

################################################################################
# Add our CMake module directory to the list of module search directories
################################################################################
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/modules")

################################################################################
# Useful CMake functions/Macros
################################################################################
include(CheckCXXSourceCompiles)

################################################################################
# Compiler flags for Z3 components.
# Subsequent commands will append to this
################################################################################
set(Z3_COMPONENT_CXX_DEFINES "")
set(Z3_COMPONENT_CXX_FLAGS "")
set(Z3_COMPONENT_EXTRA_INCLUDE_DIRS "")
set(Z3_DEPENDENT_LIBS "")
set(Z3_DEPENDENT_EXTRA_CXX_LINK_FLAGS "")

################################################################################
# Build type
################################################################################
message(STATUS "CMake generator: ${CMAKE_GENERATOR}")
set(available_build_types Debug Release RelWithDebInfo MinSizeRel)
if (DEFINED CMAKE_CONFIGURATION_TYPES)
  # Multi-configuration build (e.g. Visual Studio and Xcode). Here
  # CMAKE_BUILD_TYPE doesn't matter
  message(STATUS "Available configurations: ${CMAKE_CONFIGURATION_TYPES}")
else()
  # Single configuration generator (e.g. Unix Makefiles, Ninja)
  if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "CMAKE_BUILD_TYPE is not set. Setting default")
    message(STATUS "The available build types are: ${available_build_types}")
    set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE String
        "Options are ${available_build_types}"
        FORCE)
    # Provide drop down menu options in cmake-gui
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS ${available_build_types})
  endif()
  message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

  # Check the selected build type is valid
  list(FIND available_build_types "${CMAKE_BUILD_TYPE}" _build_type_index)
  if ("${_build_type_index}" EQUAL "-1")
    message(FATAL_ERROR "\"${CMAKE_BUILD_TYPE}\" is an invalid build type.\n"
      "Use one of the following build types ${available_build_types}")
  endif()
endif()

# CMAKE_BUILD_TYPE has no meaning for multi-configuration generators
# (e.g. Visual Studio) so use generator expressions instead to add
# the right definitions when doing a particular build type.
#
# Note for some reason we have to leave off ``-D`` here otherwise
# we get ``-D-DZ3DEBUG`` passed to the compiler
list(APPEND Z3_COMPONENT_CXX_DEFINES $<$<CONFIG:Debug>:Z3DEBUG>)
list(APPEND Z3_COMPONENT_CXX_DEFINES $<$<CONFIG:Debug>:LEAN_DEBUG>)
list(APPEND Z3_COMPONENT_CXX_DEFINES $<$<CONFIG:Release>:_EXTERNAL_RELEASE>)
list(APPEND Z3_COMPONENT_CXX_DEFINES $<$<CONFIG:RelWithDebInfo>:_EXTERNAL_RELEASE>)

################################################################################
# Find Python
################################################################################
find_package(PythonInterp REQUIRED)
message(STATUS "PYTHON_EXECUTABLE: ${PYTHON_EXECUTABLE}")

################################################################################
# Target architecture detection
################################################################################
include(${PROJECT_SOURCE_DIR}/cmake/target_arch_detect.cmake)
detect_target_architecture(TARGET_ARCHITECTURE)
message(STATUS "Detected target architecture: ${TARGET_ARCHITECTURE}")
if ("${TARGET_ARCHITECTURE}" STREQUAL "x86_64")
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_AMD64_")
endif()


################################################################################
# Function for detecting C++ compiler flag support
################################################################################
include(${PROJECT_SOURCE_DIR}/cmake/z3_add_cxx_flag.cmake)

################################################################################
# Platform detection
################################################################################
if ("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  message(STATUS "Platform: Linux")
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_LINUX_")
  if ("${TARGET_ARCHITECTURE}" STREQUAL "x86_64")
    list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_USE_THREAD_LOCAL")
  endif()
elseif ("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  # Does OSX really not need any special flags?
  message(STATUS "Platform: Darwin")
elseif ("${CMAKE_SYSTEM_NAME}" MATCHES "FreeBSD")
  message(STATUS "Platform: FreeBSD")
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_FREEBSD_")
elseif ("${CMAKE_SYSTEM_NAME}" MATCHES "OpenBSD")
  message(STATUS "Platform: OpenBSD")
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_OPENBSD_")
elseif (CYGWIN)
  message(STATUS "Platform: Cygwin")
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_CYGWIN")
elseif (WIN32)
  message(STATUS "Platform: Windows")
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_WINDOWS")
else()
  message(FATAL_ERROR "Platform \"${CMAKE_SYSTEM_NAME}\" not recognised")
endif()

list(APPEND Z3_COMPONENT_EXTRA_INCLUDE_DIRS 
  "${PROJECT_BINARY_DIR}/src"
  "${PROJECT_SOURCE_DIR}/src"
)
################################################################################
# GNU multiple precision library support
################################################################################
option(Z3_USE_LIB_GMP "Use GNU Multiple Precision Library" OFF)
if (Z3_USE_LIB_GMP)
  # Because this is off by default we will make the configure fail if libgmp
  # can't be found
  find_package(GMP REQUIRED)
  message(STATUS "Using libgmp")
  list(APPEND Z3_DEPENDENT_LIBS ${GMP_C_LIBRARIES})
  list(APPEND Z3_COMPONENT_EXTRA_INCLUDE_DIRS ${GMP_INCLUDE_DIRS})
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_MP_GMP")
else()
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_MP_INTERNAL")
  message(STATUS "Not using libgmp")
endif()

################################################################################
# OpenMP support
################################################################################
find_package(OpenMP)
if (OPENMP_FOUND)
  set(Z3_USE_OPENMP_DEFAULT ON)
else()
  set(Z3_USE_OPENMP_DEFAULT OFF)
endif()
# By setting `Z3_USE_OPENMP` this way configuration will fail during the first
# configure if the user explicitly passes `-DZ3_USE_OPENMP=ON` and the compiler
# does not support OpenMP. However if the option is not set explicitly during
# the first configure OpenMP support will be automatically enabled/disabled
# depending on whether OpenMP is available.
option(Z3_USE_OPENMP "Use OpenMP" ${Z3_USE_OPENMP_DEFAULT})

if (Z3_USE_OPENMP)
  if (NOT OPENMP_FOUND)
    message(FATAL_ERROR "Z3_USE_OPENMP is ON but your compiler does not support OpenMP")
  endif()

  list(APPEND Z3_COMPONENT_CXX_FLAGS ${OpenMP_CXX_FLAGS})
  # GCC and Clang need to have additional flags passed to the linker.
  # We can't do ``target_link_libraries(libz3 INTERFACE ${OpenMP_CXX_FLAGS})``
  # because ``/openmp`` is interpreted as file name rather than a linker
  # flag by MSVC and breaks the build
  if (("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang") OR
      ("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU"))
    list(APPEND Z3_DEPENDENT_EXTRA_CXX_LINK_FLAGS ${OpenMP_CXX_FLAGS})
  endif()
  message(STATUS "Using OpenMP")
else()
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_NO_OMP_")
  message(STATUS "Not using OpenMP")
endif()

################################################################################
# API Log sync
################################################################################
option(Z3_API_LOG_SYNC
  "Use locking when logging Z3 API calls (experimental)"
  OFF
)
if (Z3_API_LOG_SYNC)
  if (NOT Z3_USE_OPENMP)
    message(FATAL_ERROR "Z3_API_LOG_SYNC feature requires OpenMP")
  endif()
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-DZ3_LOG_SYNC")
  message(STATUS "Using Z3_API_LOG_SYNC")
else()
  message(STATUS "Not using Z3_API_LOG_SYNC")
endif()

################################################################################
# FP math
################################################################################
# FIXME: Support ARM "-mfpu=vfp -mfloat-abi=hard"
if (("${TARGET_ARCHITECTURE}" STREQUAL "x86_64") OR ("${TARGET_ARCHITECTURE}" STREQUAL "i686"))
  if (("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU") OR ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang"))
    set(SSE_FLAGS "-mfpmath=sse" "-msse" "-msse2")
    # FIXME: Remove "x.." when CMP0054 is set to NEW
  elseif ("x${CMAKE_CXX_COMPILER_ID}" STREQUAL "xMSVC")
    set(SSE_FLAGS "/arch:SSE2")
  else()
    message(FATAL_ERROR "Unknown compiler ${CMAKE_CXX_COMPILER_ID}")
  endif()
  CHECK_CXX_COMPILER_FLAG("${SSE_FLAGS}" HAS_SSE2)
  if (HAS_SSE2)
    list(APPEND Z3_COMPONENT_CXX_FLAGS ${SSE_FLAGS})
  endif()
  unset(SSE_FLAGS)
endif()



# FIXME: Remove "x.." when CMP0054 is set to NEW
if ("x${CMAKE_CXX_COMPILER_ID}" STREQUAL "xMSVC")
  # This is the default for MSVC already but to replicate the
  # python/Makefile build system behaviour this flag is set
  # explicitly.
  z3_add_cxx_flag("/fp:precise" REQUIRED)
endif()
# There doesn't seem to be an equivalent for clang/gcc

################################################################################
# Threading support
################################################################################
find_package(Threads)
list(APPEND Z3_DEPENDENT_LIBS ${CMAKE_THREAD_LIBS_INIT})

################################################################################
# Compiler warnings
################################################################################
include(${PROJECT_SOURCE_DIR}/cmake/compiler_warnings.cmake)

################################################################################
# Option to control what type of library we build
################################################################################
option(Z3_BUILD_LIBZ3_SHARED "Build libz3 as a shared library if true, otherwise build a static library" OFF)


################################################################################
# Symbol visibility
################################################################################
if (NOT MSVC)
  z3_add_cxx_flag("-fvisibility=hidden" REQUIRED)
endif()

################################################################################
# Tracing
################################################################################
option(Z3_ENABLE_TRACING_FOR_NON_DEBUG "Enable tracing in non-debug builds." OFF)
if (Z3_ENABLE_TRACING_FOR_NON_DEBUG)
  list(APPEND Z3_COMPONENT_CXX_DEFINES "-D_TRACE")
else()
  # Tracing is always enabled in debug builds
  list(APPEND Z3_COMPONENT_CXX_DEFINES $<$<CONFIG:Debug>:_TRACE>)
endif()

################################################################################
# Postion independent code
################################################################################
# This is required because code built in the components will end up in a shared
# library. If not building a shared library ``-fPIC`` isn't needed and would add
# unnecessary overhead.
if (Z3_BUILD_LIBZ3_SHARED)
  # Avoid adding -fPIC compiler switch if we compile with MSVC (which does not
  # support the flag) or if we target Windows, which generally does not use
  # position independent code for native code shared libraries (DLLs).
  if (NOT (MSVC OR MINGW OR WIN32))
    z3_add_cxx_flag("-fPIC" REQUIRED)
  endif()
endif()

################################################################################
# Link time optimization
################################################################################
include(${PROJECT_SOURCE_DIR}/cmake/compiler_lto.cmake)

################################################################################
# Control flow integrity
################################################################################
option(Z3_ENABLE_CFI "Enable control flow integrity checking" OFF)
if (Z3_ENABLE_CFI)
  set(build_types_with_cfi "RELEASE" "RELWITHDEBINFO")
  if (NOT Z3_LINK_TIME_OPTIMIZATION)
    message(FATAL_ERROR "Cannot enable control flow integrity checking without link-time optimization."
      "You should set Z3_LINK_TIME_OPTIMIZATION to ON or Z3_ENABLE_CFI to OFF.")
  endif()
  if (DEFINED CMAKE_CONFIGURATION_TYPES)
    # Multi configuration generator
    message(STATUS "Note CFI is only enabled for the following configurations: ${build_types_with_cfi}")
    # No need for else because this is the same as the set that LTO requires.
  endif()
  if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    z3_add_cxx_flag("-fsanitize=cfi" REQUIRED)
    z3_add_cxx_flag("-fsanitize-cfi-cross-dso" REQUIRED)
  elseif ("x${CMAKE_CXX_COMPILER_ID}" STREQUAL "xMSVC")
    z3_add_cxx_flag("/guard:cf" REQUIRED)
    message(STATUS "Enabling CFI for MSVC")
    foreach (_build_type ${build_types_with_cfi})
      message(STATUS "Enabling CFI for MSVC")
      string(APPEND CMAKE_EXE_LINKER_FLAGS_${_build_type} " /GUARD:CF")
      string(APPEND CMAKE_SHARED_LINKER_FLAGS_${_build_type} " /GUARD:CF")
    endforeach()
  else()
    message(FATAL_ERROR "Can't enable control flow integrity for compiler \"${CMAKE_CXX_COMPILER_ID}\"."
      "You should set Z3_ENABLE_CFI to OFF or use Clang or MSVC to compile.")
  endif()
endif()

################################################################################
# MSVC specific flags inherited from old build system
################################################################################
# FIXME: Remove "x.." when CMP0054 is set to NEW
if ("x${CMAKE_CXX_COMPILER_ID}" STREQUAL "xMSVC")
  include(${PROJECT_SOURCE_DIR}/cmake/msvc_legacy_quirks.cmake)
endif()

################################################################################
# Report default CMake flags
################################################################################
# This is mainly for debugging.
message(STATUS "CMAKE_CXX_FLAGS: \"${CMAKE_CXX_FLAGS}\"")
message(STATUS "CMAKE_EXE_LINKER_FLAGS: \"${CMAKE_EXE_LINKER_FLAGS}\"")
message(STATUS "CMAKE_STATIC_LINKER_FLAGS: \"${CMAKE_STATIC_LINKER_FLAGS}\"")
message(STATUS "CMAKE_SHARED_LINKER_FLAGS: \"${CMAKE_SHARED_LINKER_FLAGS}\"")
if (DEFINED CMAKE_CONFIGURATION_TYPES)
  # Multi configuration generator
  string(TOUPPER "${available_build_types}" build_types_to_report)
else()
  # Single configuration generator
  string(TOUPPER "${CMAKE_BUILD_TYPE}" build_types_to_report)
endif()
foreach (_build_type ${build_types_to_report})
  message(STATUS "CMAKE_CXX_FLAGS_${_build_type}: \"${CMAKE_CXX_FLAGS_${_build_type}}\"")
  message(STATUS "CMAKE_EXE_LINKER_FLAGS_${_build_type}: \"${CMAKE_EXE_LINKER_FLAGS_${_build_type}}\"")
  message(STATUS "CMAKE_SHARED_LINKER_FLAGS_${_build_type}: \"${CMAKE_SHARED_LINKER_FLAGS_${_build_type}}\"")
  message(STATUS "CMAKE_STATIC_LINKER_FLAGS_${_build_type}: \"${CMAKE_STATIC_LINKER_FLAGS_${_build_type}}\"")
endforeach()

################################################################################
# Report Z3_COMPONENT flags
################################################################################
message(STATUS "Z3_COMPONENT_CXX_DEFINES: ${Z3_COMPONENT_CXX_DEFINES}")
message(STATUS "Z3_COMPONENT_CXX_FLAGS: ${Z3_COMPONENT_CXX_FLAGS}")
message(STATUS "Z3_DEPENDENT_LIBS: ${Z3_DEPENDENT_LIBS}")
message(STATUS "Z3_COMPONENT_EXTRA_INCLUDE_DIRS: ${Z3_COMPONENT_EXTRA_INCLUDE_DIRS}")
message(STATUS "Z3_DEPENDENT_EXTRA_CXX_LINK_FLAGS: ${Z3_DEPENDENT_EXTRA_CXX_LINK_FLAGS}")

################################################################################
# Z3 installation locations
################################################################################
include(GNUInstallDirs)
set(CMAKE_INSTALL_PKGCONFIGDIR
  "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
  CACHE
  PATH
  "Directory to install pkgconfig files"
)

set(CMAKE_INSTALL_Z3_CMAKE_PACKAGE_DIR
  "${CMAKE_INSTALL_LIBDIR}/cmake/z3"
  CACHE
  PATH
  "Directory to install Z3 CMake package files"
)
message(STATUS "CMAKE_INSTALL_LIBDIR: \"${CMAKE_INSTALL_LIBDIR}\"")
message(STATUS "CMAKE_INSTALL_BINDIR: \"${CMAKE_INSTALL_BINDIR}\"")
message(STATUS "CMAKE_INSTALL_INCLUDEDIR: \"${CMAKE_INSTALL_INCLUDEDIR}\"")
message(STATUS "CMAKE_INSTALL_PKGCONFIGDIR: \"${CMAKE_INSTALL_PKGCONFIGDIR}\"")
message(STATUS "CMAKE_INSTALL_Z3_CMAKE_PACKAGE_DIR: \"${CMAKE_INSTALL_Z3_CMAKE_PACKAGE_DIR}\"")

################################################################################
# Uninstall rule
################################################################################
configure_file(
  "${PROJECT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  @ONLY
)

# Target needs to be declared before the components so that they can add
# dependencies to this target so they can run their own custom uninstall rules.
add_custom_target(uninstall
  COMMAND
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  COMMENT "Uninstalling..."
  ${ADD_CUSTOM_COMMAND_USES_TERMINAL_ARG}
  VERBATIM
)

################################################################################
# CMake build file locations
################################################################################
# To mimic the python build system output these into the root of the build
# directory
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")

################################################################################
# Extra dependencies for build rules that use the Python infrastructure to
# generate files used for Z3's build. Changes to these files will trigger
# a rebuild of all the generated files.
################################################################################
# Note: ``update_api.py`` is deliberately not here because it not used
# to generate every generated file. The targets that need it list it explicitly.
set(Z3_GENERATED_FILE_EXTRA_DEPENDENCIES
  "${PROJECT_SOURCE_DIR}/scripts/mk_genfile_common.py"
)

################################################################################
# Z3 components, library and executables
################################################################################
include(${PROJECT_SOURCE_DIR}/cmake/z3_add_component.cmake)
include(${PROJECT_SOURCE_DIR}/cmake/z3_append_linker_flag_list_to_target.cmake)
add_subdirectory(src)

################################################################################
# Create `Z3Config.cmake` and related files for the build tree so clients can
# use Z3 via CMake.
################################################################################
include(CMakePackageConfigHelpers)
export(EXPORT Z3_EXPORTED_TARGETS
  NAMESPACE z3::
  FILE "${PROJECT_BINARY_DIR}/cmake/Z3Targets.cmake"
)
set(Z3_FIRST_PACKAGE_INCLUDE_DIR "${PROJECT_BINARY_DIR}/src/api")
set(Z3_SECOND_PACKAGE_INCLUDE_DIR "${PROJECT_SOURCE_DIR}/src/api")
set(Z3_CXX_PACKAGE_INCLUDE_DIR "${PROJECT_SOURCE_DIR}/src/api/c++")
set(AUTO_GEN_MSG "Automatically generated. DO NOT EDIT")
set(CONFIG_FILE_TYPE "build tree")
configure_package_config_file("${PROJECT_SOURCE_DIR}/cmake/Z3Config.cmake.in"
  "Z3Config.cmake"
  INSTALL_DESTINATION "${PROJECT_BINARY_DIR}"
  PATH_VARS
    Z3_FIRST_PACKAGE_INCLUDE_DIR
    Z3_SECOND_PACKAGE_INCLUDE_DIR
    Z3_CXX_PACKAGE_INCLUDE_DIR
)
unset(Z3_FIRST_PACKAGE_INCLUDE_DIR)
unset(Z3_SECOND_PACKAGE_INCLUDE_DIR)
unset(Z3_CXX_PACKAGE_INCLUDE_DIR)
unset(AUTO_GEN_MSG)
unset(CONFIG_FILE_TYPE)
# TODO: Provide a `Z3Version.cmake` file so that clients can specify the version
# of Z3 they want.

################################################################################
# Create `Z3Config.cmake` and related files for install tree so clients can use
# Z3 via CMake.
################################################################################
install(EXPORT
  Z3_EXPORTED_TARGETS
  FILE "Z3Targets.cmake"
  NAMESPACE z3::
  DESTINATION "${PROJECT_BINARY_DIR}/cmake"
)

set(Z3_INSTALL_TREE_CMAKE_CONFIG_FILE "${PROJECT_BINARY_DIR}/cmake/Z3Config.cmake")
set(Z3_FIRST_PACKAGE_INCLUDE_DIR "${CMAKE_INSTALL_INCLUDEDIR}")
set(Z3_SECOND_INCLUDE_DIR "")
set(Z3_CXX_PACKAGE_INCLUDE_DIR "")
set(AUTO_GEN_MSG "Automatically generated. DO NOT EDIT")
set(CONFIG_FILE_TYPE "install tree")
# We use `configure_package_config_file()` to try and create CMake files
# that are re-locatable so that it doesn't matter if the files aren't placed
# in the original install prefix.
configure_package_config_file("${PROJECT_SOURCE_DIR}/cmake/Z3Config.cmake.in"
  "${Z3_INSTALL_TREE_CMAKE_CONFIG_FILE}"
  INSTALL_DESTINATION "${CMAKE_INSTALL_Z3_CMAKE_PACKAGE_DIR}"
  PATH_VARS Z3_FIRST_PACKAGE_INCLUDE_DIR
)
unset(Z3_FIRST_PACKAGE_INCLUDE_DIR)
unset(Z3_SECOND_PACKAGE_INCLUDE_DIR)
unset(Z3_CXX_PACKAGE_INCLUDE_DIR)
unset(AUTO_GEN_MSG)
unset(CONFIG_FILE_TYPE)

# Add install rule to install ${Z3_INSTALL_TREE_CMAKE_CONFIG_FILE}
install(
  FILES "${Z3_INSTALL_TREE_CMAKE_CONFIG_FILE}"
  DESTINATION "${CMAKE_INSTALL_Z3_CMAKE_PACKAGE_DIR}"
)