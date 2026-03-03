cmake_minimum_required (VERSION 3.14.0)

project(casacore VERSION 3.8.0)
set(_casacore_dir "casa6/casatools/casacore")
set(_casacore_source_dir "${CMAKE_SOURCE_DIR}/${_casacore_dir}")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/${_casacore_dir}/cmake")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

include_directories( BEFORE ${CMAKE_CURRENT_BINARY_DIR} "${CMAKE_CURRENT_LIST_DIR}/${_casacore_dir}" "${CMAKE_CURRENT_LIST_DIR}/casa6/casatools/src/code")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")

include(CheckCXXCompilerFlag)
include(CheckCCompilerFlag)
include(CheckFunctionExists)
include(CheckCXXSourceCompiles)
include(${_casacore_dir}/cmake/PCHSupport.cmake)

# Change ~/.casarc to ~/.carta-casacore/casarc to facilitate updating the CASA measures data in CARTA.
add_custom_target(casarc_patch ALL
    COMMAND sed -i.orig 's|/.casarc:|/.carta-casacore/casarc:|g' ${_casacore_source_dir}/casa/System/Aipsrc.cc
)

# Silence internal deprecation warnings until https://github.com/casacore/casacore/issues/1460 is fixed
add_custom_target(pi_patch ALL
    COMMAND sed -i.orig 's/C::pi/M_PI/g' ${_casacore_source_dir}/lattices/LatticeMath/Fit2D*
    COMMAND sed -i.orig 's/C::pi/M_PI/g' ${_casacore_source_dir}/scimath/Functionals/Lorentzian1DParam.tcc
    COMMAND sed -i.orig 's/C::_1_pi/M_1_PI/g' ${_casacore_source_dir}/scimath/Functionals/Lorentzian1DParam.tcc
)

# fixes warnings on cmake 3.x+ For now we want the old behavior to be backwards compatible
if (POLICY CMP0115)
    cmake_policy (SET CMP0115 NEW)
endif()

# Enable <packageName>_ROOT variables for recent versions of CMake
if (POLICY CMP0074)
    cmake_policy (SET CMP0074 NEW)
endif()

# Increment the PROJECT_SOVERSION every time you update VERSION_MINOR!
SET(PROJECT_SOVERSION 9)

SET(NO_SOVERSION FALSE CACHE BOOL "do not add version information to shared libraries")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

set(CPACK_GENERATOR "DEB")
set(CPACK_PACKAGE_NAME "casacore")
set(CPACK_PACKAGE_VERSION "${PROJECT_VERSION}")

set(CPACK_PACKAGE_CONTACT "Malte Marquarding") #required
set(CPACK_PACKAGE_VENDOR "https://github.com/casacore/casacore")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Astronomical data processing library")

option (BUILD_DEPRECATED "build and install deprecated classes (such as Map)" NO)
option (BUILD_FFTPACK_DEPRECATED "build FFTPack" NO)
option (BUILD_DYSCO "Build the Dysco compression storage manager" YES)
option (BUILD_SISCO "Build the Sisco compression storage manager" NO)

# By default build shared libraries
option (ENABLE_SHARED "Build shared libraries" YES)
option (ENABLE_RPATH "Include rpath in executables and shared libraries" YES)

# Check if O_DIRECT is available.
# Note this does not work well with check_c_source_compiles.
check_cxx_source_compiles("
  #include <fcntl.h>
  main() { int i = O_DIRECT; }
  " HAVE_O_DIRECT)
if (HAVE_O_DIRECT)
    add_definitions(-DHAVE_O_DIRECT)
endif()

# By default do not use ADIOS2, MPI, or STACKTRACE
option (ENABLE_TABLELOCKING "Make locking for concurrent table access possible" YES)
option (USE_READLINE "Build readline support" YES)
option (USE_ADIOS2 "Build ADIOS2 " NO)
option (USE_HDF5 "Build HDF5 " YES)
option (USE_THREADS "Use Mutex thread synchronization" YES)
option (USE_OPENMP "Use OpenMP threading" YES)
option (USE_MPI "Use MPI for parallel IO" NO)
option (USE_STACKTRACE "Show stacktrace in case of exception" NO)
option (BUILD_TESTING "Enable build of the unit tests" NO)

set(CASA_DEFAULT_ALIGNMENT "32" CACHE STRING "Default alignment of casa::AlignedAllocator")

# ccache use is optional
option( UseCcache OFF )

if (UseCcache)
    message (STATUS "Searching for ccache.")
    find_program(CCACHE_FOUND ccache)
    if(CCACHE_FOUND)
        message (STATUS "Ccache found.")
        set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
        set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
    endif(CCACHE_FOUND)
endif()


# Test if shared libraries have to be built.
if (ENABLE_SHARED)
    option (BUILD_SHARED_LIBS "" YES)
    if (ENABLE_RPATH)
        # Set RPATH to use for installed targets; append linker search path
        set(CMAKE_INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib" )
        set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
        set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
        set(CMAKE_MACOSX_RPATH TRUE)
    endif (ENABLE_RPATH)
else()
    option (BUILD_SHARED_LIBS "" NO)
endif (ENABLE_SHARED)

# Define the compiler flags to be used.
# Note: -Wshadow and -Wunreachable-code give (too) many warnings.
# Casacore uses longlong, so no warnings for it.
# Clang gives warning on bison generated code; disable unneeded-internal-declaration.
if (NOT CMAKE_CXX_FLAGS)
    set (CMAKE_CXX_FLAGS "-Wextra -Wall -W -Wpointer-arith -Woverloaded-virtual -Wwrite-strings -pedantic -Wno-long-long")
    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unneeded-internal-declaration")
    endif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
endif (NOT CMAKE_CXX_FLAGS)

# Set build type if not given.
if (NOT CMAKE_BUILD_TYPE)
    # Use debug mode if building in dbg or debug directory.
    get_filename_component(_cmpvar ${CMAKE_BINARY_DIR} NAME)
    if(_cmpvar STREQUAL "dbg" OR _cmpvar STREQUAL "debug")
        set (CMAKE_BUILD_TYPE Debug)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DAIPS_DEBUG")
    else()
        if(_cmpvar STREQUAL "cov")
            set (CMAKE_BUILD_TYPE Debug)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0 --coverage")
            set(CMAKE_LD_FLAGS "${CMAKE_LD_FLAGS} --coverage")
        else()
            set (CMAKE_BUILD_TYPE Release)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNDEBUG")
        endif(_cmpvar STREQUAL "cov")
    endif(_cmpvar STREQUAL "dbg" OR _cmpvar STREQUAL "debug")
endif (NOT CMAKE_BUILD_TYPE)

# Sisco requires a C++20 compatible compiler. If Sisco is not built, fall back to C++17 for the
# time being.
if(BUILD_SISCO)
  set(CMAKE_CXX_STANDARD 20)
else(BUILD_SISCO)
  set(CMAKE_CXX_STANDARD 17)
endif(BUILD_SISCO)

set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Ensure clang is not complaining about unused arguments.
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    set(CMAKE_CXX_FLAGS "-Qunused-arguments ${CMAKE_CXX_FLAGS}")
    set(CMAKE_C_FLAGS "-Qunused-arguments ${CMAKE_C_FLAGS}")
endif()

# Since 2015, we need pread and pwrite (POSIX 2001)
check_function_exists(pread HAVE_PREAD)
check_function_exists(pwrite HAVE_PWRITE)
if (NOT (HAVE_PREAD AND HAVE_PWRITE))
        message(FATAL_ERROR "Casacore requires pread and pwrite functionality" )
endif (NOT (HAVE_PREAD AND HAVE_PWRITE))

# Since 2020, we need GCC 4.9 for regex support
# Additionally we need cmake 2.8.10 for CMAKE_CXX_COMPILER_VERSION
if (CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9)
    message(FATAL_ERROR "Casacore requires at least gcc-4.9")
endif()

# FindHDF5 uses environment variables, so set it if needed.
if (HDF5_ROOT_DIR)
    set (ENV{HDF5_ROOT} ${HDF5_ROOT_DIR})
endif (HDF5_ROOT_DIR)

# Find out which modules to build.
if (NOT MODULE)
   set (MODULE all)
endif (NOT MODULE)
set (_usebison NO)
set (_uselapack NO)
set (_usefortran NO)
set (_usefits NO)
set (_usewcs NO)
set (_modules casa)
set (_modules2 )
if (NOT ${MODULE} STREQUAL "casa")
    set (_modules ${_modules} tables)
    set (_usebison YES)
    if (NOT ${MODULE} STREQUAL "tables")
        set (_modules ${_modules} scimath_f scimath measures meas)
        set (_uselapack YES)
        set (_usefortran YES)
        if (NOT ${MODULE} STREQUAL "measures")
            if (${MODULE} STREQUAL "ms")
                set (_modules ${_modules} ms derivedmscal)
            endif()
            if (${MODULE} STREQUAL "msfits" OR ${MODULE} STREQUAL "all")
                set (_modules2 ${_modules2} ms msfits derivedmscal)
                set (_usefits YES)
            endif()
            if (${MODULE} STREQUAL "images" OR ${MODULE} STREQUAL "all")
                set (_modules2 ${_modules2} lattices mirlib coordinates images)
                set (_usewcs YES)
                set (_usefits YES)
            endif()
        endif()
    endif()
endif()

if (_usefits)
    set (_modules ${_modules} fits)
endif()
set (_modules ${_modules} ${_modules2})


# Determine which external packages to use.
include (CTest)

if (_usefortran)
  enable_language (Fortran)

  # use faster fortran rules for complex operations, removes restoring complex
  # infinities if naive computation results in NAN + NAN * I
  #Handling complex multiplication and division with correct treating of complex
  #infinities (one element Inf regardless of the other) according to the C is
  #complicated.
  #E.g. a = NaN + 1e30 i; a * a is not NaN but a complex infinity (-Inf - NaN).
  #
  #Treating this situation correctly has large performance impact. In GCC's
  #implementation it is about 4 times slower than the naive implementation, with
  #vectorization enabled the impact is even larger.
  #As correct treatment of complex infinities when NaN appear in results is seldom
  #accounted for, or not required and most other languages do not have these
  #rules, the correct treatmeant can be disabled with the -fcx-fortran-rules flag.
  #This changes the semantics to those of the FORTRAN language which is removes
  #the need for rescuing the result when NaN appear. Python also follows FORTRAN
  #rules.
  #Additionally the correct behavior is not implemented in all compilers,
  #most notably clang which is the default compiler on MacOS. So turning off
  #correct treatment with GCC does not only make our code faster but also behave
  #the same on more compilers.
  #
  #This has measurable impact on e.g. applycal where the cpu performance improves
  #by about 20%.
  # For one reason or another the check on the compiler flag has no
  # effect; it still adds the option which fails for clang.
  if (NOT "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
      check_cxx_compiler_flag(-fcx-fortran-rules HAS_GXX_FORTRAN_RULES)
      check_c_compiler_flag(-fcx-fortran-rules HAS_GCC_FORTRAN_RULES)
      # added before cmake flags so it can be disabled with
      # -fno-cx-fortran-rules for testing
      if (HAS_GXX_FORTRAN_RULES)
          set(CMAKE_CXX_FLAGS "-fcx-fortran-rules ${CMAKE_CXX_FLAGS}")
      endif()
      if (HAS_GCC_FORTRAN_RULES)
          set(CMAKE_C_FLAGS "-fcx-fortran-rules ${CMAKE_C_FLAGS}")
      endif()
  endif()
endif()

find_package (DL)
if (USE_READLINE)
    find_package (Readline REQUIRED)
endif (USE_READLINE)

if (BUILD_TESTING)
    find_package (SOFA)
endif()

if (USE_ADIOS2)
    find_package (ADIOS2 2.6.0 CONFIG REQUIRED)
    if(NOT USE_MPI)
        set(CASACORE_ADIOS_LIBRARY adios2::cxx11)
    elseif (NOT ADIOS2_HAVE_MPI)
        message(FATAL_ERROR "ADIOS2 found without MPI support but MPI support requested")
    else()
        set(CASACORE_ADIOS_LIBRARY adios2::cxx11_mpi)
    endif()
endif (USE_ADIOS2)

if (USE_HDF5)
    find_package (HDF5 REQUIRED)
endif (USE_HDF5)

if (_usebison STREQUAL YES)
    find_package (FLEX REQUIRED)
    find_package (BISON 3 REQUIRED)
endif (_usebison STREQUAL YES)

if (_uselapack STREQUAL YES)
    find_package (BLAS REQUIRED)
    find_package (LAPACK REQUIRED)
    if (FFTW3_DISABLE_THREADS)
        find_package (FFTW3 COMPONENTS single double REQUIRED)
    else()
        find_package (FFTW3 COMPONENTS single double threads REQUIRED)
    endif (FFTW3_DISABLE_THREADS)
endif (_uselapack STREQUAL YES)

if (_usefits STREQUAL YES)
    find_package (CFITSIO 3.030 REQUIRED) # Should pad to three decimal digits
endif (_usefits STREQUAL YES)

if (_usewcs STREQUAL YES)
    find_package (WCSLIB 4.7 REQUIRED)     # needed for CASA
    if (WCSLIB_VERSION_STRING STREQUAL "5.14")
      # WCSlib 5.14 throws segfaults on lots of tests, e.g. tTempImage
      message(WARNING "Casacore is not compatible with wcslib 5.14, see issue gh-384.")
    endif (WCSLIB_VERSION_STRING STREQUAL "5.14")
endif (_usewcs STREQUAL YES)

# Set the include directories and HAVE compiler variables
include_directories (${CMAKE_SOURCE_DIR} ${CMAKE_BINARY_DIR})
if (BUILD_DYSCO)
    add_definitions(-DHAVE_DYSCO)
endif (BUILD_DYSCO)
if (BUILD_SISCO)
    add_definitions(-DHAVE_SISCO)
endif (BUILD_SISCO)
if (WCSLIB_FOUND)
    include_directories (${WCSLIB_INCLUDE_DIRS})
    add_definitions (-DWCSLIB_VERSION_MAJOR=${WCSLIB_VERSION_MAJOR} -DWCSLIB_VERSION_MINOR=${WCSLIB_VERSION_MINOR})
endif (WCSLIB_FOUND)
if (CFITSIO_FOUND)
    include_directories (${CFITSIO_INCLUDE_DIRS})
    add_definitions (-DCFITSIO_VERSION_MAJOR=${CFITSIO_VERSION_MAJOR} -DCFITSIO_VERSION_MINOR=${CFITSIO_VERSION_MINOR})
endif (CFITSIO_FOUND)
if (ADIOS2_FOUND)
    include_directories (${ADIOS2_INCLUDE_DIRS})
    add_definitions(-DHAVE_ADIOS2)
endif (ADIOS2_FOUND)
if (HDF5_FOUND)
    # Use BEFORE to achieve that HDF5 is not taken from /usr/local/include
    ##include_directories (BEFORE ${HDF5_INCLUDE_DIRS})
    include_directories (${HDF5_INCLUDE_DIRS})
    add_definitions(-DHAVE_HDF5)
    if (DEFINED "HDF5_hdf5_LIBRARY")
        set(_hdf5_library "${HDF5_hdf5_LIBRARY}")
    elseif (DEFINED "HDF5_C_LIBRARY")
        set(_hdf5_library "${HDF5_C_LIBRARY}")
    elseif (DEFINED "HDF5_C_LIBRARY_hdf5")
        set(_hdf5_library "${HDF5_C_LIBRARY_hdf5}")
    endif ()
endif (HDF5_FOUND)

include_directories (${FFTW3_INCLUDE_DIRS})
add_definitions(-DHAVE_FFTW3)
if (NOT FFTW3_DISABLE_THREADS)
    add_definitions(-DHAVE_FFTW3_THREADS)
endif (NOT FFTW3_DISABLE_THREADS)

if (DL_FOUND)
    add_definitions(-DHAVE_DL)
endif (DL_FOUND)
if (READLINE_FOUND)
    add_definitions(-DHAVE_READLINE)
endif (READLINE_FOUND)

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
   ## setting intel libraries with e.g.
   ##
   ##    find_library( INTEL_IRNG irng HINTS ${INTEL_PATH} )
   ##
   ## causes CMAKE to substitute fully qualified paths which makes
   ## python shared object unrelocatable in the case of libirng.so
   ##
   get_filename_component(INTEL_PATH ${CMAKE_CXX_COMPILER} DIRECTORY)
   set(INTEL_LIB_PATH ${INTEL_PATH}/../lib/intel64)
   set(CASACORE_ARCH_LIBS ${CASACORE_ARCH_LIBS} -L${INTEL_LIB_PATH} -limf -lsvml -lirng -lintlc -lifport -lifcore -liomp5)
endif()

if(USE_MPI)
    find_package(MPI COMPONENTS C REQUIRED)
    include_directories (${MPI_C_INCLUDE_PATH})
    add_definitions(-DOMPI_SKIP_MPICXX -DMPICH_SKIP_MPICXX)
    add_definitions(-DHAVE_MPI)

    set(ENABLE_TABLELOCKING OFF)
    set(CASACORE_MPI_LIBRARY ${MPI_C_LIBRARIES})
    message (STATUS "ENABLE_TABLELOCKING is turned off as required by MPI")
endif(USE_MPI)

if(USE_OPENMP)
    set (USE_THREADS YES)
    find_package (OpenMP)
    if (OPENMP_FOUND)
        if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -qopenmp")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -qopenmp")
        else( )
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
        endif( )
    else (OPENMP_FOUND)
        message(WARNING "Cannot fulfill USE_OPENMP, compiler does not support it")
    endif (OPENMP_FOUND)
endif()

# Thread support?
if(USE_THREADS)
    set(_errmsg "FIXME: Don't know how to enable thread support for ")
    find_package(Pthreads REQUIRED)
    add_definitions(-DUSE_THREADS)
    if(CMAKE_COMPILER_IS_GNUCC)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pthread")
    elseif ("${CMAKE_C_COMPILER_ID}" STREQUAL "Intel")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pthread")
    else()
        if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pthread")
        else()
            message(FATAL_ERROR "${_errmsg} (${CMAKE_C_COMPILER_ID}): ${CMAKE_C_COMPILER}")
        endif ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    endif(CMAKE_COMPILER_IS_GNUCC)
    if(CMAKE_COMPILER_IS_GNUCXX)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
    else()
        if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
        elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
        else()
            message(FATAL_ERROR "${_errmsg} (${CMAKE_C_COMPILER_ID}): ${CMAKE_CXX_COMPILER}")
        endif ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    endif(CMAKE_COMPILER_IS_GNUCXX)
else()
    # Compile with -fno-threadsafe-statics if supp to reduce some thread safety overhead.
    check_cxx_compiler_flag(-fno-threadsafe-statics HAS_CXX_NO_THREADSAFE_STATICS)
    if (HAS_CXX_NO_THREADSAFE_STATICS)
        set(CMAKE_CXX_FLAGS "-fno-threadsafe-statics ${CMAKE_CXX_FLAGS}")
    endif()
    check_c_compiler_flag(-fno-threadsafe-statics HAS_C_NO_THREADSAFE_STATICS)
    if (HAS_C_NO_THREADSAFE_STATICS)
        set(CMAKE_C_FLAGS "-fno-threadsafe-statics ${CMAKE_C_FLAGS}")
    endif()
endif(USE_THREADS)

# Set default DATA_DIR if undefined.
if (NOT DATA_DIR)
   set (DATA_DIR ${CMAKE_INSTALL_PREFIX}/share/casacore/data)
endif (NOT DATA_DIR)
# Let cmake cache DATA_DIR.
set (DATA_DIR "${DATA_DIR}" CACHE PATH "Measures tables root")


# Set compiler flag if deprecated Casacore header files are allowed.
if (BUILD_DEPRECATED)
    add_definitions(-DAIPS_USE_DEPRECATED)
endif ()
# Set compiler flag if no table locking.
if (NOT ENABLE_TABLELOCKING)
    add_definitions(-DAIPS_TABLE_NOLOCKING)
endif (NOT ENABLE_TABLELOCKING)
# Set compiler flag if stack tracing.
if(USE_STACKTRACE)
    add_definitions(-DUSE_STACKTRACE)
endif(USE_STACKTRACE)

# Enable cmake testing and add make check target that builds and runs the test
enable_testing()
add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})
# This legacy flag always builds the tests and runs them with make test. There
# seems to be no good way to make test executable depend on the test target
if (NOT BUILD_TESTING)
    set(EXCL_ALL EXCLUDE_FROM_ALL)
endif (NOT BUILD_TESTING)

# Define SOVERSION as compile variable.
set (LIB_VERSION "${PROJECT_SOVERSION}")
set (LIB_SOVERSION "${PROJECT_SOVERSION}")

# Add the modules to be built.
add_subdirectory (${_casacore_dir}/build-tools)
foreach (module ${_modules})
    add_subdirectory (${_casacore_dir}/${module})
    set_target_properties(
      casa_${module}
      PROPERTIES
          VERSION "${LIB_VERSION}"
          SOVERSION "${LIB_SOVERSION}"
    )
    if (APPLE)
        if (${module} STREQUAL scimath_f OR ${module} STREQUAL fits OR ${module} STREQUAL mirlib OR ${module} STREQUAL coordinates)
            set_target_properties(casa_${module} PROPERTIES LINK_FLAGS -single_module)
        endif (${module} STREQUAL scimath_f OR ${module} STREQUAL fits OR ${module} STREQUAL mirlib OR ${module} STREQUAL coordinates)
    endif (APPLE)
    set(PRIVATE_LIBS "${PRIVATE_LIBS} -lcasa_${module}")
endforeach (module)

# Add custom imageanalysis component
add_subdirectory (imageanalysis)

install(
    FILES scripts/casa_data_autoupdate.sh
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
    DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
    RENAME casa_data_autoupdate
)

# Show summary.
message (STATUS "CMAKE_SYSTEM .......... = ${CMAKE_SYSTEM}")
message (STATUS "CMAKE_BUILD_TYPE ...... = ${CMAKE_BUILD_TYPE}")
message (STATUS "UseCcache ............. = ${UseCcache}")
message (STATUS "Modules to be built ... = ${_modules}")
message (STATUS "BUILD_SHARED_LIBS ..... = ${BUILD_SHARED_LIBS}")
message (STATUS "ENABLE_RPATH .......... = ${ENABLE_RPATH}")
message (STATUS "CMAKE_INSTALL_NAME_DIR  = ${CMAKE_INSTALL_NAME_DIR}")
message (STATUS "ENABLE_TABLELOCKING ... = ${ENABLE_TABLELOCKING}")
message (STATUS "USE_PCH ............... = ${USE_PCH}")
message (STATUS "USE_THREADS ........... = ${USE_THREADS}")
message (STATUS "USE_OPENMP ............ = ${USE_OPENMP}")
message (STATUS "USE_MPI ............... = ${USE_MPI}")
message (STATUS "USE_STACKTRACE ........ = ${USE_STACKTRACE}")
message (STATUS "BUILD_FFTPACK_DEPRECATED= ${BUILD_FFTPACK_DEPRECATED}")
message (STATUS "BUILD_DEPRECATED ...... = ${BUILD_DEPRECATED}")
message (STATUS "BUILD_DYSCO ........... = ${BUILD_DYSCO}")
message (STATUS "BUILD_SISCO ........... = ${BUILD_SISCO}")
message (STATUS "BUILD_TESTING ......... = ${BUILD_TESTING}")
message (STATUS "HAVE_O_DIRECT ......... = ${HAVE_O_DIRECT}")
message (STATUS "CMAKE_CXX_COMPILER .... = ${CMAKE_CXX_COMPILER}")
message (STATUS "CMAKE_CXX_FLAGS ....... = ${CMAKE_CXX_FLAGS}")
message (STATUS "DATA directory ........ = ${DATA_DIR}")
message (STATUS "DL library? ........... = ${DL_LIBRARIES}")
message (STATUS "Pthreads library? ..... = ${PTHREADS_LIBRARIES}")
message (STATUS "Readline library? ..... = ${READLINE_LIBRARIES}")
message (STATUS "BLAS library? ......... = ${BLAS_LIBRARIES}")
message (STATUS "LAPACK library? ....... = ${LAPACK_LIBRARIES}")
message (STATUS "WCS library? .......... = ${WCSLIB_LIBRARIES}")
message (STATUS "SOFA library? ......... = ${SOFA_LIBRARIES}")
message (STATUS "CFitsio library? ...... = ${CFITSIO_LIBRARIES}")
message (STATUS "ADIOS2 library? ....... = ${CASACORE_ADIOS_LIBRARY}")
message (STATUS "HDF5 library? ......... = ${_hdf5_library}")
message (STATUS "FFTW3 library? ........ = ${FFTW3_LIBRARIES}")

# List of build variables and defaults.
#  ENABLE_SHARED                 YES
#  ENABLE_RPATH                  YES
#  ENABLE_TABLELOCKING           YES
#  USE_ADIOS2                    NO
#  USE_HDF5                      YES
#  USE_THREADS                   YES
#  USE_OPENMP                    YES
#  USE_MPI                       NO
#  USE_STACKTRACE                NO
#  BUILD_DEPRECATED              NO
#  BUILD_TESTING                 NO
#  UseCcache                     NO
#  DATA_DIR                      ${CMAKE_INSTALL_PREFIX}/share/casacore/data
#  MODULE                        all
#                                Possible value for MODULE (previous built too):
#                                - casa     (casa)
#                                - tables   (tables)
#                                - measures (scimath,scimath_f,measures,meas)
#                                -  ms      (ms,derivedmscal)
#                                or msfits  (fits,ms,msfits,derivedmscal)
#                                or images  (fits,lattices,mirlib,coordinates,images)
#                                - all
#
# List of possibly used external packages and where
#  CFITSIO      fits
#  WCSLIB       coordinates
#  SOFA         measures (optional, only for testing)
#  DL           casa (optional)
#  READLINE     casa (optional)
#  HDF5         casa (optional)
#  BISON        casa,tables,images
#  FLEX         casa,tables,images
#  ADIOS2       tables (optional)
#  LAPACK       scimath
#  BLAS         scimath
#  FFTW         scimath
