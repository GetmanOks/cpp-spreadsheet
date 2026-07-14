cmake_minimum_required(VERSION 3.20)
project(spreadsheet LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

include(FetchContent)

set(ANTLR_VERSION 4.13.2)
set(ANTLR_EXECUTABLE
    "${CMAKE_BINARY_DIR}/antlr-${ANTLR_VERSION}-complete.jar"
    CACHE FILEPATH "Path to the ANTLR generator"
)

if(NOT EXISTS "${ANTLR_EXECUTABLE}")
    file(
        DOWNLOAD
        "https://www.antlr.org/download/antlr-${ANTLR_VERSION}-complete.jar"
        "${ANTLR_EXECUTABLE}"
        EXPECTED_HASH
        "SHA256=eae2dfa119a64327444672aff63e9ec35a20180dc5b8090b7a6ab85125df4d76"
        TLS_VERIFY ON
        STATUS ANTLR_DOWNLOAD_STATUS
    )

    list(GET ANTLR_DOWNLOAD_STATUS 0 ANTLR_DOWNLOAD_RESULT)
    if(NOT ANTLR_DOWNLOAD_RESULT EQUAL 0)
        list(GET ANTLR_DOWNLOAD_STATUS 1 ANTLR_DOWNLOAD_ERROR)
        message(FATAL_ERROR "Failed to download ANTLR: ${ANTLR_DOWNLOAD_ERROR}")
    endif()
endif()

set(ANTLR_BUILD_CPP_TESTS OFF CACHE BOOL "" FORCE)
set(ANTLR_BUILD_SHARED OFF CACHE BOOL "" FORCE)
set(DISABLE_WARNINGS ON CACHE BOOL "" FORCE)
set(WITH_STATIC_CRT OFF CACHE BOOL "" FORCE)

FetchContent_Declare(
    antlr
    GIT_REPOSITORY https://github.com/antlr/antlr4.git
    GIT_TAG ${ANTLR_VERSION}
    GIT_SHALLOW TRUE
    SOURCE_SUBDIR runtime/Cpp
)
FetchContent_MakeAvailable(antlr)

include(${CMAKE_CURRENT_SOURCE_DIR}/FindANTLR.cmake)
antlr_target(FormulaParser Formula.g4 LEXER PARSER LISTENER)

add_executable(
    spreadsheet
    ${ANTLR_FormulaParser_CXX_OUTPUTS}
    FormulaAST.cpp
    cell.cpp
    formula.cpp
    main.cpp
    sheet.cpp
    structures.cpp
)

target_include_directories(
    spreadsheet
    PRIVATE
        ${ANTLR_FormulaParser_OUTPUT_DIR}
)

target_compile_definitions(
    spreadsheet
    PRIVATE
        ANTLR4CPP_STATIC
        _SILENCE_ALL_CXX17_DEPRECATION_WARNINGS
)

target_link_libraries(spreadsheet PRIVATE antlr4_static)

if(MSVC)
    # ANTLR 4.13.2 uses std::chrono in ProfilingATNSimulator.cpp without
    # including <chrono>. Recent MSVC versions no longer provide it transitively.
    target_compile_options(antlr4_static PRIVATE /FIchrono /W0)
    target_compile_options(spreadsheet PRIVATE /W4 /permissive-)
else()
    target_compile_options(spreadsheet PRIVATE -Wall -Wextra -Wpedantic)
endif()

enable_testing()
add_test(NAME spreadsheet_tests COMMAND spreadsheet)

install(TARGETS spreadsheet RUNTIME DESTINATION bin)

set_directory_properties(PROPERTIES VS_STARTUP_PROJECT spreadsheet)