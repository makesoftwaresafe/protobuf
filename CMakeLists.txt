# Minimum CMake required. If available, accept the policy-controlled behavior up
# to 3.26.
cmake_minimum_required(VERSION 3.16...3.26)

# Revert to old behavior for MSVC debug symbols.
if(POLICY CMP0141)
  cmake_policy(SET CMP0141 OLD)
endif()

if(protobuf_VERBOSE)
  message(STATUS "Protocol Buffers Configuring...")
endif()

# Project
project(protobuf C CXX)

if(CMAKE_CXX_STANDARD AND CMAKE_CXX_STANDARD LESS 17)
  message(FATAL_ERROR "The minimum supported C++ standard is C++ 17")
endif()

if(protobuf_DEPRECATED_CMAKE_SUBDIRECTORY_USAGE)
  if(CMAKE_PROJECT_NAME STREQUAL "protobuf")
    get_filename_component(CMAKE_SOURCE_DIR ${CMAKE_SOURCE_DIR} DIRECTORY)
  endif()
  get_filename_component(CMAKE_CURRENT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR} DIRECTORY)
  get_filename_component(PROJECT_SOURCE_DIR ${PROJECT_SOURCE_DIR} DIRECTORY)
  get_filename_component(protobuf_SOURCE_DIR ${protobuf_SOURCE_DIR} DIRECTORY)
endif()

# Options
option(protobuf_INSTALL "Install protobuf binaries and files" ON)
option(protobuf_BUILD_TESTS "Build tests" ON)
option(protobuf_BUILD_CONFORMANCE "Build conformance tests" OFF)
option(protobuf_BUILD_EXAMPLES "Build examples" OFF)
option(protobuf_BUILD_PROTOBUF_BINARIES "Build protobuf libraries and protoc compiler" ON)
option(protobuf_BUILD_PROTOC_BINARIES "Build libprotoc and protoc compiler" ON)
option(protobuf_BUILD_LIBPROTOC "Build libprotoc" OFF)
option(protobuf_BUILD_LIBUPB "Build libupb" ON)
option(protobuf_DISABLE_RTTI "Remove runtime type information in the binaries" OFF)
option(protobuf_TEST_XML_OUTDIR "Output directory for XML logs from tests." "")
option(protobuf_ALLOW_CCACHE "Adjust build flags to allow for ccache support." OFF)
option(protobuf_FORCE_FETCH_DEPENDENCIES "Force all dependencies to be downloaded from GitHub.  Local installations will be ignored." OFF)
option(protobuf_LOCAL_DEPENDENCIES_ONLY "Prevent downloading any dependencies from GitHub. If this option is set, the dependency must be available locally as an installed package." OFF)

# We support Unity (Jumbo) builds best-effort.
option(protobuf_USE_UNITY_BUILD "Enable Unity (Jumbo) build for" OFF)
if (BUILD_SHARED_LIBS)
  set(protobuf_BUILD_SHARED_LIBS_DEFAULT ON)
else (BUILD_SHARED_LIBS)
  set(protobuf_BUILD_SHARED_LIBS_DEFAULT OFF)
endif (BUILD_SHARED_LIBS)
option(protobuf_BUILD_SHARED_LIBS "Build Shared Libraries" ${protobuf_BUILD_SHARED_LIBS_DEFAULT})
include(CMakeDependentOption)
cmake_dependent_option(protobuf_MSVC_STATIC_RUNTIME "Link static runtime libraries" ON
  "NOT protobuf_BUILD_SHARED_LIBS" OFF)
set(protobuf_WITH_ZLIB_DEFAULT ON)
option(protobuf_WITH_ZLIB "Build with zlib support" ${protobuf_WITH_ZLIB_DEFAULT})
set(protobuf_DEBUG_POSTFIX "d"
  CACHE STRING "Default debug postfix")
mark_as_advanced(protobuf_DEBUG_POSTFIX)

if(WITH_PROTOC)
  set(protobuf_PROTOC_EXE protoc)
  set(protobuf_BUILD_PROTOC_BINARIES OFF)
  add_executable(protoc IMPORTED GLOBAL)
  add_executable(protobuf::protoc ALIAS protoc)
  set_property(TARGET protoc PROPERTY IMPORTED_LOCATION ${WITH_PROTOC})
endif()

# User options
include(${protobuf_SOURCE_DIR}/cmake/protobuf-options.cmake)

if (protobuf_BUILD_SHARED_LIBS)
  # This is necessary for linking in Abseil.
  set(CMAKE_POSITION_INDEPENDENT_CODE ON)

  # Build Abseil as shared libraries to avoid ODR violations.
  set(BUILD_SHARED_LIBS ON)

  # Output directory is correct by default for most build setups. However, when
  # building Protobuf as a DLL, it is important to have the DLL in the same
  # directory as the executable using it. Thus, we put all binaries in a single
  # /bin directory.
  if (MSVC)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
    set(CMAKE_PDB_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
  endif ()
endif ()

# Version metadata
set(protobuf_VERSION_STRING "6.33.0")
set(protobuf_DESCRIPTION "Protocol Buffers")
set(protobuf_CONTACT "protobuf@googlegroups.com")

# Overrides for option dependencies
if (protobuf_BUILD_PROTOC_BINARIES OR protobuf_BUILD_TESTS)
  set(protobuf_BUILD_LIBPROTOC ON)
endif ()
if (NOT protobuf_BUILD_PROTOBUF_BINARIES)
  set(protobuf_INSTALL OFF)
endif()
# Parse version tweaks
set(protobuf_VERSION_REGEX "^([0-9]+)\\.([0-9]+)\\.([0-9]+)([-]rc[-]|\\.)?([0-9]*)$")
string(REGEX REPLACE     "${protobuf_VERSION_REGEX}" "\\1"
  protobuf_VERSION_MAJOR "${protobuf_VERSION_STRING}")
string(REGEX REPLACE     "${protobuf_VERSION_REGEX}" "\\2"
  protobuf_VERSION_MINOR "${protobuf_VERSION_STRING}")
string(REGEX REPLACE     "${protobuf_VERSION_REGEX}" "\\3"
  protobuf_VERSION_PATCH "${protobuf_VERSION_STRING}")
string(REGEX REPLACE     "${protobuf_VERSION_REGEX}" "\\5"
  protobuf_VERSION_PRERELEASE "${protobuf_VERSION_STRING}")

if (protobuf_FORCE_FETCH_DEPENDENCIES AND protobuf_LOCAL_DEPENDENCIES_ONLY)
  message(FATAL_ERROR "Conflicting options protobuf_FORCE_FETCH_DEPENDENCIES and protobuf_LOCAL_DEPENDENCIES_ONLY both set")
endif()

# Package version
set(protobuf_VERSION
  "${protobuf_VERSION_MINOR}.${protobuf_VERSION_PATCH}")

if(protobuf_VERSION_PRERELEASE)
  message(STATUS "${protobuf_VERSION_PRERELEASE}")
  set(protobuf_VERSION "${protobuf_VERSION}.${protobuf_VERSION_PRERELEASE}")
else()
  set(protobuf_VERSION "${protobuf_VERSION}.0")
endif()
message(STATUS "protobuf version: ${protobuf_VERSION}")

if(protobuf_VERBOSE)
  message(STATUS "Configuration script parsing status [")
  message(STATUS "  Description : ${protobuf_DESCRIPTION}")
  message(STATUS "  Version     : ${protobuf_VERSION} (${protobuf_VERSION_STRING})")
  message(STATUS "  Contact     : ${protobuf_CONTACT}")
  message(STATUS "]")
endif()

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/cmaketest.map
"{
  global:
    main;
  local:
    *;
};")
# CheckLinkerFlag module available in CMake >=3.18.
if(${CMAKE_VERSION} VERSION_GREATER 3.18 OR ${CMAKE_VERSION} VERSION_EQUAL 3.18)
  include(CheckLinkerFlag)
  check_linker_flag(CXX -Wl,--version-script=${CMAKE_CURRENT_BINARY_DIR}/cmaketest.map protobuf_HAVE_LD_VERSION_SCRIPT)
else()
  include(CheckCXXSourceCompiles)
  set(OLD_CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS})
  set(CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS} -Wl,--version-script=${CMAKE_CURRENT_BINARY_DIR}/cmaketest.map)
  check_cxx_source_compiles("
    int main() {
      return 0;
    }
  " protobuf_HAVE_LD_VERSION_SCRIPT)
  set(CMAKE_REQUIRED_FLAGS ${OLD_CMAKE_REQUIRED_FLAGS})
endif()
file(REMOVE ${CMAKE_CURRENT_BINARY_DIR}/cmaketest.map)

find_package(Threads REQUIRED)

set(_protobuf_FIND_ZLIB)
if (protobuf_WITH_ZLIB)
  find_package(ZLIB)
  if (ZLIB_FOUND)
    set(HAVE_ZLIB 1)
    # FindZLIB module define ZLIB_INCLUDE_DIRS variable
    # Set ZLIB_INCLUDE_DIRECTORIES for compatible
    set(ZLIB_INCLUDE_DIRECTORIES ${ZLIB_INCLUDE_DIRECTORIES} ${ZLIB_INCLUDE_DIRS})
    # Using imported target if exists
    if (TARGET ZLIB::ZLIB)
      set(ZLIB_LIBRARIES ZLIB::ZLIB)
      set(_protobuf_FIND_ZLIB "if(NOT ZLIB_FOUND)\n  find_package(ZLIB)\nendif()")
    endif (TARGET ZLIB::ZLIB)
  else (ZLIB_FOUND)
    set(HAVE_ZLIB 0)
    # Explicitly set these to empty (override NOT_FOUND) so cmake doesn't
    # complain when we use them later.
    set(ZLIB_INCLUDE_DIRECTORIES)
    set(ZLIB_LIBRARIES)
  endif (ZLIB_FOUND)
endif (protobuf_WITH_ZLIB)

# We need to link with libatomic on systems that do not have builtin atomics, or
# don't have builtin support for 8 byte atomics (e.g. 32-bit powerpc).
set(protobuf_LINK_LIBATOMIC false)
if (NOT MSVC)
  include(CheckCXXSourceCompiles)
  check_cxx_source_compiles("
    #include <atomic>
    int main() {
      return static_cast<int>(std::atomic<int64_t>{});
    }
  " protobuf_HAVE_BUILTIN_ATOMICS)
  if (NOT protobuf_HAVE_BUILTIN_ATOMICS)
    set(protobuf_LINK_LIBATOMIC true)
  endif (NOT protobuf_HAVE_BUILTIN_ATOMICS)
endif (NOT MSVC)

if (protobuf_BUILD_SHARED_LIBS)
  set(protobuf_SHARED_OR_STATIC "SHARED")
else (protobuf_BUILD_SHARED_LIBS)
  set(protobuf_SHARED_OR_STATIC "STATIC")
  set(ABSL_MSVC_STATIC_RUNTIME ${protobuf_MSVC_STATIC_RUNTIME})
  if (protobuf_MSVC_STATIC_RUNTIME)
      set(CMAKE_MSVC_RUNTIME_LIBRARY MultiThreaded$<$<CONFIG:Debug>:Debug>)
  else()
      set(CMAKE_MSVC_RUNTIME_LIBRARY MultiThreaded$<$<CONFIG:Debug>:Debug>DLL)
  endif()
endif (protobuf_BUILD_SHARED_LIBS)

# Export all symbols on Windows when building shared libraries
SET(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

if (MSVC)
  string(REPLACE "/" "\\" PROTOBUF_SOURCE_WIN32_PATH ${protobuf_SOURCE_DIR})
  string(REPLACE "/" "\\" PROTOBUF_BINARY_WIN32_PATH ${protobuf_BINARY_DIR})
  string(REPLACE "." ","  protobuf_RC_FILEVERSION "${protobuf_VERSION}")

  if (protobuf_ALLOW_CCACHE)
    # In order to support ccache, we need to remove the /Zi option because it
    # puts debug symbols into separate pdb files (which in incompatible with
    # ccache).  This can be replaced with /Z7 to preserve debug symbols, which
    # embeds debug symbols into the object files instead of creating a separate
    # pdb file, which isn't currently supported by ccache.  However, this bloats
    # the ccache size by about a factor of 2x, making it very expensive in CI.
    # Instead, we strip debug symbols to reduce this overhead.
    foreach(v
        CMAKE_C_FLAGS_DEBUG
        CMAKE_CXX_FLAGS_DEBUG
        CMAKE_C_FLAGS_RELWITHDEBINFO
        CMAKE_CXX_FLAGS_RELWITHDEBINFO
        )
      string(REGEX REPLACE "[-/]Z[iI7]" "/DEBUG:NONE" ${v} "${${v}}")
    endforeach()
  endif()

  # Suppress linker warnings about files with no symbols defined.
  string(APPEND CMAKE_STATIC_LINKER_FLAGS " /ignore:4221")

  # use English language (0x409) in resource compiler
  string(APPEND CMAKE_RC_FLAGS " -l0x409")

  # Generate the version.rc file used elsewhere.
  configure_file(${protobuf_SOURCE_DIR}/cmake/version.rc.in ${CMAKE_CURRENT_BINARY_DIR}/version.rc @ONLY)
  set(protobuf_version_rc_file ${CMAKE_CURRENT_BINARY_DIR}/version.rc)

  # Add the "lib" prefix for generated .lib outputs.
  set(LIB_PREFIX lib)
else (MSVC)
  # No version.rc file.
  set(protobuf_version_rc_file)

  # When building with "make", "lib" prefix will be added automatically by
  # the build tool.
  set(LIB_PREFIX)
endif (MSVC)

include_directories(
  ${ZLIB_INCLUDE_DIRECTORIES}
  ${protobuf_BINARY_DIR}
  # Support #include-ing other top-level directories, i.e. upb_generator.
  ${protobuf_SOURCE_DIR}
  ${protobuf_BINARY_DIR}/src
  ${protobuf_SOURCE_DIR}/src)

if (protobuf_BUILD_TESTS)
  include(${protobuf_SOURCE_DIR}/cmake/gtest.cmake)
endif (protobuf_BUILD_TESTS)

include(${protobuf_SOURCE_DIR}/cmake/abseil-cpp.cmake)

if (protobuf_BUILD_PROTOBUF_BINARIES)
  include(${protobuf_SOURCE_DIR}/cmake/utf8_range.cmake)
  include(${protobuf_SOURCE_DIR}/cmake/libprotobuf-lite.cmake)
  if (NOT DEFINED protobuf_LIB_PROTOBUF_LITE)
    set(protobuf_LIB_PROTOBUF_LITE libprotobuf-lite)
  endif ()
  include(${protobuf_SOURCE_DIR}/cmake/libprotobuf.cmake)
  if (NOT DEFINED protobuf_LIB_PROTOBUF)
    set(protobuf_LIB_PROTOBUF libprotobuf)
  endif ()
  if (protobuf_BUILD_LIBPROTOC)
    include(${protobuf_SOURCE_DIR}/cmake/libprotoc.cmake)
    if (NOT DEFINED protobuf_LIB_PROTOC)
      set(protobuf_LIB_PROTOC libprotoc)
    endif ()
  endif ()
  if (protobuf_BUILD_LIBUPB)
    include(${protobuf_SOURCE_DIR}/cmake/libupb.cmake)
    if (NOT DEFINED protobuf_LIB_UPB)
      set(protobuf_LIB_UPB libupb)
    endif ()
    include(${protobuf_SOURCE_DIR}/cmake/upb_generators.cmake)
  endif ()
  if (protobuf_BUILD_PROTOC_BINARIES)
    include(${protobuf_SOURCE_DIR}/cmake/protoc.cmake)
    if (NOT DEFINED protobuf_PROTOC_EXE)
      set(protobuf_PROTOC_EXE protoc)
    endif ()
  endif ()
else ()
  find_package(Protobuf NO_MODULE)
  if (Protobuf_FOUND)
    set(protobuf_PROTOC_EXE protobuf::protoc)
    set(protobuf_LIB_PROTOC protobuf::libprotoc)
    set(protobuf_LIB_PROTOBUF protobuf::libprotobuf)
    set(protobuf_LIB_PROTOBUF_LITE protobuf::libprotobuf-lite)
    set(protobuf_LIB_UPB protobuf::libupb)
    message(STATUS "CMake installation of Protobuf found.")
  endif ()
endif ()

# Ensure we have a protoc executable and protobuf libraries if we need one
if (protobuf_BUILD_TESTS OR protobuf_BUILD_CONFORMANCE OR protobuf_BUILD_EXAMPLES)
  if (NOT DEFINED protobuf_PROTOC_EXE)
    find_program(protobuf_PROTOC_EXE protoc REQUIRED)
    message(STATUS "Found system ${protobuf_PROTOC_EXE}.")
  endif ()
  if(protobuf_VERBOSE)
    message(STATUS "Using protoc : ${protobuf_PROTOC_EXE}")
    message(STATUS "Using libprotobuf : ${protobuf_LIB_PROTOBUF}")
    message(STATUS "Using libprotobuf-lite : ${protobuf_LIB_PROTOBUF_LITE}")
    message(STATUS "Using libprotoc : ${protobuf_LIB_PROTOC}")
    message(STATUS "Using libupb : ${protobuf_LIB_UPB}")
  endif(protobuf_VERBOSE)
endif ()

if (protobuf_BUILD_TESTS)
  enable_testing()
  include(${protobuf_SOURCE_DIR}/cmake/tests.cmake)
endif (protobuf_BUILD_TESTS)

if (protobuf_BUILD_CONFORMANCE)
  include(${protobuf_SOURCE_DIR}/cmake/conformance.cmake)
endif (protobuf_BUILD_CONFORMANCE)

if (protobuf_INSTALL)
  include(${protobuf_SOURCE_DIR}/cmake/install.cmake)
endif (protobuf_INSTALL)

if (protobuf_BUILD_EXAMPLES)
  include(${protobuf_SOURCE_DIR}/cmake/examples.cmake)
endif (protobuf_BUILD_EXAMPLES)

if(protobuf_VERBOSE)
  message(STATUS "Protocol Buffers Configuring done")
endif(protobuf_VERBOSE)
