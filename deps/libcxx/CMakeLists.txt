# See www/CMake.html for instructions on how to build libcxx with CMake.

#===============================================================================
# Setup Project
#===============================================================================

project(libcxx CXX C)
cmake_minimum_required(VERSION 2.8)

set(PACKAGE_NAME libcxx)
set(PACKAGE_VERSION trunk-svn)
set(PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")
set(PACKAGE_BUGREPORT "llvmbugs@cs.uiuc.edu")

# Add path for custom modules
set(CMAKE_MODULE_PATH
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake"
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules"
  ${CMAKE_MODULE_PATH}
  )

# Require out of source build.
include(MacroEnsureOutOfSourceBuild)
MACRO_ENSURE_OUT_OF_SOURCE_BUILD(
 "${PROJECT_NAME} requires an out of source build. Please create a separate
 build directory and run 'cmake /path/to/${PROJECT_NAME} [options]' there."
 )

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  set(LIBCXX_BUILT_STANDALONE 1)
endif()

#===============================================================================
# Setup CMake Options
#===============================================================================

# Define options.
option(LIBCXX_ENABLE_EXCEPTIONS "Use exceptions." ON)
option(LIBCXX_ENABLE_RTTI "Use run time type information." ON)
option(LIBCXX_ENABLE_ASSERTIONS "Enable assertions independent of build mode." ON)
option(LIBCXX_ENABLE_PEDANTIC "Compile with pedantic enabled." ON)
option(LIBCXX_ENABLE_WERROR "Fail and stop if a warning is triggered." OFF)
option(LIBCXX_ENABLE_CXX1Y "Enable -std=c++1y and use of c++1y language features if the compiler supports it." OFF)
option(LIBCXX_ENABLE_SHARED "Build libc++ as a shared library." ON)
option(LIBCXX_INSTALL_SUPPORT_HEADERS "Install libc++ support headers." ON)
if (LIBCXX_BUILT_STANDALONE)
  set(LLVM_USE_SANITIZER "" CACHE STRING
      "Define the sanitizer used to build the library and tests")
endif()

set(CXXABIS none libcxxabi libcxxrt libstdc++ libsupc++)
if (NOT LIBCXX_CXX_ABI)
  if (NOT DEFINED LIBCXX_BUILT_STANDALONE AND
      IS_DIRECTORY "${CMAKE_SOURCE_DIR}/projects/libcxxabi")
    set(LIBCXX_CXX_ABI_LIBNAME "libcxxabi")
    set(LIBCXX_LIBCXXABI_INCLUDE_PATHS "${CMAKE_SOURCE_DIR}/projects/libcxxabi/include")
    set(LIBCXX_CXX_ABI_INTREE 1)
  else ()
    set(LIBCXX_CXX_ABI_LIBNAME "none")
  endif ()
else ()
  set(LIBCXX_CXX_ABI_LIBNAME "${LIBCXX_CXX_ABI}")
endif ()
set(LIBCXX_CXX_ABI "${LIBCXX_CXX_ABI}" CACHE STRING
    "Specify C++ ABI library to use." FORCE)
set_property(CACHE LIBCXX_CXX_ABI PROPERTY STRINGS ;${CXXABIS})

#===============================================================================
# Configure System
#===============================================================================

# Get triples.
include(GetTriple)
get_host_triple(LIBCXX_HOST_TRIPLE
  LIBCXX_HOST_ARCH
  LIBCXX_HOST_VENDOR
  LIBCXX_HOST_OS
  )
set(LIBCXX_HOST_TRIPLE ${LIBCXX_HOST_TRIPLE} CACHE STRING "Host triple.")
get_target_triple(LIBCXX_TARGET_TRIPLE
  LIBCXX_TARGET_ARCH
  LIBCXX_TARGET_VENDOR
  LIBCXX_TARGET_OS
  )
set(LIBCXX_TARGET_TRIPLE ${LIBCXX_TARGET_TRIPLE} CACHE STRING "Target triple.")

#===============================================================================
# Add an ABI library if appropriate
#===============================================================================

#
# _setup_abi: Set up the build to use an ABI library
#
# Parameters:
#   abidefines: A list of defines needed to compile libc++ with the ABI library
#   abilibs   : A list of libraries to link against
#   abifiles  : A list of files (which may be relative paths) to copy into the
#               libc++ build tree for the build.  These files will also be
#               installed alongside the libc++ headers.
#   abidirs   : A list of relative paths to create under an include directory
#               in the libc++ build directory.
#
macro(setup_abi_lib abipathvar abidefines abilibs abifiles abidirs)
  list(APPEND LIBCXX_CXX_FEATURE_FLAGS ${abidefines})
  set(${abipathvar} "${${abipathvar}}"
    CACHE PATH
    "Paths to C++ ABI header directories separated by ';'." FORCE
    )
  set(LIBCXX_CXX_ABI_LIBRARIES ${abilibs})
  set(LIBCXX_ABILIB_FILES ${abifiles})

  file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/include")
  foreach(_d ${abidirs})
    file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/include/${_d}")
  endforeach()

  foreach(fpath ${LIBCXX_ABILIB_FILES})
    set(found FALSE)
    foreach(incpath ${${abipathvar}})
      if (EXISTS "${incpath}/${fpath}")
        set(found TRUE)
        get_filename_component(dstdir ${fpath} PATH)
        get_filename_component(ifile ${fpath} NAME)
        file(COPY "${incpath}/${fpath}"
          DESTINATION "${CMAKE_BINARY_DIR}/include/${dstdir}"
          )
        install(FILES "${CMAKE_BINARY_DIR}/include/${fpath}"
          DESTINATION include/c++/v1/${dstdir}
          PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
          )
        list(APPEND abilib_headers "${CMAKE_BINARY_DIR}/include/${fpath}")
      endif()
    endforeach()
    if (NOT found)
      message(FATAL_ERROR "Failed to find ${fpath}")
    endif()
  endforeach()

  add_custom_target(LIBCXX_CXX_ABI_DEPS DEPENDS ${abilib_headers})
  include_directories("${CMAKE_BINARY_DIR}/include")

endmacro()

if ("${LIBCXX_CXX_ABI_LIBNAME}" STREQUAL "libstdc++" OR
    "${LIBCXX_CXX_ABI_LIBNAME}" STREQUAL "libsupc++")
  set(_LIBSUPCXX_INCLUDE_FILES
    cxxabi.h bits/c++config.h bits/os_defines.h bits/cpu_defines.h
    bits/cxxabi_tweaks.h bits/cxxabi_forced.h
    )
  if ("${LIBCXX_CXX_ABI_LIBNAME}" STREQUAL "libstdc++")
    set(_LIBSUPCXX_DEFINES "-DLIBSTDCXX")
    set(_LIBSUPCXX_LIBNAME stdc++)
  else()
    set(_LIBSUPCXX_DEFINES "")
    set(_LIBSUPCXX_LIBNAME supc++)
  endif()
  setup_abi_lib("LIBCXX_LIBSUPCXX_INCLUDE_PATHS"
    "-D__GLIBCXX__ ${_LIBSUPCXX_DEFINES}"
    "${_LIBSUPCXX_LIBNAME}" "${_LIBSUPCXX_INCLUDE_FILES}" "bits"
    )
elseif ("${LIBCXX_CXX_ABI_LIBNAME}" STREQUAL "libcxxabi")
  if (LIBCXX_CXX_ABI_INTREE)
    # Link against just-built "cxxabi" target.
    set(CXXABI_LIBNAME cxxabi)
  else()
    # Assume c++abi is installed in the system, rely on -lc++abi link flag.
    set(CXXABI_LIBNAME "c++abi")
  endif()
  setup_abi_lib("LIBCXX_LIBCXXABI_INCLUDE_PATHS" ""
    ${CXXABI_LIBNAME} "cxxabi.h" ""
    )
elseif ("${LIBCXX_CXX_ABI_LIBNAME}" STREQUAL "libcxxrt")
  setup_abi_lib("LIBCXX_LIBCXXRT_INCLUDE_PATHS" "-DLIBCXXRT"
    "cxxrt" "cxxabi.h;unwind.h;unwind-arm.h;unwind-itanium.h" ""
    )
elseif (NOT "${LIBCXX_CXX_ABI_LIBNAME}" STREQUAL "none")
  message(FATAL_ERROR
    "Currently libstdc++, libsupc++, libcxxabi, libcxxrt and none are "
    "supported for c++ abi."
    )
endif ()

# Configure compiler.
include(config-ix)

#===============================================================================
# Setup Compiler Flags
#===============================================================================

# Get required flags.
# On all systems the system c++ standard library headers need to be excluded.
if (MSVC)
  # MSVC only has -X, which disables all default includes; including the crt.
  # Thus, we do nothing and hope we don't accidentally include any of the C++
  # headers.
else()
  if (LIBCXX_HAS_NOSTDINCXX_FLAG)
    list(APPEND LIBCXX_CXX_REQUIRED_FLAGS -nostdinc++)
    string(REPLACE "-stdlib=libc++" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
  endif()
  # If c++1y has been enabled then attempt to use it. Fail if it is no supported
  # by the compiler. Otherwise choose c++11 and ensure the compiler supports it.
  if (LIBCXX_ENABLE_CXX1Y)
    if (LIBCXX_HAS_STDCXX1Y_FLAG)
      set(LIBCXX_STD_VERSION c++1y)
    else()
      message(FATAL_ERROR "c++1y was enabled but the compiler does not support it.")
    endif()
  else()
    if (LIBCXX_HAS_STDCXX11_FLAG)
      set(LIBCXX_STD_VERSION c++11)
    else()
      message(FATAL_ERROR "c++11 is required by libc++ but is not supported by the compiler")
    endif()
  endif()
  # LIBCXX_STD_VERSION should always be set at this point.
  list(APPEND LIBCXX_CXX_REQUIRED_FLAGS "-std=${LIBCXX_STD_VERSION}")
endif()

macro(append_if list condition var)
  if (${condition})
    list(APPEND ${list} ${var})
  endif()
endmacro()

# Get warning flags
if (NOT MSVC)
  append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_WALL_FLAG -Wall)
  list(APPEND LIBCXX_CXX_REQUIRED_FLAGS -Werror=return-type)
endif()

append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_W_FLAG -W)
append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_WNO_UNUSED_PARAMETER_FLAG -Wno-unused-parameter)
append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_WWRITE_STRINGS_FLAG -Wwrite-strings)
append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_WNO_LONG_LONG_FLAG -Wno-long-long)
if (LIBCXX_ENABLE_WERROR)
  append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_WERROR_FLAG -Werror)
  append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_WX_FLAG -WX)
else()
  append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_WNO_ERROR_FLAG -Wno-error)
  append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_NO_WX_FLAG -WX-)
endif()
if (LIBCXX_ENABLE_PEDANTIC)
  append_if(LIBCXX_CXX_WARNING_FLAGS LIBCXX_HAS_PEDANTIC_FLAG -pedantic)
endif()

# Get feature flags.
# Exceptions
if (LIBCXX_ENABLE_EXCEPTIONS)
  # Catches C++ exceptions only and tells the compiler to assume that extern C
  # functions never throw a C++ exception.
  append_if(LIBCXX_CXX_FEATURE_FLAGS LIBCXX_HAS_EHSC_FLAG -EHsc)
else()
  list(APPEND LIBCXX_CXX_FEATURE_FLAGS -D_LIBCPP_NO_EXCEPTIONS)
  append_if(LIBCXX_CXX_FEATURE_FLAGS LIBCXX_HAS_NO_EHS_FLAG -EHs-)
  append_if(LIBCXX_CXX_FEATURE_FLAGS LIBCXX_HAS_NO_EHA_FLAG -EHa-)
  append_if(LIBCXX_CXX_FEATURE_FLAGS LIBCXX_HAS_FNO_EXCEPTIONS_FLAG -fno-exceptions)
endif()
# RTTI
if (NOT LIBCXX_ENABLE_RTTI)
  list(APPEND LIBCXX_CXX_FEATURE_FLAGS -D_LIBCPP_NO_RTTI)
  append_if(LIBCXX_CXX_FEATURE_FLAGS LIBCXX_HAS_NO_GR_FLAG -GR-)
  append_if(LIBCXX_CXX_FEATURE_FLAGS LIBCXX_HAS_FNO_RTTI_FLAG -fno-rtti)
endif()
# Assert
string(TOUPPER "${CMAKE_BUILD_TYPE}" uppercase_CMAKE_BUILD_TYPE)
if (LIBCXX_ENABLE_ASSERTIONS)
  # MSVC doesn't like _DEBUG on release builds. See PR 4379.
  if (NOT MSVC)
    list(APPEND LIBCXX_CXX_FEATURE_FLAGS -D_DEBUG)
  endif()
  # On Release builds cmake automatically defines NDEBUG, so we
  # explicitly undefine it:
  if (uppercase_CMAKE_BUILD_TYPE STREQUAL "RELEASE")
    list(APPEND LIBCXX_CXX_FEATURE_FLAGS -UNDEBUG)
  endif()
else()
  if (NOT uppercase_CMAKE_BUILD_TYPE STREQUAL "RELEASE")
    list(APPEND LIBCXX_CXX_FEATURE_FLAGS -DNDEBUG)
  endif()
endif()
# Static library
if (NOT LIBCXX_ENABLE_SHARED)
  list(APPEND LIBCXX_CXX_FEATURE_FLAGS -D_LIBCPP_BUILD_STATIC)
endif()

# This is the _ONLY_ place where add_definitions is called.
if (MSVC)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif()

# Configure for sanitizers. If LIBCXX_BUILT_STANDALONE then we have to do
# the flag translation ourselves. Othewise LLVM's CMakeList.txt will handle it.
if (LIBCXX_BUILT_STANDALONE)
  # NOTE: LLVM_USE_SANITIZER checks for a UNIX like system instead of MSVC.
  # But we don't have LLVM_ON_UNIX so checking for MSVC is the best we can do.
  if (LLVM_USE_SANITIZER AND NOT MSVC)
    append_if(LIBCXX_CXX_FEATURE_FLAGS LIBCXX_HAS_NO_OMIT_FRAME_POINTER_FLAG
            "-fno-omit-frame-pointer")
    if (LLVM_USE_SANITIZER STREQUAL "Address")
      list(APPEND LIBCXX_CXX_FEATURE_FLAGS "-fsanitize=address")
    elseif (LLVM_USE_SANITIZER MATCHES "Memory(WithOrigins)?")
      list(APPEND LIBCXX_CXX_FEATURE_FLAGS "-fsanitize=memory")
      if (LLVM_USE_SANITIZER STREQUAL "MemoryWithOrigins")
        list(APPEND LIBCXX_CXX_FEATURE_FLAGS "-fsanitize-memory-track-origins")
      endif()
    else()
      message(WARNING "Unsupported value of LLVM_USE_SANITIZER: ${LLVM_USE_SANITIZER}")
    endif()
  elseif(MSVC)
    message(WARNING "LLVM_USE_SANITIZER is not supported with MSVC")
  endif()
endif()

string(REPLACE ";" " " LIBCXX_CXX_REQUIRED_FLAGS "${LIBCXX_CXX_REQUIRED_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${LIBCXX_CXX_REQUIRED_FLAGS}")

string(REPLACE ";" " " LIBCXX_CXX_WARNING_FLAGS "${LIBCXX_CXX_WARNING_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${LIBCXX_CXX_WARNING_FLAGS}")

string(REPLACE ";" " " LIBCXX_CXX_FEATURE_FLAGS "${LIBCXX_CXX_FEATURE_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${LIBCXX_CXX_FEATURE_FLAGS}")

#===============================================================================
# Setup Source Code
#===============================================================================

include_directories(include)
add_subdirectory(include)

# Add source code. This also contains all of the logic for deciding linker flags
# soname, etc...
add_subdirectory(lib)

#===============================================================================
# Setup Tests
#===============================================================================

add_subdirectory(test)
