cmake_minimum_required(VERSION 3.14)

include(cmake/PreventInSourceBuilds.cmake)

# ---- Initialize Project ----

# Used to support find_package
set(package_name "cpptrace")

project(
  cpptrace
  VERSION 0.5.2
  DESCRIPTION "Simple, portable, and self-contained stacktrace library for C++11 and newer "
  HOMEPAGE_URL "https://github.com/jeremy-rifkin/cpptrace"
  LANGUAGES C CXX
)

# Don't change include order, OptionVariables checks if project is top level
include(cmake/ProjectIsTopLevel.cmake)
include(cmake/OptionVariables.cmake)

include(GNUInstallDirs)
include(CheckCXXSourceCompiles)
include(CheckCXXCompilerFlag)

if(PROJECT_IS_TOP_LEVEL)
  find_program(CCACHE_FOUND ccache)
  if(CCACHE_FOUND)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
  endif()
endif()

if(PROJECT_IS_TOP_LEVEL)
  if(CMAKE_GENERATOR STREQUAL "Ninja")
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdiagnostics-color=always")
    elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "AppleClang")
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcolor-diagnostics")
    endif()
    if("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
      SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fdiagnostics-color=always")
    elseif("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "AppleClang")
      SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fcolor-diagnostics")
    endif()
  endif()
endif()

if(CPPTRACE_SANITIZER_BUILD)
  add_compile_options(-fsanitize=address)
  add_link_options(-fsanitize=address)
endif()

if(NOT "${CPPTRACE_BACKTRACE_PATH}" STREQUAL "")
  # quotes used over <> because of a macro substitution issue where
  # </usr/lib/gcc/x86_64-linux-gnu/12/include/backtrace.h>
  # is expanded to
  # </usr/lib/gcc/x86_64-1-gnu/12/include/backtrace.h>
  string(CONCAT CPPTRACE_BACKTRACE_PATH "\"" ${CPPTRACE_BACKTRACE_PATH})
  string(CONCAT CPPTRACE_BACKTRACE_PATH ${CPPTRACE_BACKTRACE_PATH} "\"")
  #message(STATUS ${CPPTRACE_BACKTRACE_PATH})
  string(CONCAT CPPTRACE_BACKTRACE_PATH_DEFINITION "-DCPPTRACE_BACKTRACE_PATH=" ${CPPTRACE_BACKTRACE_PATH})
  #message(STATUS ${CPPTRACE_BACKTRACE_PATH_DEFINITION})
else()
  set(CPPTRACE_BACKTRACE_PATH_DEFINITION "")
endif()

# =============================================== Platform Support ===============================================
function(check_support var source includes libraries definitions)
  set(CMAKE_REQUIRED_INCLUDES "${includes}")
  list(APPEND CMAKE_REQUIRED_INCLUDES "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
  set(CMAKE_REQUIRED_LIBRARIES "${libraries}")
  set(CMAKE_REQUIRED_DEFINITIONS "${definitions}")
  string(CONCAT full_source "#include \"${source}\"" ${nonce})
  check_cxx_source_compiles(${full_source} ${var})
  set(${var} ${${var}} PARENT_SCOPE)
endfunction()

if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  check_support(HAS_CXXABI has_cxxabi.cpp "" "" "")
endif()

if(NOT WIN32)
  check_support(HAS_UNWIND has_unwind.cpp "" "" "")
  check_support(HAS_EXECINFO has_execinfo.cpp "" "" "")
  check_support(HAS_BACKTRACE has_backtrace.cpp "" "backtrace" "${CPPTRACE_BACKTRACE_PATH_DEFINITION}")
  set(STACKTRACE_LINK_LIB "stdc++_libbacktrace")
else()
  check_support(HAS_STACKWALK has_stackwalk.cpp "" "dbghelp" "")
endif()

if(NOT WIN32 OR MINGW)
  check_support(HAS_CXX_EXCEPTION_TYPE has_cxx_exception_type.cpp "" "" "")
endif()

if(UNIX AND NOT APPLE)
  check_support(HAS_DL_FIND_OBJECT has_dl_find_object.cpp "" "" "")
endif()

# =============================================== Autoconfig unwinding ===============================================
# Unwind back-ends
if(
  NOT (
    CPPTRACE_UNWIND_WITH_UNWIND OR
    CPPTRACE_UNWIND_WITH_LIBUNWIND OR
    CPPTRACE_UNWIND_WITH_EXECINFO OR
    CPPTRACE_UNWIND_WITH_WINAPI OR
    CPPTRACE_UNWIND_WITH_DBGHELP OR
    CPPTRACE_UNWIND_WITH_NOTHING
  )
)
  # Attempt to auto-config
  if(UNIX)
    if(HAS_UNWIND)
      set(CPPTRACE_UNWIND_WITH_UNWIND On)
      message(STATUS "Cpptrace auto config: Using libgcc unwind for unwinding")
    elseif(HAS_EXECINFO)
      set(CPPTRACE_UNWIND_WITH_EXECINFO On)
      message(STATUS "Cpptrace auto config: Using execinfo.h for unwinding")
    else()
      set(CPPTRACE_UNWIND_WITH_NOTHING On)
      message(FATAL_ERROR "Cpptrace auto config: No unwinding back-end seems to be supported, stack tracing will not work. To compile anyway set CPPTRACE_UNWIND_WITH_NOTHING.")
    endif()
  elseif(MINGW OR WIN32)
    if(HAS_STACKWALK)
      set(CPPTRACE_UNWIND_WITH_DBGHELP On)
      message(STATUS "Cpptrace auto config: Using dbghelp for unwinding")
    else()
      set(CPPTRACE_UNWIND_WITH_WINAPI On)
      message(STATUS "Cpptrace auto config: Using winapi for unwinding")
    endif()
  endif()
else()
  #message(STATUS "MANUAL CONFIG SPECIFIED")
endif()

# =============================================== Autoconfig symbols ===============================================
if(
  NOT (
    CPPTRACE_GET_SYMBOLS_WITH_LIBBACKTRACE OR
    CPPTRACE_GET_SYMBOLS_WITH_LIBDL OR
    CPPTRACE_GET_SYMBOLS_WITH_ADDR2LINE OR
    CPPTRACE_GET_SYMBOLS_WITH_LIBDWARF OR
    CPPTRACE_GET_SYMBOLS_WITH_DBGHELP OR
    CPPTRACE_GET_SYMBOLS_WITH_NOTHING
  )
)
  if(UNIX)
    message(STATUS "Cpptrace auto config: Using libdwarf for symbols")
    set(CPPTRACE_GET_SYMBOLS_WITH_LIBDWARF On)
  elseif(MINGW)
    message(STATUS "Cpptrace auto config: Using libdwarf + dbghelp for symbols")
    # Use both dbghelp and libdwarf under mingw: Some files may use pdb symbols, e.g. system dlls like KERNEL32.dll and
    # ntdll.dll at the very least, but also other libraries linked with may have pdb symbols.
    set(CPPTRACE_GET_SYMBOLS_WITH_LIBDWARF On)
    set(CPPTRACE_GET_SYMBOLS_WITH_DBGHELP On)
  else()
    message(STATUS "Cpptrace auto config: Using dbghelp for symbols")
    set(CPPTRACE_GET_SYMBOLS_WITH_DBGHELP On)
  endif()
endif()

# =============================================== Autoconfig demangling ===============================================
# Handle demangle configuration
if(
  NOT (
    CPPTRACE_DEMANGLE_WITH_CXXABI OR
    CPPTRACE_DEMANGLE_WITH_WINAPI OR
    CPPTRACE_DEMANGLE_WITH_NOTHING
  )
)
  if(HAS_CXXABI)
    message(STATUS "Cpptrace auto config: Using cxxabi for demangling")
    set(CPPTRACE_DEMANGLE_WITH_CXXABI On)
  elseif(WIN32 AND NOT MINGW)
    message(STATUS "Cpptrace auto config: Using dbghelp for demangling")
    set(CPPTRACE_DEMANGLE_WITH_WINAPI On)
  else()
    set(CPPTRACE_DEMANGLE_WITH_NOTHING On)
  endif()
else()
  #message(STATUS "Manual demangling back-end specified")
endif()

# =============================================== Now define the library ===============================================

# Target that we can modify (can't modify ALIAS targets)
# Target name should not be the same as ${PROJECT_NAME}, causes add_subdirectory issues
set(target_name "cpptrace-lib")
add_library(${target_name} ${build_type})

# Alias to cause error at configuration time instead of link time if target is missing
add_library(cpptrace::cpptrace ALIAS ${target_name})

# Add /include files to target
# This is solely for IDE benefit, doesn't affect building
target_sources(
    ${target_name} PRIVATE
    include/cpptrace/cpptrace.hpp
    include/ctrace/ctrace.h
)

# add /src files to target
target_sources(
    ${target_name} PRIVATE
    # src
    src/cpptrace.cpp
    src/ctrace.cpp
    src/demangle/demangle_with_cxxabi.cpp
    src/demangle/demangle_with_winapi.cpp
    src/demangle/demangle_with_nothing.cpp
    src/symbols/symbols_with_addr2line.cpp
    src/symbols/symbols_with_dbghelp.cpp
    src/symbols/symbols_with_dl.cpp
    src/symbols/symbols_with_libbacktrace.cpp
    src/symbols/symbols_with_libdwarf.cpp
    src/symbols/symbols_with_nothing.cpp
    src/symbols/symbols_core.cpp
    src/unwind/unwind_with_execinfo.cpp
    src/unwind/unwind_with_libunwind.cpp
    src/unwind/unwind_with_nothing.cpp
    src/unwind/unwind_with_unwind.cpp
    src/unwind/unwind_with_winapi.cpp
    src/unwind/unwind_with_dbghelp.cpp
    src/snippets/snippet.cpp
)

target_include_directories(
  ${target_name}
  PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/cpptrace/>
)

target_compile_options(
  ${target_name}
  PRIVATE
  $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall -Wextra -Werror=return-type -Wundef>
  $<$<CXX_COMPILER_ID:GNU>:-Wuseless-cast>
  $<$<CXX_COMPILER_ID:MSVC>:/W4 /WX /permissive->
)

if(CPPTRACE_WERROR_BUILD)
  target_compile_options(
    ${target_name}
    PRIVATE
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Werror>
    $<$<CXX_COMPILER_ID:MSVC>:/WX>
  )
endif()

# ---- Generate Build Info Headers ----

if(build_type STREQUAL "STATIC")
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_STATIC_DEFINE)
  set(CPPTRACE_STATIC_DEFINE TRUE)
endif()

# ---- Library Properties ----

# Hide all symbols by default
# Use SameMajorVersion versioning for shared library runtime linker lookup
set_target_properties(
    ${target_name} PROPERTIES
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN YES
    VERSION "${PROJECT_VERSION}"
    SOVERSION "${PROJECT_VERSION_MAJOR}"
    EXPORT_NAME "cpptrace"
    OUTPUT_NAME "cpptrace"
    POSITION_INDEPENDENT_CODE ${CPPTRACE_POSITION_INDEPENDENT_CODE}
)

# Header files generated by CMake
target_include_directories(
    ${target_name} SYSTEM PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>"
)

# Header files from /include
target_include_directories(
    ${target_name} ${warning_guard} PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
)

# Require C++11 support
target_compile_features(
    ${target_name}
    PRIVATE cxx_std_11
)

target_compile_definitions(${target_name} PRIVATE NOMINMAX)

if(CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
  SET(CMAKE_C_ARCHIVE_FINISH   "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
  SET(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
endif()

# =============================================== Apply options to build ===============================================

if(HAS_CXX_EXCEPTION_TYPE)
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_HAS_CXX_EXCEPTION_TYPE)
endif()

if(HAS_DL_FIND_OBJECT)
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_HAS_DL_FIND_OBJECT)
endif()

# Symbols
if(CPPTRACE_GET_SYMBOLS_WITH_LIBBACKTRACE)
  #if(NOT HAS_BACKTRACE)
  #  if(NOT "${CPPTRACE_BACKTRACE_PATH}" STREQUAL "")
  #    message(WARNING "Cpptrace: Using libbacktrace for symbols but libbacktrace doesn't appear installed or configured properly.")
  #  else()
  #    message(WARNING "Cpptrace: Using libbacktrace for symbols but libbacktrace doesn't appear installed or configured properly. You may need to specify CPPTRACE_BACKTRACE_PATH.")
  #  endif()
  #endif()
  #target_compile_definitions(${target_name} PUBLIC CPPTRACE_GET_SYMBOLS_WITH_LIBBACKTRACE)
  #target_link_libraries(${target_name} PRIVATE backtrace ${CMAKE_DL_LIBS})
  message(FATAL_ERROR "SpaRcle Common: only libdwarf is currently supported.")
endif()

if(CPPTRACE_GET_SYMBOLS_WITH_LIBDL)
  #target_compile_definitions(${target_name} PUBLIC CPPTRACE_GET_SYMBOLS_WITH_LIBDL)
  #target_link_libraries(${target_name} PRIVATE ${CMAKE_DL_LIBS})
  message(FATAL_ERROR "SpaRcle Common: only libdwarf is currently supported.")
endif()

if(CPPTRACE_GET_SYMBOLS_WITH_ADDR2LINE)
  ## set(CPPTRACE_ADDR2LINE_PATH "" CACHE STRING "Absolute path to the addr2line executable you want to use.")
  ## option(CPPTRACE_ADDR2LINE_SEARCH_SYSTEM_PATH "" OFF)
  #if(CPPTRACE_ADDR2LINE_SEARCH_SYSTEM_PATH)
  #  target_compile_definitions(${target_name} PUBLIC CPPTRACE_ADDR2LINE_SEARCH_SYSTEM_PATH)
  #else()
  #  if("${CPPTRACE_ADDR2LINE_PATH}" STREQUAL "")
  #    if(APPLE)
  #      find_program(CPPTRACE_ADDR2LINE_PATH_FINAL atos PATHS ENV PATH REQUIRED)
  #    else()
  #      find_program(CPPTRACE_ADDR2LINE_PATH_FINAL addr2line PATHS ENV PATH REQUIRED)
  #    endif()
  #  else()
  #    set(CPPTRACE_ADDR2LINE_PATH_FINAL "${CPPTRACE_ADDR2LINE_PATH}")
  #  endif()
  #  message(STATUS "Cpptrace: Using ${CPPTRACE_ADDR2LINE_PATH_FINAL} for addr2line path")
  #  target_compile_definitions(${target_name} PUBLIC CPPTRACE_ADDR2LINE_PATH="${CPPTRACE_ADDR2LINE_PATH_FINAL}")
  #endif()
  #target_compile_definitions(${target_name} PUBLIC CPPTRACE_GET_SYMBOLS_WITH_ADDR2LINE)
  #if(UNIX)
  #  target_link_libraries(${target_name} PRIVATE ${CMAKE_DL_LIBS})
  #endif()

  message(FATAL_ERROR "SpaRcle Common: only libdwarf is currently supported.")
endif()

if(CPPTRACE_GET_SYMBOLS_WITH_LIBDWARF)
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_GET_SYMBOLS_WITH_LIBDWARF)
  #if(CPPTRACE_USE_EXTERNAL_LIBDWARF)
  #  find_package(libdwarf REQUIRED)
  #else()
  #  include(FetchContent)
  #  # First, dependencies: Zstd and zlib (currently relying on system zlib)
  #  if(CPPTRACE_USE_EXTERNAL_ZSTD)
  #    find_package(zstd)
  #  else()
  #    cmake_policy(SET CMP0074 NEW)
  #    FetchContent_Declare(
  #        zstd
  #        GIT_REPOSITORY https://github.com/facebook/zstd.git
  #        GIT_TAG 63779c798237346c2b245c546c40b72a5a5913fe # v1.5.5
  #        GIT_SHALLOW 1
  #        SOURCE_SUBDIR build/cmake
  #    )
  #    # FetchContent_MakeAvailable(zstd)
  #    FetchContent_GetProperties(zstd)
  #    if(NOT zstd_POPULATED)
  #      FetchContent_Populate(zstd)
  #      set(ZSTD_BUILD_PROGRAMS OFF)
  #      set(ZSTD_BUILD_CONTRIB OFF)
  #      set(ZSTD_BUILD_TESTS OFF)
  #      set(ZSTD_BUILD_STATIC ON)
  #      set(ZSTD_BUILD_SHARED OFF)
  #      set(ZSTD_LEGACY_SUPPORT OFF)
  #      add_subdirectory("${zstd_SOURCE_DIR}/build/cmake" "${zstd_BINARY_DIR}")
  #    endif()
  #  endif()
  #  # Libdwarf itself
  #  set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)
  #  # set(PIC_ALWAYS TRUE)
  #  # set(BUILD_DWARFDUMP FALSE)
  #  FetchContent_Declare(
  #    libdwarf
  #    # GIT_REPOSITORY https://github.com/davea42/libdwarf-code.git
  #    # GIT_TAG 6216e185863f41d6f19ab850caabfff7326020d7 # v0.8.0
  #    # GIT_TAG 8b0bd09d8c77d45a68cb1bb00a54186a92b683d9 # v0.9.0
  #    # GIT_TAG 8cdcc531f310d1c5ae61da469d8056bdd36b77e7 # v0.9.1 + some cmake changes
  #    # Using a lightweight mirror that's optimized for clone + configure speed
  #    GIT_REPOSITORY https://github.com/jeremy-rifkin/libdwarf-lite.git
  #    # GIT_TAG c78e984f3abbd20f6e01d6f51819e826b1691f65 # v0.8.0
  #    # GIT_TAG 71090c680b4c943448ba87a0f1f864f174e4edda # v0.9.0
  #    GIT_TAG 5c0cb251f94b27e90184e6b2d9a0c9c62593babc # v0.9.1 + some cmake changes
  #    # GIT_REPOSITORY https://github.com/jeremy-rifkin/libdwarf-code.git
  #    # GIT_TAG 308b55331b564d4fdbe3bc6856712270e5b2395b
  #    GIT_SHALLOW 1
  #  )
  #  # FetchContent_MakeAvailable(libdwarf)
  #  FetchContent_GetProperties(libdwarf)
  #  if(NOT libdwarf_POPULATED)
  #    set(PIC_ALWAYS TRUE)
  #    set(BUILD_DWARFDUMP FALSE)
  #    # set(ENABLE_DECOMPRESSION FALSE)
  #    FetchContent_Populate(libdwarf)
  #    add_subdirectory("${libdwarf_SOURCE_DIR}" "${libdwarf_BINARY_DIR}")
  #    target_include_directories(
  #      dwarf
  #      PRIVATE
  #      ${zstd_SOURCE_DIR}/lib
  #    )
  #  endif()
  #endif()

  # libdwarf's dependency on zstd
  set(ZSTD_BUILD_PROGRAMS OFF)
  set(ZSTD_BUILD_CONTRIB OFF)
  set(ZSTD_BUILD_TESTS OFF)
  set(ZSTD_BUILD_STATIC ON)
  set(ZSTD_BUILD_SHARED OFF)
  set(ZSTD_LEGACY_SUPPORT OFF)

  cmake_policy(SET CMP0074 NEW)

  add_subdirectory(libs/zstd/configs/cmake)

  # libdwarf itself
  set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)

  set(PIC_ALWAYS TRUE)
  set(BUILD_DWARFDUMP FALSE)

  add_subdirectory(libs/libdwarf)
  target_include_directories(
          dwarf
          PRIVATE
          libs/zstd/lib
  )

  if(CPPTRACE_CONAN)
    target_link_libraries(${target_name} PRIVATE libdwarf::libdwarf)
  elseif(CPPTRACE_VCPKG)
    target_link_libraries(${target_name} PRIVATE libdwarf::dwarf)
  elseif(CPPTRACE_USE_EXTERNAL_LIBDWARF)
    if(DEFINED LIBDWARF_LIBRARIES)
      target_link_libraries(${target_name} PRIVATE ${LIBDWARF_LIBRARIES})
    else()
      # if LIBDWARF_LIBRARIES wasn't set by find_package, try looking for libdwarf::dwarf-static,
      # libdwarf::dwarf-shared, libdwarf::dwarf, then libdwarf
      # libdwarf v0.8.0 installs with the target libdwarf::dwarf somehow, despite creating libdwarf::dwarf-static or
      # libdwarf::dwarf-shared under fetchcontent
      if(TARGET libdwarf::dwarf-static)
        set(LIBDWARF_LIBRARIES libdwarf::dwarf-static)
      elseif(TARGET libdwarf::dwarf-shared)
        set(LIBDWARF_LIBRARIES libdwarf::dwarf-shared)
      elseif(TARGET libdwarf::dwarf)
        set(LIBDWARF_LIBRARIES libdwarf::dwarf)
      elseif(TARGET libdwarf)
        set(LIBDWARF_LIBRARIES libdwarf)
      else()
        message(FATAL_ERROR "Couldn't find libdwarf target name to link against")
      endif()
      target_link_libraries(${target_name} PRIVATE ${LIBDWARF_LIBRARIES})
    endif()
    # There seems to be no consistency at all about where libdwarf decides to place its headers........ Figure out if
    # it's libdwarf/libdwarf.h and libdwarf/dwarf.h or just libdwarf.h and dwarf.h
    include(CheckIncludeFileCXX)
    # libdwarf's cmake doesn't properly set variables to indicate where its libraries live
    get_target_property(LIBDWARF_INTERFACE_INCLUDE_DIRECTORIES ${LIBDWARF_LIBRARIES} INTERFACE_INCLUDE_DIRECTORIES)
    set(CMAKE_REQUIRED_INCLUDES ${LIBDWARF_INTERFACE_INCLUDE_DIRECTORIES})
    CHECK_INCLUDE_FILE_CXX("libdwarf/libdwarf.h" LIBDWARF_IS_NESTED)
    CHECK_INCLUDE_FILE_CXX("libdwarf.h" LIBDWARF_IS_NOT_NESTED)
    # check_include_file("libdwarf/libdwarf.h" LIBDWARF_IS_NESTED)
    # check_support(LIBDWARF_IS_NESTED nested_libdwarf_include.cpp "" "" "")
    if(${LIBDWARF_IS_NESTED})
      target_compile_definitions(${target_name} PRIVATE CPPTRACE_USE_NESTED_LIBDWARF_HEADER_PATH)
    elseif(NOT LIBDWARF_IS_NOT_NESTED)
      message(FATAL_ERROR "Couldn't find libdwarf.h")
    endif()
  else()
    target_link_libraries(${target_name} PRIVATE libdwarf::dwarf-static)
  endif()
  if(UNIX)
    target_link_libraries(${target_name} PRIVATE ${CMAKE_DL_LIBS})
  endif()
endif()

if(CPPTRACE_GET_SYMBOLS_WITH_DBGHELP)
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_GET_SYMBOLS_WITH_DBGHELP)
  target_link_libraries(${target_name} PRIVATE dbghelp)
endif()

if(CPPTRACE_GET_SYMBOLS_WITH_NOTHING)
  #target_compile_definitions(${target_name} PUBLIC CPPTRACE_GET_SYMBOLS_WITH_NOTHING)

  message(FATAL_ERROR "SpaRcle Common: only libdwarf is currently supported.")
endif()

# Unwinding
if(CPPTRACE_UNWIND_WITH_UNWIND)
  if(NOT HAS_UNWIND)
    message(WARNING "Cpptrace: CPPTRACE_UNWIND_WITH_UNWIND specified but libgcc unwind doesn't seem to be available.")
  endif()
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_UNWIND_WITH_UNWIND)
endif()

if(CPPTRACE_UNWIND_WITH_LIBUNWIND)
  find_package(PkgConfig)
  if(PkgConfig_FOUND)
    pkg_check_modules(LIBUNWIND QUIET libunwind)
    if(libunwind_FOUND)
      target_compile_options(${target_name} PRIVATE ${LIBUNWIND_CFLAGS_OTHER})
      target_include_directories(${target_name} PRIVATE ${LIBUNWIND_INCLUDE_DIRS})
      target_link_libraries(${target_name} PRIVATE ${LIBUNWIND_LDFLAGS})
    endif()
  endif()
  if(NOT libunwind_FOUND)
    # set_property(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS ON)
    # set_property(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB32_PATHS ON)
    find_path(LIBUNWIND_INCLUDE_DIRS NAMES "libunwind.h")
    find_library(LIBUNWIND NAMES unwind libunwind libunwind8 libunwind.so.8 REQUIRED PATHS "/usr/lib/x86_64-linux-gnu/")
    if(LIBUNWIND)
      set(libunwind_FOUND TRUE)
    endif()
  endif()
  if(NOT libunwind_FOUND)
    # message(FATAL_ERROR "Unable to locate libunwind")
    # Try to link with it if it's where it should be
    # This path can be entered if libunwind was installed via the system package manager, sometimes. I probably messed
    # up the find_library above.
    set(LIBUNWIND_LDFLAGS "-lunwind")
  endif()
  target_compile_options(${target_name} PRIVATE ${LIBUNWIND_CFLAGS_OTHER})
  target_include_directories(${target_name} PRIVATE ${LIBUNWIND_INCLUDE_DIRS})
  target_link_libraries(${target_name} PRIVATE ${LIBUNWIND_LDFLAGS})
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_UNWIND_WITH_LIBUNWIND UNW_LOCAL_ONLY)
endif()

if(CPPTRACE_UNWIND_WITH_EXECINFO)
  if(NOT HAS_EXECINFO)
    message(WARNING "Cpptrace: CPPTRACE_UNWIND_WITH_EXECINFO specified but execinfo.h doesn't seem to be available.")
  endif()
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_UNWIND_WITH_EXECINFO)
endif()

if(CPPTRACE_UNWIND_WITH_WINAPI)
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_UNWIND_WITH_WINAPI)
endif()

if(CPPTRACE_UNWIND_WITH_DBGHELP)
  if(NOT HAS_STACKWALK)
    message(WARNING "Cpptrace: CPPTRACE_UNWIND_WITH_DBGHELP specified but dbghelp stackwalk64 doesn't seem to be available.")
  endif()
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_UNWIND_WITH_DBGHELP)
  target_link_libraries(${target_name} PRIVATE dbghelp)
endif()

if(CPPTRACE_UNWIND_WITH_NOTHING)
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_UNWIND_WITH_NOTHING)
endif()

# Demangling
if(CPPTRACE_DEMANGLE_WITH_CXXABI)
  if(NOT HAS_CXXABI)
    message(WARNING "Cpptrace: CPPTRACE_DEMANGLE_WITH_CXXABI specified but cxxabi.h doesn't seem to be available.")
  endif()
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_DEMANGLE_WITH_CXXABI)
endif()

if(CPPTRACE_DEMANGLE_WITH_WINAPI)
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_DEMANGLE_WITH_WINAPI)
  target_link_libraries(${target_name} PRIVATE dbghelp)
endif()

if(CPPTRACE_DEMANGLE_WITH_NOTHING)
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_DEMANGLE_WITH_NOTHING)
endif()

if(NOT "${CPPTRACE_BACKTRACE_PATH}" STREQUAL "")
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_BACKTRACE_PATH=${CPPTRACE_BACKTRACE_PATH})
endif()

if(NOT "${CPPTRACE_HARD_MAX_FRAMES}" STREQUAL "")
  target_compile_definitions(${target_name} PUBLIC CPPTRACE_HARD_MAX_FRAMES=${CPPTRACE_HARD_MAX_FRAMES})
endif()

# =============================================== Install ===============================================

if(NOT CMAKE_SKIP_INSTALL_RULES)
  include(cmake/InstallRules.cmake)
endif()

# =============================================== Demo/test ===============================================

macro(add_test_dependencies exec_name)
  target_compile_features(${exec_name} PRIVATE cxx_std_11)
  target_link_libraries(${exec_name} PRIVATE ${target_name})
  # Clang has been fast to adopt dwarf 5, other tools (e.g. addr2line from binutils) have not
  check_cxx_compiler_flag("-gdwarf-4" HAS_DWARF4)
  if(HAS_DWARF4)
    target_compile_options(${exec_name} PRIVATE "$<$<CONFIG:Debug>:-gdwarf-4>")
  endif()
  # TODO: add debug info for mingw clang?
  if(CPPTRACE_BUILD_TEST_RDYNAMIC)
    set_property(TARGET ${exec_name} PROPERTY ENABLE_EXPORTS ON)
  endif()
endmacro()

if(CPPTRACE_BUILD_TESTING)
  add_executable(test test/test.cpp)
  add_executable(demo test/demo.cpp)
  add_executable(c_demo test/ctrace_demo.c)

  add_test_dependencies(test)
  add_test_dependencies(demo)
  add_test_dependencies(c_demo)

  if(UNIX)
    add_executable(signal_demo test/signal_demo.cpp)
    target_compile_features(signal_demo PRIVATE cxx_std_11)
    target_link_libraries(signal_demo PRIVATE ${target_name})

    add_executable(signal_tracer test/signal_tracer.cpp)
    target_compile_features(signal_tracer PRIVATE cxx_std_11)
    target_link_libraries(signal_tracer PRIVATE ${target_name})
  endif()
endif()
