# Define CMake version, project name, and C++ standard
cmake_minimum_required(VERSION 3.13)
project(Cloak)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Must include to avoid failing on Linux with std::thread
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

# Import Packages
find_package(GTest REQUIRED) # Unit-Testing Library
# find_package(benchmark REQUIRED) # Benchmarking Library
find_package(spdlog CONFIG REQUIRED) # Logging Library

# Include Library Directories
include_directories(${GTEST_INCLUDE_DIR})
include_directories(include)

# Options
option(CODE_COVERAGE "Enable Code Coverage" OFF)
option(BUILD_TESTS "Build Tests" ON)
option(BENCH_TESTS "Build Benchmarks" OFF)
option(LOGGING "Enable Logging" ON) # Disable on Final Release

# If CODE_COVERAGE has been enabled, then update compilation flags
if(CODE_COVERAGE)
    message(STATUS "Enabling Code Coverage")
endif()

if (USE_SSE)
    add_compile_definitions(USE_SSE)
    add_compile_definitions(__BMI2__)
    message("-- Compiling using SSE")
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wshadow -pedantic -march=native") # -Werror
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -O3") # -O3
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3")
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4 /WX")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /O3")
endif(CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU")

# Put all source files excluding main.cpp into library source variable
set(CLOAK_LIB_SRCS
    src/attack.cpp
    src/bitmask.cpp
    src/board.cpp
    src/chess_clock.cpp
    src/chess_hash.cpp
    src/chess_move.cpp
    src/eval.cpp
    src/magic.cpp
    src/move_gen.cpp
    src/protocol.cpp
    src/search.cpp
    src/string_manip.cpp
    src/uci_options.cpp
    src/transposition_table.cpp
    src/book.cpp
)

# Create library, link external libraries against it
add_library(Cloak-lib ${CLOAK_LIB_SRCS})
target_include_directories(Cloak-lib PUBLIC ${CMAKE_SOURCE_DIR}/include ${CMAKE_BINARY_DIR}/include ${GTEST_INCLUDE_DIR} ${SPDLOG_INCLUDE_DIR})

# Create Main Executable
add_executable(cloak src/main.cpp)
target_link_libraries(cloak Cloak-lib Threads::Threads spdlog::spdlog)
install(TARGETS cloak DESTINATION bin)
install(TARGETS Cloak-lib ARCHIVE DESTINATION lib LIBRARY DESTINATION lib RUNTIME DESTINATION bin)

# If we want to build a test, then build them
if(BUILD_TESTS)
    add_subdirectory(test)
    add_subdirectory(tools)
    add_subdirectory(benchmark)

    # If we want to do code coverage, then do it
    if (CODE_COVERAGE MATCHES ON)
        include(cmake/CodeCoverage.cmake)
        target_link_libraries(Cloak-lib --coverage)
        append_coverage_compiler_flags()

        setup_target_for_coverage_lcov(NAME BitmaskCoverage EXECUTABLE BitmaskTestRunner BASE_DIRECTORY "src")
        setup_target_for_coverage_lcov(NAME BoardCoverage EXECUTABLE BoardTestRunner BASE_DIRECTORY "src" EXECUTABLE_ARGS data/board-test/FullBoardTest.fen)
        setup_target_for_coverage_lcov(NAME ChessClockCoverage EXECUTABLE ChessClockTestRunner BASE_DIRECTORY "src")
        setup_target_for_coverage_lcov(NAME ChessMoveCoverage EXECUTABLE ChessMoveTestRunner BASE_DIRECTORY "src")
        setup_target_for_coverage_lcov(NAME StringManipCoverage EXECUTABLE StringManipTestRunner BASE_DIRECTORY "src")
        setup_target_for_coverage_lcov(NAME UCIOptionsCoverage EXECUTABLE UCIOptionsTestRunner BASE_DIRECTORY "src")
        setup_target_for_coverage_lcov(NAME PerftCoverage EXECUTABLE PerftTestRunner BASE_DIRECTORY "src" EXECUTABLE_ARGS data/perft/coverage.perft)
        setup_target_for_coverage_lcov(NAME EvalCoverage EXECUTABLE EvalTestRunner BASE_DIRECTORY "src" EXECUTABLE_ARGS data/eval-test/FullEvalTest.fen data/eval-test/NearlyFullEvalTest.fen)
        setup_target_for_coverage_lcov(NAME SearchCoverage EXECUTABLE SearchTestRunner BASE_DIRECTORY "src" EXECUTABLE_ARGS data/puzzles/mate-in-2.puz)
    endif()
endif()