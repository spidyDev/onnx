# Minimum CMake required
cmake_minimum_required(VERSION 3.1)

# Project
project(onnx C CXX)

# Set C++11 as standard for the whole project
set(CMAKE_CXX_STANDARD 11)

set(ONNX_ROOT ${PROJECT_SOURCE_DIR})
if (WIN32)
    # parallel build
    add_compile_options(/MP)
    add_compile_options(/WX)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /WX")
    set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS} /WX")
endif()
add_subdirectory(third_party/pybind11)
FIND_PACKAGE(Protobuf REQUIRED)

# function(RELATIVE_PROTOBUF_GENERATE_CPP SRCS HDRS ROOT_DIR)
# from https://github.com/tensorflow/tensorflow/blob/d2c3b873c6f8ff999a2e4ee707a84ff00d9c15a5/tensorflow/contrib/cmake/tf_core_framework.cmake
# to solve the problem that customized dir can't be specified when calling PROTOBUF_GENERATE_CPP.
function(RELATIVE_PROTOBUF_GENERATE_CPP SRCS HDRS ROOT_DIR)
  if(NOT ARGN)
    message(SEND_ERROR "Error: RELATIVE_PROTOBUF_GENERATE_CPP() called without any proto files")
    return()
  endif()

  if(MSVC)
    set(DLLEXPORT_STR "dllexport_decl=ONNX_API:")
  else()
    set(DLLEXPORT_STR "")
  endif()

  set(${SRCS})
  set(${HDRS})
  foreach(FIL ${ARGN})
    message(${FIL})
    set(ABS_FIL ${ROOT_DIR}/${FIL})
    get_filename_component(FIL_WE ${FIL} NAME_WE)
    get_filename_component(FIL_DIR ${ABS_FIL} DIRECTORY)
    file(RELATIVE_PATH REL_DIR ${ROOT_DIR} ${FIL_DIR})

    list(APPEND ${SRCS} "${CMAKE_CURRENT_BINARY_DIR}/${REL_DIR}/${FIL_WE}.pb.cc")
    list(APPEND ${HDRS} "${CMAKE_CURRENT_BINARY_DIR}/${REL_DIR}/${FIL_WE}.pb.h")

    add_custom_command(
      OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${REL_DIR}/${FIL_WE}.pb.cc"
             "${CMAKE_CURRENT_BINARY_DIR}/${REL_DIR}/${FIL_WE}.pb.h"
      COMMAND ${PROTOBUF_PROTOC_EXECUTABLE}
      ARGS --cpp_out  ${DLLEXPORT_STR}${CMAKE_CURRENT_BINARY_DIR} -I ${ROOT_DIR} ${ABS_FIL} -I ${PROTOBUF_INCLUDE_DIRS} -I ${ROOT_DIR}/${REL_DIR}
      COMMENT "Running C++ protocol buffer compiler on ${FIL}"
      VERBATIM )
  endforeach()

  set_source_files_properties(${${SRCS}} ${${HDRS}} PROPERTIES GENERATED TRUE)
  set(${SRCS} ${${SRCS}} PARENT_SCOPE)
  set(${HDRS} ${${HDRS}} PARENT_SCOPE)
endfunction()

RELATIVE_PROTOBUF_GENERATE_CPP(PROTO_SRCS PROTO_HDRS ${ONNX_ROOT}
    onnx/onnx.proto
)

# Temp solution to remove files generated by calling 'pip install ..."
# Should be removed after changing setup.py to use cmake to build c++ part.
file(REMOVE
    "${ONNX_ROOT}/onnx/onnx.pb.h"
    "${ONNX_ROOT}/onnx/onnx.pb.cc"
    "${ONNX_ROOT}/onnx/onnx-operators.pb.h"
    "${ONNX_ROOT}/onnx/onnx-operators.pb.cc"
)

file(GLOB_RECURSE onnx_src
    "${ONNX_ROOT}/onnx/*.h"
    "${ONNX_ROOT}/onnx/*.cc"
)

if(MSVC)
    set(DLLEXPORT_OPTION "-DONNX_API=__declspec(dllexport)")
else()
    set(DLLEXPORT_OPTION "-DONNX_API=")
endif()

add_library(onnx_proto ${PROTO_SRCS} ${PROTO_HDRS})
target_include_directories(onnx_proto PUBLIC "${CMAKE_CURRENT_BINARY_DIR}" "${PROTOBUF_INCLUDE_DIRS}")
target_link_libraries(onnx_proto PUBLIC ${PROTOBUF_LIBRARIES})


if(MSVC)
    target_compile_options(onnx_proto PRIVATE /WX- ${DLLEXPORT_OPTION})
endif()

add_library(onnx ${onnx_src})
target_include_directories(onnx PUBLIC ${ONNX_ROOT} "${CMAKE_CURRENT_BINARY_DIR}" "${PYBIND11_INCLUDE_DIR}" "${PYTHON_INCLUDE_DIRS}")
target_link_libraries(onnx PUBLIC onnx_proto)

if (WIN32)
    target_compile_options(onnx PRIVATE
        /wd4800 # disable warning type' : forcing value to bool 'true' or 'false' (performance warning)
        /wd4503 # identifier' : decorated name length exceeded, name was truncated
    )
    set(onnx_static_library_flags
        -IGNORE:4221 # LNK4221: This object file does not define any previously undefined public symbols, so it will not be used by any link operation that consumes this library
    )
    set_target_properties(onnx PROPERTIES
        STATIC_LIBRARY_FLAGS "${onnx_static_library_flags}")
endif()
if(APPLE)
    set_target_properties(onnx PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
endif()
