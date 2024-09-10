# =================================================================================
# LLVM package definition and configuration
# =================================================================================

if(DEFINED LLVM_ROOT OR DEFINED ENV{LLVM_ROOT})
    if(NOT DEFINED LLVM_ROOT)
        set(LLVM_ROOT $ENV{LLVM_ROOT})
    endif()

    message(STATUS "Finding LLVM in custom LLVM root: ${LLVM_ROOT}")
    if(EXISTS "${LLVM_ROOT}/lib/cmake/llvm")
        list(APPEND CMAKE_PREFIX_PATH "${LLVM_ROOT}/lib/cmake/llvm/")
    else()
        list(APPEND CMAKE_PREFIX_PATH "${LLVM_ROOT}")
    endif()
endif()

# Find the LLVM instance to use for building SvfLLVM (makes add_llvm_library() available)
find_package(
    LLVM
    CONFIG
    REQUIRED
    HINTS
    ${LLVM_DIR}
    $ENV{LLVM_DIR})

# Add LLVM's include directories and link directory for all targets defined hereafter
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
# link_directories(${LLVM_LIBRARY_DIRS})
add_definitions(${LLVM_DEFINITIONS})

# Ensure SVF is built with RTTI if the used LLVM instance has them enabled
set(SVF_ENABLE_RTTI
    ${LLVM_ENABLE_RTTI}
    PARENT_SCOPE)

# If LLVM was built as a single shared library (libLLVM.so), link to that, otherwise to the static libraries
if(LLVM_LINK_LLVM_DYLIB)
    message(STATUS "Linking to LLVM dynamic shared library object")
    set(llvm_libs LLVM)
else()
    message(STATUS "Linking to separate LLVM static libraries")
    llvm_map_components_to_libnames(
        llvm_libs
        analysis
        bitwriter
        core
        instcombine
        instrumentation
        ipo
        irreader
        linker
        scalaropts
        support
        target
        transformutils)
endif()

# Find an instance of 'clang' in the found LLVM installation
find_program(
    LLVM_CLANG
    NAMES clang
    HINTS "${LLVM_BINARY_DIR}"
    PATH_SUFFIXES "bin" REQUIRED)

# # Make the "add_llvm_library()" command available and configure LLVM/CMake
# list(APPEND CMAKE_MODULE_PATH ${LLVM_CMAKE_DIR})
# include(AddLLVM)

# =================================================================================
# ExtAPI.bc generation & target definition
# =================================================================================
set(EXTAPI_SRC_FILE "${CMAKE_CURRENT_LIST_DIR}/lib/extapi.c")
set(EXTAPI_OUT_FILE "${SVF_BINARY_DIR}/lib/extapi.bc")

if(EXISTS "${EXTAPI_SRC_FILE}")
    message(STATUS "Found extapi.c input file at: ${EXTAPI_SRC_FILE}")
else()
    message(FATAL_ERROR "Failed to find extapi.c input file: ${EXTAPI_SRC_FILE}")
endif()

# Set the flags to compile extapi.c with
set(EXTAPI_COMPILE_FLAGS
    -w
    -S
    -c
    -fPIC
    -std=gnu11
    -emit-llvm
    -Xclang
    -disable-O0-optnone)

# If opaque pointers were disabled, add the appropriate flags
if(SVF_NO_OPAQUE_PTRS)
    message(STATUS "Building extapi.bc with opaque pointers disabled")
    list(APPEND SVF_EXTAPI_FLAGS -Xclang -no-opaque-pointers)
else()
    message(STATUS "Building extapi.bc with opaque pointers enabled")
endif()

# Add a custom command to compile the extapi.bc bitcode file from extapi.c
add_custom_command(
    OUTPUT "${EXTAPI_OUT_FILE}"
    COMMAND "${LLVM_CLANG}" ${EXTAPI_COMPILE_FLAGS} -o "${EXTAPI_OUT_FILE}" "${EXTAPI_SRC_FILE}"
    DEPENDS "${EXTAPI_SRC_FILE}")

# Add a custom target that has the custom command as its creation command
add_custom_target(gen_extapi_ir DEPENDS "${EXTAPI_OUT_FILE}")

# Install the bitcode file as well; install it to <prefix>/include/SVF/extapi.bc
install(FILES "${EXTAPI_OUT_FILE}" DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/SVF)

# =================================================================================
# Main SVF LLVM library definition
# =================================================================================

# Gather all header and source files
file(GLOB_RECURSE SVF_LLVM_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/include/*.h)
file(GLOB_RECURSE SVF_LLVM_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/lib/*.cpp)

# Define the actual SvfLLVM library & set the (so)versions for dynamic library exports
add_library(SvfLLVM)
add_library(SVF::SvfLLVM ALIAS SvfLLVM)
set_target_properties(SvfLLVM PROPERTIES VERSION ${SVF_VERSION} SOVERSION ${SVF_VERSION_MAJOR})

# Add the source files (private)
target_sources(SvfLLVM PRIVATE ${SVF_LLVM_SOURCES})

# Set the header files as public (so they're exported & installed)
target_sources(
    SvfLLVM
    PUBLIC FILE_SET
           HEADERS
           BASE_DIRS
           include
           FILES
           ${SVF_LLVM_HEADERS})

# Add the build include tree privately & the local & installed include directories depending on the interface
target_include_directories(
    SvfLLVM
    PRIVATE $<BUILD_INTERFACE:${SVF_BINARY_DIR}/include>
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    PUBLIC $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

# Link Z3 privately to SVF (users of SVF don't directly use its Z3 instance)
target_link_libraries(SvfLLVM PRIVATE ${Z3_IMPORTED_TARGET})

# Also add Z3 as private system include directories
target_include_directories(SvfLLVM SYSTEM PRIVATE ${Z3_INCLUDES})

# # Ensure that the build artifacts are placed into lib/bin/include relative to the top of the build directory
set_target_properties(
    SvfLLVM
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${SVF_BINARY_DIR}/bin
               LIBRARY_OUTPUT_DIRECTORY ${SVF_BINARY_DIR}/lib
               ARCHIVE_OUTPUT_DIRECTORY ${SVF_BINARY_DIR}/lib)

# Add a dependency of SvfLLVM to the extapi.bc file
add_dependencies(SvfLLVM gen_extapi_ir)

# Link the SVF Core library to this library
target_link_libraries(SvfLLVM PRIVATE SvfCore)

# Install the different components of the SVF LLVM library
install(
    TARGETS SvfLLVM
    EXPORT SVF_EXPORTED_TARGETS
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            FILE_SET HEADERS
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/SVF)

# =================================================================================
# Tools
# =================================================================================
add_subdirectory(tools)
