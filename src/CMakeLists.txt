# Copyright 2018 The clvk authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.


if(NOT MSVC)
    add_compile_options(-Wall -W -Wextra)
    add_compile_options(-Wno-ignored-attributes)
else()
	set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

if (UNIX AND NOT APPLE)
    set(CMAKE_SHARED_LINKER_FLAGS
        "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/exports.map")
endif()

if (UNIX AND NOT APPLE)
    # Bind global symbol references to definitions within the shared library,
    # to allow the ICD dispatch table to be statically defined.
    set(CMAKE_SHARED_LINKER_FLAGS
        "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-Bsymbolic")
endif()

option(CLVK_CLSPV_ONLINE_COMPILER "Use the Clspv C++ API for compilation of kernels")
if (CLVK_CLSPV_ONLINE_COMPILER)
  add_compile_options(-DCLSPV_ONLINE_COMPILER)
endif()

add_library(OpenCL-objects OBJECT
  api.cpp
  device.cpp
  init.cpp
  kernel.cpp
  log.cpp
  memory.cpp
  program.cpp
  queue.cpp
  sha1.cpp
  utils.cpp
  exports.map
)

# Disable deprecation warnings on the file that has the entrypoints.
if (NOT MSVC)
set_source_files_properties(api.cpp PROPERTIES COMPILE_FLAGS
    -Wno-deprecated-declarations)
endif()

set_property(TARGET OpenCL-objects PROPERTY POSITION_INDEPENDENT_CODE 1)
target_include_directories(OpenCL-objects SYSTEM BEFORE PRIVATE
    ${SPIRV_HEADERS_SOURCE_DIR}/include
    ${SPIRV_TOOLS_SOURCE_DIR}/include
    ${Vulkan_INCLUDE_DIRS}
)

find_package(Threads)

set(OpenCL-dependencies
  ${CMAKE_THREAD_LIBS_INIT}
  SPIRV-Tools-opt
  SPIRV-Tools-link
  ${Vulkan_LIBRARIES}
)

# The reflection parser requires the headers.
target_include_directories(OpenCL-objects PRIVATE
  "${CLSPV_SOURCE_DIR}/include")
if (CLVK_CLSPV_ONLINE_COMPILER)
  set(OpenCL-dependencies ${OpenCL-dependencies} clspv_core)
else()
  add_dependencies(OpenCL-objects clspv)
  set_target_properties(clspv PROPERTIES RUNTIME_OUTPUT_DIRECTORY
    ${CMAKE_BINARY_DIR})
  add_dependencies(OpenCL-objects llvm-spirv)
  set_target_properties(llvm-spirv PROPERTIES RUNTIME_OUTPUT_DIRECTORY
    ${CMAKE_BINARY_DIR})
  target_compile_definitions(OpenCL-objects PRIVATE
    DEFAULT_LLVMSPIRV_BINARY_PATH="$<TARGET_FILE:llvm-spirv>")
  target_compile_definitions(OpenCL-objects PRIVATE
    DEFAULT_CLSPV_BINARY_PATH="$<TARGET_FILE:clspv>")
endif()

if(WIN32)
	target_compile_definitions(OpenCL-objects PRIVATE
		NOMINMAX)
endif()

function(CLLibrary target type)
  add_library(${target} ${type} $<TARGET_OBJECTS:OpenCL-objects>)
  target_link_libraries(${target} ${OpenCL-dependencies})
  set_target_properties(${target} PROPERTIES LIBRARY_OUTPUT_DIRECTORY
      ${CMAKE_BINARY_DIR})
endfunction(CLLibrary)

CLLibrary(OpenCL SHARED)
CLLibrary(OpenCL-static STATIC)

set_target_properties(OpenCL PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION 1)

