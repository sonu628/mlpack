# Build rules for the MATLAB bindings for MLPACK.  These may not work well on
# non-Linux systems.

# We need the mex compiler for this to work.
find_package(MatlabMex REQUIRED)

# If the mex compiler is wrapping an "unsupported" version, warn the user that
# they may have issues with the produced bindings for a multitude of reasons.
# We can only reasonably check this on a UNIX-like system.
if(UNIX)
  # The file test.cpp does not exist, but mex will still print a warning if it's
  # using a weird version.
  execute_process(COMMAND "${MATLAB_MEX}" test.cpp
                  RESULT_VARIABLE MEX_RESULT_TRASH
                  OUTPUT_VARIABLE MEX_OUTPUT
                  ERROR_VARIABLE MEX_ERROR_TRASH)

  string(REGEX MATCH "Warning: You are using" MEX_WARNING "${MEX_OUTPUT}")

  if(MEX_WARNING)
    # We have to find the old compiler version and the new compiler version; if
    # the MATLAB version is newer, then we don't need to worry.  If this step
    # fails somehow, we will just issue the warning anyway (just in case).
    string(REGEX REPLACE
        ".*using [a-zA-Z]* version \"([0-9.]*)[^\"]*\".*"
        "\\1" OTHER_COMPILER_VERSION "${MEX_OUTPUT}")
    string(REGEX REPLACE
        ".*currently supported with MEX is \"([0-9.]*)[^\"]*\".*"
        "\\1" MEX_COMPILER_VERSION "${MEX_OUTPUT}")

    # If MEX_COMPILER_VERSION is greater than OTHER_COMPILER_VERSION, we don't
    # need to issue a warning.
    set(NEED_TO_WARN 1)
    if(MEX_COMPILER_VERSION AND OTHER_COMPILER_VERSION)
      # We seem to have read two valid version strings.  So we can compare
      # them, and maybe we don't need to issue the warning.
      if(NOT ("${MEX_COMPILER_VERSION}" VERSION_LESS
          "${OTHER_COMPILER_VERSION}"))
        # The mex compiler is newer than our version.  So no warning is
        # needed.
        set(NEED_TO_WARN 0)
      endif(NOT ("${MEX_COMPILER_VERSION}" VERSION_LESS
          "${OTHER_COMPILER_VERSION}"))
    endif()

    if(NEED_TO_WARN EQUAL 1)
      message(WARNING "The MATLAB runtime glibc is different than the system "
          " glibc.  This can (and probably will) cause the MLPACK bindings "
          "generated by this build script to fail with odd GLIBCXX_a_b_c "
          "version complaints when they are run.  Assuming that the system "
          "glibc is newer than the MATLAB-provided version, the MATLAB version "
          "can probably be deleted (always save a copy in case this is wrong!)."
          "\nFor more information on this confusing issue, see\n"
          "http://dovgalecs.com/blog/matlab-glibcxx_3-4-11-not-found/\nand for "
          "an overly-detailed dissertation/rant on why it is not possible to "
          "work around this issue in any way, see\n"
          "http://www.mlpack.org/trac/ticket/253 for more information.")
    endif()
  endif()
endif()

# Ignore the fact that we are setting CMAKE_SHARED_LIBRARY_CXX_FLAGS on CMake
# 2.8.9 and newer.  Because we are requiring at least CMake 2.8.5, we only have
# to check the patch version.
if(${CMAKE_PATCH_VERSION} GREATER 8)
  cmake_policy(SET CMP0018 OLD)
endif()

# Use the mex compiler to compile.
set(CMAKE_CXX_COMPILER "${MATLAB_MEX}")

# Set flags for the mex compiler, because a lot of the default CMake flags
# aren't accepted by mex.  The user who wants to customize these things should
# probably modify their mexopts.sh so that mex uses those flags by default.
# There is no easy way to tell mex to compile with profiling symbols, so that is
# not done even if PROFILE is set.
if(DEBUG)
  set(CMAKE_CXX_FLAGS "-g")
  set(CMAKE_C_FLAGS "-g")
else()
  set(CMAKE_CXX_FLAGS "-O")
  set(CMAKE_C_FLAGS "-O")
endif()

# Don't give -fPIC; mex will do that for us.
set(CMAKE_SHARED_LIBRARY_C_FLAGS "")
set(CMAKE_SHARED_LIBRARY_CXX_FLAGS "")

# Don't make 'lib<method>.mexglx'.
set(CMAKE_SHARED_LIBRARY_PREFIX "")
set(CMAKE_SHARED_MODULE_PREFIX "")

# Set custom commands for mex compilation, because the flags are (in general)
# odd and different.
set(CMAKE_CXX_COMPILE_OBJECT "<CMAKE_CXX_COMPILER> -outdir <OBJECT_DIR> <FLAGS> -c <SOURCE>")
set(CMAKE_CXX_CREATE_SHARED_MODULE "<CMAKE_CXX_COMPILER> -cxx <LINK_FLAGS> -output <TARGET> <OBJECTS> <LINK_LIBRARIES>")
set(CMAKE_CXX_CREATE_SHARED_LIBRARY "${CMAKE_CXX_CREATE_SHARED_MODULE}")

# mex is weird because it doesn't respect the -o option, but in general it
# appears to turn <source>.cpp into <source>.o, so CMake needs to know to
# replace the extension.
set(CMAKE_CXX_OUTPUT_EXTENSION_REPLACE 1)

if(${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64")
  set(CMAKE_SHARED_LIBRARY_SUFFIX ".mexa64")
  set(CMAKE_SHARED_MODULE_SUFFIX  ".mexa64")
elseif(${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86" OR ${CMAKE_SYSTEM_PROCESSOR}
    STREQUAL "i686")
  set(CMAKE_SHARED_LIBRARY_SUFFIX ".mexglx")
  set(CMAKE_SHARED_MODULE_SUFFIX  ".mexglx")
endif()

# Place MATLAB bindings in matlab/.
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/matlab/)

include_directories(${CMAKE_SOURCE_DIR}/src/) # So we can include <mlpack/...>.

# Set MATLAB toolbox install directory.
set(MATLAB_TOOLBOX_DIR "${MATLAB_ROOT}/toolbox")

# CHANGE HERE FOR NEW BINDINGS!!!!
add_subdirectory(allkfn)
add_subdirectory(allknn)
add_subdirectory(emst)
add_subdirectory(kmeans)
add_subdirectory(range_search)
add_subdirectory(gmm)
add_subdirectory(pca)
add_subdirectory(kernel_pca)
add_subdirectory(lars)
add_subdirectory(nca)
add_subdirectory(nmf)

# Create a target whose sole purpose is to modify the pathdef.m MATLAB file so
# that the MLPACK toolbox is added to the MATLAB default path.
add_custom_target(matlab ALL
    # Modify pathdef.m.
    COMMAND ${CMAKE_COMMAND} -D MATLAB_ROOT="${MATLAB_ROOT}" -D
        PATHDEF_OUTPUT_FILE="${CMAKE_BINARY_DIR}/matlab/pathdef.m" -P
        ${CMAKE_SOURCE_DIR}/CMake/ModifyMatlabPathdef.cmake
    # Due to the dependencies, 'make matlab' makes all the bindings.
    DEPENDS
    allknn_mex
    allkfn_mex
    emst_mex
    gmm_mex
    kmeans_mex
    range_search_mex
)

install(FILES "${CMAKE_BINARY_DIR}/matlab/pathdef.m"
    DESTINATION "${MATLAB_ROOT}/toolbox/local/"
)

