cmake_minimum_required(VERSION 3.15)
project(hsql VERSION 1.0 LANGUAGES CXX)

add_compile_definitions(-D_CRT_SECURE_NO_WARNINGS)

# Options configuration
option(HSQL_BUILD_TESTS "Build tests" OFF)
option(HSQL_BUILD_BENCHMARKS "Build benchmarks" OFF)
option(HSQL_BUILD_EXAMPLES "Build examples" OFF)
option(HSQL_ENABLE_WERROR "Treat warnings as errors" ON)
option(HSQL_ENABLE_RELAXED "Disable compiler warnings" OFF)
option(BUILD_SHARED_LIBS "Build shared library" ON)

# find_package(FLEX REQUIRED)
# find_package(BISON REQUIRED)

# Configure compiler flags
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT HSQL_ENABLE_RELAXED)
    add_compile_options(-Wall -Wextra)
    if(HSQL_ENABLE_WERROR)
        add_compile_options(-Werror)
    endif()
endif()

# Generate parser files
# bison_target(
#     BisonParser
#     src/parser/bison_parser.y
#     ${CMAKE_CURRENT_BINARY_DIR}/bison_parser.cpp
#     COMPILE_FLAGS "--defines=${CMAKE_CURRENT_BINARY_DIR}/bison_parser.h"
# )

# flex_target(
#     FlexLexer
#     src/parser/flex_lexer.l
#     ${CMAKE_CURRENT_BINARY_DIR}/flex_lexer.cpp
# )

set(BisonParser_OUTPUTS "src/parser/bison_parser.cpp")
set(FlexLexer_OUTPUTS "src/parser/flex_lexer.cpp")

# Configure parser warnings suppression
set_property(SOURCE ${BisonParser_OUTPUTS}
    PROPERTY COMPILE_FLAGS "-Wno-unused-but-set-variable -Wno-deprecated-declarations -Wno-unused-parameter -Wno-unused-function"
)
set_property(SOURCE ${FlexLexer_OUTPUTS} 
    PROPERTY COMPILE_FLAGS "-Wno-sign-compare -Wno-unneeded-internal-declaration -Wno-register -Wno-deprecated-declarations"
)

# Collect source files
file(GLOB_RECURSE LIB_SOURCES
    "src/*.cpp"
    "src/sql/*.cpp"
    "src/util/*.cpp"
)
list(FILTER LIB_SOURCES EXCLUDE REGEX "src/parser/.*\\.cpp")

add_library(sqlparser
    ${LIB_SOURCES}
    ${BisonParser_OUTPUTS}
    ${FlexLexer_OUTPUTS}
)

target_include_directories(sqlparser PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    $<INSTALL_INTERFACE:include/hsql>
)

# Install configuration
install(TARGETS sqlparser
    EXPORT sqlparser-targets
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)

install(DIRECTORY src/
    DESTINATION include/hsql
    FILES_MATCHING 
    PATTERN "*.h"
    PATTERN "parser" EXCLUDE
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/bison_parser.h
    ${CMAKE_CURRENT_BINARY_DIR}/flex_lexer.h
    DESTINATION include/hsql/parser
)

install(EXPORT sqlparser-targets
    FILE sqlparser-config.cmake
    NAMESPACE hsql::
    DESTINATION lib/cmake/sqlparser
)

# Tests
if(HSQL_BUILD_TESTS)
    enable_testing()
    file(GLOB TEST_SOURCES test/*.cpp)
    add_executable(hsql-tests ${TEST_SOURCES})
    target_link_libraries(hsql-tests PRIVATE sqlparser)
    target_include_directories(hsql-tests PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/test
    )

    add_test(NAME sqlparser-tests COMMAND hsql-tests)
endif()

# Benchmarks
if(HSQL_BUILD_BENCHMARKS)
    find_package(benchmark REQUIRED)
    
    file(GLOB BENCHMARK_SOURCES benchmark/*.cpp)
    add_executable(hsql-benchmarks ${BENCHMARK_SOURCES})
    target_link_libraries(hsql-benchmarks PRIVATE
        sqlparser
        benchmark::benchmark
    )
endif()

# Examples
if(HSQL_BUILD_EXAMPLES)
    add_executable(hsql-example example/example.cpp)
    target_link_libraries(hsql-example PRIVATE sqlparser)
endif()