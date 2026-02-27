cmake_minimum_required(VERSION 3.20)

project(PPMalloc LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

option(PPMALLOC_FETCH_DEPS "Fetch missing EA dependencies from GitHub via FetchContent" ON)
option(PPMALLOC_BUILD_TESTS "Build PPMalloc tests (requires additional EA packages)" OFF)

set(EA_GIT_BASE_URL "https://github.com" CACHE STRING "Base URL for EA dependency repositories")
set(EA_GITHUB_ORG "DOCaCola" CACHE STRING "GitHub org/user for EA package repositories")
set(EA_GIT_TAG "main" CACHE STRING "Git tag/branch/commit for EA package repositories")

function(ea_apply_msvc_options tgt)
  if(MSVC)
    target_compile_definitions(${tgt} PRIVATE _CRT_SECURE_NO_WARNINGS)
    target_compile_options(${tgt} PRIVATE
      /MP
      /W4
      $<$<CONFIG:RELEASE>:-Zo -d2SSAOptimizer->
      -bigobj /FS /Zf
      -w44946 -w44555 -w44312 -w44296 -w44287 -w44265 -w44254 -w44242
      /wd4574 /wd4365 /wd4100 /wd4324 /wd4350 /wd4464 /wd4482 /wd4577 /wd4711 /wd4746 /wd4986 /wd4987 /wd4819
    )
  endif()
endfunction()

function(ea_apply_common_config_defines tgt)
  target_compile_definitions(${tgt} PRIVATE
    $<$<CONFIG:RELEASE>:NDEBUG>
    $<$<CONFIG:DEBUG>:EA_DEBUG>
    $<$<CONFIG:RelWithDebInfo>:EA_DEBUG>
  )
endfunction()

function(ea_repo_url out_var repo_name)
  set(${out_var} "${EA_GIT_BASE_URL}/${EA_GITHUB_ORG}/${repo_name}.git" PARENT_SCOPE)
endfunction()

function(ea_add_dep name)
  if(TARGET ${name})
    return()
  endif()

  if(NOT PPMALLOC_FETCH_DEPS)
    message(FATAL_ERROR "Missing dependency target '${name}'. Provide it in the parent project or set PPMALLOC_FETCH_DEPS=ON.")
  endif()

  string(TOUPPER "${name}" name_upper)
  set(repo_var "EA_${name_upper}_GIT_REPOSITORY")
  set(tag_var  "EA_${name_upper}_GIT_TAG")

  if(NOT DEFINED ${repo_var})
    ea_repo_url(default_repo "${name}")
    set(${repo_var} "${default_repo}" CACHE STRING "Git repository for ${name}")
  endif()
  if(NOT DEFINED ${tag_var})
    set(${tag_var} "${EA_GIT_TAG}" CACHE STRING "Git tag/branch/commit for ${name}")
  endif()

  FetchContent_Declare(
    ${name}
    GIT_REPOSITORY "${${repo_var}}"
    GIT_TAG "${${tag_var}}"
    GIT_SHALLOW TRUE
    GIT_SUBMODULES ""
  )
  FetchContent_MakeAvailable(${name})
endfunction()

# PPMalloc depends on EACallstack and EAThread; those projects will pull their own transitive deps.
ea_add_dep(EABase)
ea_add_dep(EAThread)
ea_add_dep(EACallstack)

file(GLOB ppmalloc_sources CONFIGURE_DEPENDS
  "${CMAKE_CURRENT_LIST_DIR}/source/cmn/*.cpp"
  "${CMAKE_CURRENT_LIST_DIR}/extras/EASmallBlockAllocator.cpp"
  "${CMAKE_CURRENT_LIST_DIR}/extras/EASmallBlockAllocatorPool.cpp"
  "${CMAKE_CURRENT_LIST_DIR}/extras/EAAllocInfo.cpp"
)

# This file explicitly errors out on Win64, so only compile it for 32-bit Windows.
if(WIN32 AND (CMAKE_SIZEOF_VOID_P EQUAL 4))
  list(APPEND ppmalloc_sources "${CMAKE_CURRENT_LIST_DIR}/source/pc/EAGeneralAllocatorDebugWin32.cpp")
endif()

file(GLOB_RECURSE ppmalloc_headers CONFIGURE_DEPENDS
  "${CMAKE_CURRENT_LIST_DIR}/include/*.h"
)

add_library(PPMalloc ${ppmalloc_sources})
target_sources(PPMalloc PRIVATE ${ppmalloc_headers})

target_include_directories(PPMalloc PUBLIC "${CMAKE_CURRENT_LIST_DIR}/include")
target_link_libraries(PPMalloc PUBLIC EABase EAThread EACallstack)

option(PPMallocExtraTests "Enable PPMalloc Test Extra configuration" OFF)
if(PPMallocExtraTests)
  target_compile_definitions(PPMalloc PRIVATE PPM_MIN_ALIGN=16 PPM_DEBUG=2)
endif()

target_compile_definitions(PPMalloc PRIVATE
  EASTL_CORE_ALLOCATOR_ENABLED=1
  EASTL_EASTDC_VSNPRINTF=1
  PPM_HIGH_SUPPORTED=1
)

ea_apply_common_config_defines(PPMalloc)
ea_apply_msvc_options(PPMalloc)

if(PPMALLOC_BUILD_TESTS)
  message(FATAL_ERROR "PPMALLOC_BUILD_TESTS is ON, but test wiring (EATest/EAMain) isn't implemented yet."
                      " Tell me where you want to source EATest/EAMain from (GitHub org/tag) and I'll add it.")
endif()
