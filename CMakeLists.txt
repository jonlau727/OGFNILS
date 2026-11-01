cmake_minimum_required(VERSION 3.24)
project(SimForgeS4 VERSION 0.2.0 LANGUAGES CXX C)

# ---------------------------------------------------------------------------
# SimForge S4 — synthetic season lab.
# Scope boundary (project rule): everything in this solution operates on a
# self-contained simulated world rendered by this process. There is no code
# that opens, reads, injects into, or influences any other process.
# ---------------------------------------------------------------------------

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Static CRT for release distribution so end users don't need VC redist just for us.
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded")

# /GL + /LTCG (link-time optimization)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)

if (MSVC)
  add_compile_options(/W4 /permissive- /utf-8 /EHsc /MP)
  add_compile_definitions(_CRT_SECURE_NO_WARNINGS NOMINMAX WIN32_LEAN_AND_MEAN UNICODE _UNICODE)
  # Release: strip symbols/PDB from anything shipped; optimize hard.
  set(CMAKE_EXE_LINKER_FLAGS_RELEASE "/DEBUG:NONE /OPT:REF /OPT:ICF /INCREMENTAL:NO")
endif()

include(cmake/deps.cmake)

# Optional OpenCV research module (off by default; find_package on the
# operator machine — nothing is fetched).
option(SFS4_WITH_OPENCV "Enable the OpenCV synthetic-image research module" OFF)

enable_testing()

add_subdirectory(src)
