include_directories (${CMAKE_CURRENT_BINARY_DIR})

# GSL dependency
find_package(GSL REQUIRED)

set(_casa_code_path "${CMAKE_SOURCE_DIR}/casa6/casatools/src/code")
set(_all_sources)
set(_all_headers)

foreach (_subdir imageanalysis casatools components stdcasa)
    set(_path "${_casa_code_path}/${_subdir}")

    # List the files that are going to be compiled. It is a recursive search
    file(GLOB_RECURSE ${_subdir}_library_sources "${_path}/*.cc" "${_path}/*.c")

    # Ignore files under test directories
    list(FILTER ${_subdir}_library_sources EXCLUDE REGEX "/test/")
  
    # List the header files and template header files, also recursive
    file(GLOB_RECURSE ${_subdir}_public_headers CONFIGURE_DEPENDS "${_path}/*.h" "${_path}/*.tcc" "${_path}/*.hpp")
    
    set(_all_sources ${_all_sources} ${${_subdir}_library_sources})
    set(_all_headers ${_all_headers} ${${_subdir}_public_headers})
    
    # Install the header and template header files
    # Note that this preserves the directory structure of the files
    # That's why ${_subdir}_public_headers is not reused
    install(DIRECTORY ${_path}
        DESTINATION include/casacode/
        FILES_MATCHING
        REGEX "/.*(h|tcc|hpp)$"
        PATTERN "test" EXCLUDE
    )

endforeach (_subdir)

add_library (casa_imageanalysis)

# Custom SOVERSION for imageanalysis is derived from the casa minor version rather than the casacore minor version.
# Base CASA uses different values, but we should continue using our convention for consistency.
# Check git tags for CASA version (currently 6.7.5)
# If the minor version has changed, bump this:
set(CASA_PROJECT_SOVERSION 7)

set_target_properties(
    casa_imageanalysis
    PROPERTIES
        VERSION "${CASA_PROJECT_SOVERSION}"
        SOVERSION "${CASA_PROJECT_SOVERSION}"
)

target_sources(casa_imageanalysis
    PRIVATE 
    ${_all_sources}
    PUBLIC 
    ${_all_headers})

target_link_libraries (casa_imageanalysis
    casa_images
    casa_coordinates
    casa_mirlib
    casa_lattices
    ${CASACORE_ARCH_LIBS}
    ${GSL_LIBRARIES}
)

install (
    TARGETS casa_imageanalysis
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib${LIB_SUFFIX}
    ARCHIVE DESTINATION lib${LIB_SUFFIX}
    LIBRARY PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
)
