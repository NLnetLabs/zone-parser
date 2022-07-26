cmake_minimum_required(VERSION 3.10)
project(zone LANGUAGES C VERSION 0.0.1 DESCRIPTION "Fast and standards compliant DNS zone file parser")

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/Modules")

if(MSVC)
  add_compile_options(/W4 /WX)
else()
  add_compile_options(-Wall -Wextra -pedantic -Werror)
endif()


set(ANALYZER "" CACHE STRING "Analyzer to enable on the build.")
if(ANALYZER)
  # GCC and Visual Studio offer builtin analyzers. Clang supports static
  # analysis through separate tools, e.g. Clang-Tidy, which can be used in
  # conjunction with other compilers too. Specifying -DANALYZER=on enables
  # the builtin analyzer for the compiler, enabling clang-tidy in case of
  # Clang. Specifying -DANALYZER=clang-tidy always enables clang-tidy.
  string(REPLACE " " "" ANALYZER "${ANALYZER}")
  string(TOLOWER "${ANALYZER}" ANALYZER)
  if(CMAKE_C_COMPILER_ID MATCHES "Clang" AND ANALYZER STREQUAL "on")
    set(ANALYZER "clang-tidy")
  endif()

  if(ANALYZER STREQUAL "clang-tidy")
    # Clang-Tidy is an extensible tool that offers more than static analysis.
    # https://clang.llvm.org/extra/clang-tidy/checks/list.html
    message(STATUS "Enabling analyzer: clang-tidy")
    set(CMAKE_C_CLANG_TIDY "clang-tidy;-checks=-*,clang-analyzer-*,-clang-analyzer-security.insecureAPI.strcpy,-clang-analyzer-security.insecureAPI.DeprecatedOrUnsafeBufferHandling")
    if(WERROR)
      set(CMAKE_C_CLANG_TIDY "${CMAKE_C_CLANG_TIDY};--warnings-as-errors=*")
    endif()
  elseif(ANALYZER STREQUAL "on")
    if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
      if(CMAKE_C_COMPILER_VERSION VERSION_GREATER_EQUAL "10")
        message(STATUS "Enabling analyzer: GCC")
        # -Wanalyzer-malloc-leak generates lots of false positives
        add_compile_options(-fanalyzer -Wno-analyzer-malloc-leak)
      endif()
    endif()
  endif()
endif()


set(SANITIZER "" CACHE STRING "Sanitizers to enable on the build.")
if(SANITIZER)
  string(REGEX REPLACE " " "" SANITIZER "${SANITIZER}")
  string(REGEX REPLACE "[,;]+" ";" SANITIZER "${SANITIZER}")
  foreach(san ${SANITIZER})
    if(san STREQUAL "address")
      add_compile_options("-fno-omit-frame-pointer")
      add_link_options("-fno-omit-frame-pointer")
    endif()
    if(san AND NOT san STREQUAL "none")
      message(STATUS "Enabling sanitizer: ${san}")
      add_compile_options("-fsanitize=${san}")
      add_link_options("-fsanitize=${san}")
    endif()
  endforeach()
endif()

# generate types from dnsextlang stanzas
add_custom_command(
  OUTPUT src/types.h
  COMMAND "${CMAKE_COMMAND}"
  ARGS
    "-DSTANZAS=${CMAKE_CURRENT_SOURCE_DIR}/src/type.stanzas"
    "-DTEMPLATE=${CMAKE_CURRENT_SOURCE_DIR}/src/types.h.in"
    "-DHEADER=src/types.h"
    "-P" "${CMAKE_CURRENT_SOURCE_DIR}/dnsextlang.cmake"
  DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/type.stanzas"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/types.h.in"
    "${CMAKE_CURRENT_SOURCE_DIR}/dnsextlang.cmake")

# generate descriptors from dnsextlang stanzas
add_custom_command(
  OUTPUT src/grammar.h
  COMMAND "${CMAKE_COMMAND}"
  ARGS
    "-DSTANZAS=${CMAKE_CURRENT_SOURCE_DIR}/src/type.stanzas"
    "-DTEMPLATE=${CMAKE_CURRENT_SOURCE_DIR}/src/grammar.h.in"
    "-DHEADER=src/grammar.h"
    "-P" "${CMAKE_CURRENT_SOURCE_DIR}/dnsextlang.cmake"
  DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/type.stanzas"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/grammar.h.in"
    "${CMAKE_CURRENT_SOURCE_DIR}/dnsextlang.cmake")

# FIXME: scanner-types.h can be generated eventually
add_library(zone STATIC
  src/zone.c
  src/scanner.h
  src/base16.h
  src/base64.h
  src/nsec.h
  src/wks.h
  src/parser.c
  "${CMAKE_CURRENT_BINARY_DIR}/src/types.h"
  "${CMAKE_CURRENT_BINARY_DIR}/src/grammar.h")
target_include_directories(
  zone PUBLIC include $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
                      $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/src>)
set_target_properties(zone PROPERTIES C_STANDARD_REQUIRED 11)
target_link_libraries(zone)

add_executable(zone-parser src/main.c)
target_link_libraries(zone-parser PRIVATE zone)

option(BUILD_TESTING "Build the testing tree." OFF)

include(CTest)

if(BUILD_TESTING)
  add_subdirectory(tests)
endif()
