cmake_minimum_required(VERSION 3.14)

project(fit_sdk LANGUAGES CXX)

# Build the examples by default only when this is the top-level project,
# not when the SDK is pulled in with add_subdirectory() or FetchContent.
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(FIT_SDK_TOP_LEVEL ON)
else()
    set(FIT_SDK_TOP_LEVEL OFF)
endif()

option(FIT_SDK_BUILD_EXAMPLES "Build the FIT SDK examples" ${FIT_SDK_TOP_LEVEL})

# The sources in src/ are regenerated with each SDK release, so collect them
# with a glob instead of maintaining a file list by hand.
file(GLOB FIT_SDK_SOURCES CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/plugins/*.cpp")

# Always static: the SDK has no symbol export annotations, so a shared
# library would export nothing on Windows.
add_library(fit_sdk STATIC ${FIT_SDK_SOURCES})
add_library(fit::sdk ALIAS fit_sdk)

set(FIT_SDK_INCLUDE_DIRS
    "${CMAKE_CURRENT_SOURCE_DIR}/src"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/plugins")

# Consumers see the headers as system headers so that warnings from the SDK
# do not break projects that build with -Werror.
target_include_directories(fit_sdk PRIVATE ${FIT_SDK_INCLUDE_DIRS})
target_include_directories(fit_sdk SYSTEM INTERFACE ${FIT_SDK_INCLUDE_DIRS})

target_compile_features(fit_sdk PUBLIC cxx_std_11)

if(FIT_SDK_BUILD_EXAMPLES)
    add_executable(fit_sdk_decode examples/decode/decode.cpp)
    set_target_properties(fit_sdk_decode PROPERTIES OUTPUT_NAME decode)
    target_link_libraries(fit_sdk_decode PRIVATE fit::sdk)

    add_executable(fit_sdk_encode examples/encode/encode.cpp)
    set_target_properties(fit_sdk_encode PROPERTIES OUTPUT_NAME encode)
    target_link_libraries(fit_sdk_encode PRIVATE fit::sdk)

    # Round trip: encode writes ExampleActivity.fit, which decode then reads back.
    enable_testing()
    add_test(NAME fit_sdk_encode COMMAND fit_sdk_encode)
    add_test(NAME fit_sdk_decode COMMAND fit_sdk_decode ExampleActivity.fit)
    set_tests_properties(fit_sdk_encode PROPERTIES FIXTURES_SETUP fit_sdk_example_activity)
    set_tests_properties(fit_sdk_decode PROPERTIES FIXTURES_REQUIRED fit_sdk_example_activity)
endif()
