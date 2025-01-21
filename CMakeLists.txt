cmake_minimum_required(VERSION 3.23)

project(
    SVF
    VERSION 3.1
    DESCRIPTION "SVF is a static value-flow analysis tool for LLVM-based languages"
    HOMEPAGE_URL "https://github.com/SVF-tools/SVF"
    LANGUAGES C CXX)

# =================================================================================
# Base configuration
# =================================================================================

# Ensure installation directories like ${CMAKE_INSTALL_LIBDIR} are available
include(FetchContent)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Build SVF with C++ standard C++17 and the C standard C11 (with GNU extensions (required))
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_EXTENSIONS ON)
set(CMAKE_CXX_EXTENSIONS ON)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# # Ensure the RPATH is used for the build tree
# set(CMAKE_SKIP_BUILD_RPATH OFF)

# # Don't use the RPATH during building; only when installing
# set(CMAKE_BUILD_WITH_INSTALL_RPATH OFF)
# set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")

# # Add paths found by linker to RPATH so they're included at install time
# set(CMAKE_INSTALL_RPATH_USE_LINK_PATH ON)

# Use LLVM's ld.lld linker
add_link_options("-fuse-ld=lld")

# =================================================================================
# SVF options & settings
# =================================================================================

# Configurable (string) options for building SVF
set(SVF_SANITIZE
    ""
    CACHE STRING "Create sanitizer build (address)")

# Set whether or not SVF was built as a shared library
set(SVF_SHARED_LIBS ${BUILD_SHARED_LIBS})

# Configurable (boolean) options for building SVF
option(SVF_COVERAGE "Create coverage build (default: off)")
option(SVF_ASSERTIONS "Enable assertion statements (default: off)")
option(SVF_DEBUG_INFO "Build with debug symbols enabled (default: on)" ON)
option(SVF_WARN_AS_ERROR "Treat warnings as errors during compilation (default: on)" ON)
option(SVF_EXPORT_DYNAMIC "Export all (also unused) symbols to symbol table (default: on)" ON)
option(SVF_NO_OPAQUE_PTRS "Disable opaque pointers when build extapi.bc (use typed-pointers) (default: off)")

message(STATUS "Using build configuration:
    CMake Configuration:
        Generator:                              ${CMAKE_GENERATOR}
        C compiler:                             ${CMAKE_C_COMPILER_ID}
        C++ compiler:                           ${CMAKE_CXX_COMPILER_ID}
        Initial C flags:                        ${CMAKE_C_FLAGS}
        Initial C++ flags:                      ${CMAKE_CXX_FLAGS}
        Main source directory:                  ${SVF_SOURCE_DIR}
        Main binary directory:                  ${SVF_BINARY_DIR}
    SVF Configuration:
        SVF version:                            ${SVF_VERSION}
        SVF option - sanitiser:                 ${SVF_SANITIZE}
        SVF option - coverage build:            ${SVF_COVERAGE}
        SVF option - enable assertions:         ${SVF_ASSERTIONS}
        SVF option - warnings as errors:        ${SVF_WARN_AS_ERROR}
        SVF option - unused dynamic symbols:    ${SVF_EXPORT_DYNAMIC}
        SVF option - disable opaque pointers:   ${SVF_NO_OPAQUE_PTRS}")

# =================================================================================
# Build configuration/settings
# =================================================================================

# By default, build SVF and its targets treating all compiler warnings as errors (except deprecations)
if(SVF_WARN_AS_ERROR)
    message(STATUS "Treating compiler warnings as errors")
    add_compile_options(-Wall -Werror)
else()
    message(STATUS "Not treating compiler warnings as errors")
endif()

# Keep assertions enabled if requested (explicitly add -UNDEBUG to undefine NDEBUG because build mode can set it)
if(SVF_ASSERTIONS)
    message(STATUS "Enabling assertion statements in compiled code")
    add_compile_definitions(UNDEBUG)

    # If this is a release build, CMake automatically adds -DNDEBUG, so remove it manually
    string(
        REGEX
        REPLACE "\-?DNDEBUG"
                ""
                CMAKE_C_FLAGS_RELEASE
                "${CMAKE_C_FLAGS_RELEASE}")
    string(
        REGEX
        REPLACE "\-?DNDEBUG"
                ""
                CMAKE_CXX_FLAGS_RELEASE
                "${CMAKE_CXX_FLAGS_RELEASE}")
    string(
        REGEX
        REPLACE "\-?DNDEBUG"
                ""
                CMAKE_C_FLAGS_RELWITHDEBINFO
                "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
    string(
        REGEX
        REPLACE "\-?DNDEBUG"
                ""
                CMAKE_CXX_FLAGS_RELWITHDEBINFO
                "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
else()
    message(STATUS "Disabling assertions statements in compiled code")
    add_compile_definitions(NDEBUG)
endif()

# Export dynamic symbols if requested (adds "-export-dynamic" to linkers that support it to enable backtraces)
if(SVF_EXPORT_DYNAMIC)
    message(STATUS "Exporting all symbols (also unused) to dynamic symbol table")
    add_link_options(-rdynamic -Wl,--export-dynamic)
else()
    message(STATUS "Only exporting used symbols to dynamic symbol table")
endif()

# Configure whether a coverage build should be created for SVF (i.e. add runtime instrumentation)
if(SVF_COVERAGE)
    message(STATUS "Building SVF with coverage instrumentation")
    add_compile_options(-fprofile-arcs -ftest-coverage)
    add_link_options(-fprofile-arcs -ftest-coverage)
else()
    message(STATUS "Building non-coverage build")
endif()

# If building with sanitiser, load the given sanitiser mode
if(SVF_SANITIZE STREQUAL "address")
    message(STATUS "Creating sanitiser build with AddressSanitiser")
    add_compile_options(-fno-omit-frame-pointer -fsanitize=address)
    add_link_options(-fsanitize=address)
elseif(SVF_SANITIZE STREQUAL "thread")
    message(STATUS "Creating sanitiser build with ThreadSanitiser")
    add_compile_options(-fsanitize=thread)
    add_link_options(-fsanitize=thread)
elseif(
    NOT
    SVF_SANITIZE
    STREQUAL
    "")
    message(FATAL_ERROR "Unrecognised sanitiser mode: ${SVF_SANITIZE}")
else()
    message(STATUS "Not building with sanitiser")
endif()

# If enabled, output debugging information
if(SVF_DEBUG_INFO)
    message(STATUS "Enabling debugging information in SVF build")
    add_compile_options(-g3)
    add_link_options(-g3)
else()
    message(STATUS "Not enabling debugging information in SVF build")
endif()

# Disable the numerous deprecation warnings
add_compile_options(-Wno-deprecated-declarations)

# =================================================================================
# Find & load external Z3 dependencies
# =================================================================================

# Try finding a system-wide (or local instance with $Z3_DIR) Z3's CMake package to import directly
find_package(
    Z3
    CONFIG
    HINTS
    ${Z3_DIR}
    $ENV{Z3_DIR})

if(Z3_FOUND)
    message(STATUS "Found Z3 CMake package (version: ${Z3_VERSION_STRING})")

    # Use the imported target from the Z3 CMake package
    set(Z3_IMPORTED_TARGET z3::libz3)
else()
    message(STATUS "Failed to find Z3 CMake package; searching for library & header files")

    # Explicitly try to find libz3.so/libz3.a and the header files
    if(SVF_SHARED_LIBS)
        find_library(Z3_LIBRARIES libz3.so HINTS ${Z3_DIR} ENV Z3_DIR)
    else()
        find_library(Z3_LIBRARIES libz3.a HINTS ${Z3_DIR} ENV Z3_DIR)
    endif()

    # Try to find any (even different static/shared mode) instances of Z3 as a fall-back
    if(NOT Z3_LIBRARIES)
        find_library(
            Z3_LIBRARIES z3
            HINTS ${Z3_DIR}
                  ENV
                  Z3_DIR
                  REQUIRED)
    endif()

    # Also find Z3's header files
    find_path(
        Z3_INCLUDES z3++.h
        HINTS ${Z3_DIR}
              ENV
              Z3_DIR
              REQUIRED)

    # If found, manually create a Z3 package to nicely export the location(s)
    message(STATUS "Found Z3 libraries (${Z3_LIBRARIES}) and headers (${Z3_INCLUDES})")
    add_library(z3::libz3 UNKNOWN IMPORTED)
    set_target_properties(z3::libz3 PROPERTIES IMPORTED_LOCATION "${Z3_LIBRARIES}")
    set_target_properties(z3::libz3 PROPERTIES INTERFACE_INCLUDE_DIRECTORIES "${Z3_INCLUDES}")

    # Set the imported target
    set(Z3_IMPORTED_TARGET z3::libz3)
endif()

# =================================================================================
# Export C/C++ config.h header file
# =================================================================================

# Create config.h based on config.in; export to <build_dir>/include/Util/config.h
file(MAKE_DIRECTORY "${SVF_BINARY_DIR}/include")
configure_file("${SVF_SOURCE_DIR}/cmake/ConfigHdr.cmake.in" "${SVF_BINARY_DIR}/include/Util/config.h")

# =================================================================================
# Test suite configuration & build
# =================================================================================

# Check if the test-suite is present, if it is then build bc files and add testing to cmake build
if(EXISTS "${SVF_SOURCE_DIR}/Test-Suite")
    message(STATUS "Testing subdirectory found; enabling tests")
    include_directories("${SVF_SOURCE_DIR}/Test-Suite")
    enable_testing()
    add_subdirectory(Test-Suite)
    include(CTest)
else()
    message(STATUS "No testing subdirectory found; not enabling SVF tests")
endif()

# =================================================================================
# Add the SVF core subdirectory
# =================================================================================
add_subdirectory(svf)

# =================================================================================
# Add the SVF LLVM module subdirectory
# =================================================================================
add_subdirectory(svf-llvm)

# =================================================================================
# Include core SVF source trees
# =================================================================================

# If LLVM was built without RTTI, disable it for SVF as well to avoid issues
if(NOT ${SVF_ENABLE_RTTI})
    message(STATUS "LLVM was built without RTTI support; disabling RTTI")
    add_compile_options(-fno-rtti)
endif()

# =================================================================================
# Installation & package configuration
# =================================================================================

# Export the targets so that SvfCore & SvfLLVM are available after find_package(SVF) also in the build tree
export(
    EXPORT SVF_EXPORTED_TARGETS
    NAMESPACE SVF::
    FILE ${SVF_BINARY_DIR}/SVFTargets.cmake)

# Generate SVFConfigVersion.cmake
write_basic_package_version_file(
    ${SVF_BINARY_DIR}/SVFConfigVersion.cmake
    VERSION "${SVF_VERSION_MAJOR}.${SVF_VERSION_MINOR}"
    COMPATIBILITY AnyNewerVersion)

# Generate the package config
configure_file(${SVF_SOURCE_DIR}/cmake/SVF.pc.cmake.in ${SVF_BINARY_DIR}/SVF.pc @ONLY)

# Install the SVFTargets.cmake file along with the package
install(
    EXPORT SVF_EXPORTED_TARGETS
    FILE SVFTargets.cmake
    NAMESPACE SVF::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/SVF)

# Also generate a relocatable package in the install tree
configure_package_config_file(${SVF_SOURCE_DIR}/cmake/SVFConfig.cmake.in ${SVF_BINARY_DIR}/SVFConfig.cmake
                              INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/SVF)

# Actually install the configuration, versioning, and pkg_config files
install(FILES ${SVF_BINARY_DIR}/SVFConfigVersion.cmake DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/SVF)
install(FILES ${SVF_BINARY_DIR}/SVFConfig.cmake DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/SVF)
install(FILES ${SVF_BINARY_DIR}/SVF.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

# Install generated configuration header (see `configure_file()`) to top-level include dir of SVF
install(FILES ${SVF_BINARY_DIR}/include/Util/config.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/SVF/Util)

# Since SVF builds into non-standard directories; symlink the compile commands
file(
    CREATE_LINK
    ${SVF_BINARY_DIR}/compile_commands.json
    ${SVF_SOURCE_DIR}/compile_commands.json
    COPY_ON_ERROR
    SYMBOLIC)
