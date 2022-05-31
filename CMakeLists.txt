cmake_minimum_required(VERSION 3.10)
project(zone LANGUAGES C VERSION 0.0.1 DESCRIPTION "Fast and standards compliant DNS zone file parser")

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/Modules")

if(MSVC)
  add_compile_options(/W4 /WX)
else()
  add_compile_options(-Wall -Wextra -pedantic -Werror)
endif()

# FIXME: types.h can be generated eventually
add_library(zone SHARED src/zone.c src/scanner.c src/types.h)
target_include_directories(zone PUBLIC include)
set_target_properties(zone PROPERTIES C_STANDARD_REQUIRED 11)

option(BUILD_TESTING "Build the testing tree." OFF)

include(CTest)

if(BUILD_TESTING)
  add_subdirectory(tests)
endif()