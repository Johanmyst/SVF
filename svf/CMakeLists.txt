# Gather all header and source files
file(GLOB_RECURSE SVF_CORE_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/include/*.h)
file(GLOB_RECURSE SVF_CORE_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/lib/*.cpp)

# Define the core library itself & set the (so)version for dynamic library exports
add_library(SvfCore)
add_library(SVF::SvfCore ALIAS SvfCore)
set_target_properties(SvfCore PROPERTIES VERSION ${SVF_VERSION} SOVERSION ${SVF_VERSION_MAJOR})

# Add the source files (private)
target_sources(SvfCore PRIVATE ${SVF_CORE_SOURCES})

# Set the header files as public (so they're exported & installed)
target_sources(
    SvfCore
    PUBLIC FILE_SET
           HEADERS
           BASE_DIRS
           include
           FILES
           ${SVF_CORE_HEADERS})

# Add the include directories (for build & install time); also add the current build dir as an include dir
target_include_directories(
    SvfCore
    PRIVATE $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    PUBLIC $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

# Link Z3 privately to SVF (users of SVF don't directly use its Z3 instance)
target_link_libraries(SvfCore PRIVATE ${Z3_IMPORTED_TARGET})

# Also add Z3 as private system include directories
target_include_directories(SvfCore SYSTEM PRIVATE ${Z3_INCLUDES})

# # Ensure that the build artifacts are placed into lib/bin/include relative to the top of the build directory
set_target_properties(
    SvfCore
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin
               LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
               ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

# Install the different components of the SVF Core library
install(
    TARGETS SvfCore
    EXPORT SVF_EXPORTED_TARGETS
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            FILE_SET HEADERS
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/SVF)
