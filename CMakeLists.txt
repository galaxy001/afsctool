cmake_minimum_required(VERSION 3.0 FATAL_ERROR)
project(afsctool VERSION 1.6.9)
set(afsctool_FULL_VERSION "${afsctool_VERSION_MAJOR}.${afsctool_VERSION_MINOR}.${afsctool_VERSION_PATCH}.${afsctool_VERSION_TWEAK}")

option(HFSCOMPRESS_TO_ZFS
    "Should afsctool compress files on ZFS dataset that claim to be HFS (testing only: the effort will be wasted)"
    OFF)
option(ZLIB_SINGLESHOT
    "Does the ZLIB compression into a sufficiently (= too) large output buffer instead of using a growing buffer.\
    May be somewhat faster at the expense of approx. 4x higher memory usage."
    OFF)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules ${CMAKE_MODULE_PATH} )

include(FeatureSummary)
include(CheckIncludeFile)

include(GetGitRevisionDescription)

find_package(ZLIBP 1.2.8 REQUIRED)
find_package(SPARSEHASH)
check_include_file("sys/decmpfs.h" HAS_DECMPFS)
find_file(LZVN_HEADER "FastCompression.h")
find_library(LZVN_LIBRARY "FastCompression")
message(STATUS "LZVN_HEADER,LIB: ${LZVN_HEADER} ${LZVN_LIBRARY}")

include_directories(${ZLIBP_INCLUDE_DIR} ${SPARSEHASH_INCLUDE_DIR} ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_SOURCE_DIR})
link_directories(${SPARSEHASH_LIBRARY})
add_definitions(-DSUPPORT_PARALLEL)
if(ZLIB_SINGLESHOT)
    add_definitions(ZLIB_SINGLESHOT_OUTBUF)
endif()
if(HAS_DECMPFS)
    message(STATUS "Found system decmpfs.h header")
    add_definitions(-DHAS_DECMPFS)
endif()
if(LZVN_HEADER AND LZVN_LIBRARY)
    message(STATUS "Enabling LZVN support")
    add_definitions(-DHAS_LZVN)
else()
    message(STATUS "LZVN support is disabled")
    unset(LZVN_LIBRARY)
endif()

git_describe(GIT_FULL_VERSION "--tags")
if(GIT_FULL_VERSION)
    # development build, i.e. from a git working copy.
    # Get the current commit version using `git describe` and
    # strip the leading v plus the part that matches the version
    # declared in the project definition above.
    string(REPLACE "v${afsctool_FULL_VERSION}" "" COMMIT_VERSION "${GIT_FULL_VERSION}")
    if(NOT ${COMMIT_VERSION} MATCHES ${GIT_FULL_VERSION})
        # `git describe` agrees with afsctool_VERSION_STRING;
        # make an atomic version string by appending the additional info
        string(APPEND afsctool_FULL_VERSION ${COMMIT_VERSION})
    else()
        # `git describe` provides a seemingly unrelated version string;
        # append it in parentheses.
        string(APPEND afsctool_FULL_VERSION " (${GIT_FULL_VERSION})")
    endif()
    message(STATUS "Setting afsctool version to: ${afsctool_FULL_VERSION}")
endif()
configure_file(afsctool_fullversion.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/afsctool_fullversion.h)

add_executable(afsctool
    afsctool.c
    main.cpp
    ParallelProcess.cpp
    Thread/Thread.cpp
    CritSectEx/CritSectEx.cpp
    CritSectEx/msemul.cpp
    CritSectEx/timing.c
)
if(HFSCOMPRESS_TO_ZFS)
    set_source_files_properties(afsctool.c PROPERTIES COMPILE_DEFINITIONS HFSCOMPRESS_TO_ZFS)
endif()

target_include_directories(afsctool PRIVATE ${CMAKE_SOURCE_DIR})
# set_target_properties(afsctool PROPERTIES COMPILE_FLAGS -DSUPPORT_PARALLEL)
target_link_libraries(afsctool ${ZLIBP_LIBRARY_LDFLAGS} ${ZLIBP_LIBRARIES} ${PKG_SPARSEHASH_LIBRARIES})
if(LZVN_LIBRARY)
    target_link_libraries(afsctool ${LZVN_LIBRARY})
endif()
if(APPLE)
    target_link_libraries(afsctool "-framework CoreServices")
else()
    target_link_libraries(afsctool "-lrt -ldl -lbsd -pthread")
endif()

install(TARGETS afsctool DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
FEATURE_SUMMARY(WHAT ALL INCLUDE_QUIET_PACKAGES FATAL_ON_MISSING_REQUIRED_PACKAGES)
