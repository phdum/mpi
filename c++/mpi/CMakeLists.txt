# MPI C++ Interface is header-only
# Create an Interface Target
file(GLOB_RECURSE headers *.hpp)
add_library(${PROJECT_NAME}_c INTERFACE)
add_library(${PROJECT_NAME}::${PROJECT_NAME}_c ALIAS ${PROJECT_NAME}_c)

# Set C++ Standard
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++20" HAS_CPP20)
if(HAS_CPP20 AND cxx_std_20 IN_LIST CMAKE_CXX_COMPILE_FEATURES)
  target_compile_features(${PROJECT_NAME}_c INTERFACE cxx_std_20)
else()
  target_compile_features(${PROJECT_NAME}_c INTERFACE cxx_std_17)
  target_compile_options(${PROJECT_NAME}_c INTERFACE $<$<CXX_COMPILER_ID:GNU>:-fconcepts>)
endif()

# Configure target and compilation
target_include_directories(${PROJECT_NAME}_c INTERFACE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/c++>)
target_include_directories(${PROJECT_NAME}_c SYSTEM INTERFACE $<INSTALL_INTERFACE:${CMAKE_INSTALL_PREFIX}/include>)
target_compile_definitions(${PROJECT_NAME}_c INTERFACE
				MPI_GIT_HASH=${PROJECT_GIT_HASH}
				$<$<CONFIG:Debug>:MPI_DEBUG>
			  )

# Install library and headers
install(TARGETS ${PROJECT_NAME}_c EXPORT ${PROJECT_NAME}-targets DESTINATION lib)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} DESTINATION include FILES_MATCHING PATTERN "*.hpp" PATTERN "*.h")

# ========= Additional Depdencies ==========

# Link against Itertools
target_link_libraries(${PROJECT_NAME}_c INTERFACE itertools::itertools_c)


message(STATUS "-------- MPI detection -------------")

set(MPI_CXX_SKIP_MPICXX TRUE CACHE BOOL "If true, the MPI-2 C++ bindings are disabled using definitions.")
find_package(MPI REQUIRED COMPONENTS CXX)

# Create an interface target
add_library(mpi INTERFACE)
target_link_libraries(mpi INTERFACE ${MPI_CXX_LIBRARIES})
target_include_directories(mpi SYSTEM INTERFACE ${MPI_CXX_INCLUDE_DIRS})
#target_link_options(mpi INTERFACE ${MPI_CXX_LINK_FLAGS}) cmake 3.13+
target_compile_options(mpi INTERFACE ${MPI_CXX_COMPILE_OPTIONS})
target_compile_definitions(mpi INTERFACE ${MPI_CXX_COMPILE_DEFINITIONS})

# Backward compatibility for older FindMPI.cmake (without MPI_CXX_SKIP_MPICXX)
if(CMAKE_VERSION VERSION_LESS 3.10.0)
    # Stop OpenMPI from including cxx binding headers, if installed
    target_compile_definitions(mpi INTERFACE MPICH_SKIP_MPICXX OMPI_SKIP_MPICXX _MPICC_H)
endif()

# Export Interface target
target_link_libraries(${PROJECT_NAME}_c INTERFACE mpi)
install(TARGETS mpi EXPORT ${PROJECT_NAME}-targets)

# Backward compatibility for older FindMPI.cmake
if(NOT MPIEXEC_EXECUTABLE)
  set(MPIEXEC_EXECUTABLE ${MPIEXEC} CACHE FILENAME "MPI Executable")
endif()

# Compatibility to Open-MPI 3.0.0: check whether MPI executable has option --oversubscribe and add it
execute_process(COMMAND ${MPIEXEC_EXECUTABLE} --oversubscribe ${MPIEXEC_NUMPROC_FLAG} 4 ${MPIEXEC_PREFLAGS} ls ${MPIEXEC_POSTFLAGS} RESULT_VARIABLE HAS_NO_OVERSUBSCRIBE OUTPUT_QUIET ERROR_QUIET)
if(NOT HAS_NO_OVERSUBSCRIBE)
  list(APPEND MPIEXEC_PREFLAGS --oversubscribe)
  set(MPIEXEC_PREFLAGS ${MPIEXEC_PREFLAGS} CACHE STRING "These flags will be directly before the executable that is being run by mpiexec." FORCE)
endif()

# ========= Static Analyzer Checks ==========

option(ANALYZE_SOURCES OFF "Run static analyzer checks if found (clang-tidy, cppcheck)")
if(ANALYZE_SOURCES)

  # Locate static analyzer tools
  find_program(CPPCHECK_EXECUTABLE NAMES "cppcheck" PATHS ENV PATH)
  find_program(CLANG_TIDY_EXECUTABLE NAMES "clang-tidy" PATHS ENV PATH)

  # Run clang-tidy if found
  if(CLANG_TIDY_EXECUTABLE)
    message(STATUS "clang-tidy found: ${CLANG_TIDY_EXECUTABLE}")
    set_target_properties(${PROJECT_NAME}_c PROPERTIES CXX_CLANG_TIDY "${CLANG_TIDY_EXECUTABLE}")
  else()
    message(STATUS "clang-tidy not found in $PATH. Please consider installing clang-tidy for additional checks!")
  endif()

  # Run cppcheck if found
  if(CPPCHECK_EXECUTABLE)
    message(STATUS "cppcheck found: ${CPPCHECK_EXECUTABLE}")
    add_custom_command(
	TARGET ${PROJECT_NAME}_c
	COMMAND ${CPPCHECK_EXECUTABLE}
	  --enable=warning,style,performance,portability
	  --std=c++17
	  --template=gcc
	  --verbose
	  --force
	  --quiet
          ${headers}
	WORKING_DIRECTORY
	  ${CMAKE_CURRENT_SOURCE_DIR}
    )
  else()
    message(STATUS "cppcheck not found in $PATH. Please consider installing cppcheck for additional checks!")
  endif()

endif()


# ========= Dynamic Analyzer Checks ==========

option(ASAN OFF "Compile library and executables with LLVM Address Sanitizer")
option(UBSAN OFF "Compile library and executables with LLVM Undefined Behavior Sanitizer")

if(ASAN)
  if(NOT TARGET asan)
    find_package(sanitizer REQUIRED "asan")
  endif()
  target_link_libraries(${PROJECT_NAME}_c INTERFACE $<BUILD_INTERFACE:asan>)
endif()
if(UBSAN)
  if(NOT TARGET ubsan)
    find_package(sanitizer REQUIRED "ubsan")
  endif()
  target_link_libraries(${PROJECT_NAME}_c INTERFACE $<BUILD_INTERFACE:ubsan>)
endif()
