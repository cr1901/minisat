cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(minisat)
cmake_policy(SET CMP0042 NEW)
#--------------------------------------------------------------------------------------------------
# Configurable options:

option(STATIC_BINARIES "Link binaries statically." ON)
option(USE_SORELEASE   "Use SORELEASE in shared library filename." ON)
option(LIBRARY_ONLY "Build only minisat library (ZLIB not needed- dependents require -DMINISAT_LIBRARY_ONLY)" OFF)

#--------------------------------------------------------------------------------------------------
# Library version:

set(MINISAT_SOMAJOR   2)
set(MINISAT_SOMINOR   1)
set(MINISAT_SORELEASE 0)

# Compute VERSION and SOVERSION:
if (USE_SORELEASE)
  set(MINISAT_VERSION ${MINISAT_SOMAJOR}.${MINISAT_SOMINOR}.${MINISAT_SORELEASE})
else()
  set(MINISAT_VERSION ${MINISAT_SOMAJOR}.${MINISAT_SOMINOR})
endif()
set(MINISAT_SOVERSION ${MINISAT_SOMAJOR})

#--------------------------------------------------------------------------------------------------
# Dependencies:

if(LIBRARY_ONLY)
  add_definitions(-DMINISAT_LIBRARY_ONLY)
else()
  find_package(ZLIB)
  include_directories(${ZLIB_INCLUDE_DIR})
endif()

include_directories(${minisat_SOURCE_DIR})

#--------------------------------------------------------------------------------------------------
# Compile flags:

add_definitions(-D__STDC_FORMAT_MACROS -D__STDC_LIMIT_MACROS)

if(WASI)
  add_definitions(-D_WASI_EMULATED_PROCESS_CLOCKS -D_WASI_EMULATED_SIGNAL)
endif()

#--------------------------------------------------------------------------------------------------
# Build Targets:

set(MINISAT_LIB_SOURCES
    minisat/utils/Options.cc
    minisat/utils/System.cc
    minisat/core/Solver.cc
    minisat/simp/SimpSolver.cc)

add_library(minisat-lib-static STATIC ${MINISAT_LIB_SOURCES})
if(NOT WASI)
  add_library(minisat-lib-shared SHARED ${MINISAT_LIB_SOURCES})
endif()

if(NOT WASI)
  target_link_libraries(minisat-lib-shared ${ZLIB_LIBRARY})
endif()
target_link_libraries(minisat-lib-static ${ZLIB_LIBRARY})

if(NOT LIBRARY_ONLY)
  if(WASI)
    message(FATAL_ERROR "WASI build of MiniSAT does not support binaries yet.")
  endif()

  add_executable(minisat_core minisat/core/Main.cc)
  add_executable(minisat_simp minisat/simp/Main.cc)

  if(STATIC_BINARIES)
    target_link_libraries(minisat_core minisat-lib-static)
    target_link_libraries(minisat_simp minisat-lib-static)
  else()
    target_link_libraries(minisat_core minisat-lib-shared)
    target_link_libraries(minisat_simp minisat-lib-shared)
  endif()
endif()

set_target_properties(minisat-lib-static PROPERTIES OUTPUT_NAME "minisat")
if(NOT WASI)
  set_target_properties(minisat-lib-shared
    PROPERTIES
      OUTPUT_NAME "minisat"
      VERSION ${MINISAT_VERSION}
      SOVERSION ${MINISAT_SOVERSION})
endif()

if(NOT LIBRARY_ONLY)
  set_target_properties(minisat_simp       PROPERTIES OUTPUT_NAME "minisat")
endif()

#--------------------------------------------------------------------------------------------------
# Installation targets:

install(TARGETS minisat-lib-static
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)

if(NOT WASI)
  install(TARGETS minisat-lib-shared
          RUNTIME DESTINATION bin
          LIBRARY DESTINATION lib
          ARCHIVE DESTINATION lib)
endif()

if(NOT LIBRARY_ONLY)
  install(TARGETS minisat_core minisat_simp
          RUNTIME DESTINATION bin
          LIBRARY DESTINATION lib
          ARCHIVE DESTINATION lib)
endif()

install(DIRECTORY minisat/mtl minisat/utils minisat/core minisat/simp
        DESTINATION include/minisat
        FILES_MATCHING PATTERN "*.h")
