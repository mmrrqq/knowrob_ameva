cmake_minimum_required(VERSION 3.5)
project(knowrob_ameva)

# Default to C99
if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

# Default to C++17
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

find_package(ament_cmake REQUIRED)
find_package(ament_index_cpp REQUIRED)
find_package(rcl REQUIRED)
find_package(rclcpp REQUIRED)

find_package(PkgConfig)
find_package(Threads)
find_package(Protobuf REQUIRED)

find_package(SWIPL REQUIRED)
if(NOT SWIPL_FOUND)
  message( FATAL_ERROR "SWIPL not found." )
endif()

if(NOT PROTOBUF_FOUND)
  message( FATAL_ERROR "PROTOBUF not found.")
endif()

pkg_check_modules(LIBWEBSOCKETS REQUIRED libwebsockets)
# link_directories(${LIBWEBSOCKETS_LIBRARY_DIRS})
message( STATUS "LIBWEBSOCKETS found.")

pkg_check_modules(CURLPP REQUIRED curlpp)
# link_directories(${CURLPP_LIBRARY_DIRS})
message(STATUS "LIBWEBSOCKETS found.")

set(proto_dir ${PROJECT_SOURCE_DIR}/src/proto)
file(GLOB proto_files "${proto_dir}/*.proto")
message(STATUS "Proto Source Dir: ${proto_dir}")
message(STATUS "Proto Source Files: ${proto_files}")

# Set up destination directories
set(proto_gen_cpp_dir build/${PROJECT_NAME}/include)
file(MAKE_DIRECTORY ${proto_gen_cpp_dir})
message(STATUS "Proto Gen Dirs:${proto_gen_cpp_dir}")

# Create lists of files to be generated.
set(proto_gen_cpp_files "")
foreach(proto_file ${proto_files})
    get_filename_component(proto_name ${proto_file} NAME_WE)
    list(APPEND proto_gen_cpp_files ${proto_gen_cpp_dir}/${proto_name}.pb.h ${proto_gen_cpp_dir}/${proto_name}.pb.cc)
endforeach(proto_file ${proto_files})

# Create lists of generated source files
set(generated_proto_source "")
foreach(source_file ${proto_gen_cpp_files})
    get_filename_component(proto_name ${source_file} NAME_WE)
    list(APPEND generated_proto_source include/${proto_name}.pb.h include/${proto_name}.pb.cc)
endforeach(source_file ${proto_gen_cpp_files})

message(STATUS "Proto Gen Files: ${generated_proto_source}")

message(STATUS "Proto Gen dir: ${proto_gen_cpp_dir}")
# Run protoc and generate language-specific headers.
add_custom_command(
    OUTPUT ${proto_gen_cpp_files}
    COMMAND ${PROTOBUF_PROTOC_EXECUTABLE} --proto_path=${proto_dir} --cpp_out=${proto_gen_cpp_dir}  ${proto_files}
    DEPENDS ${PROTOBUF_PROTOC_EXECUTABLE} ${proto_files}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

add_custom_target(knowrob_ameva_generate_headers ALL
    COMMAND echo "Creating proto generated cpp files"
    DEPENDS ${proto_gen_cpp_files}
)

set_source_files_properties(${generated_proto_source} PROPERTIES GENERATED TRUE)

# Create proto library for linking.
add_library(knowrob_ameva_proto SHARED ${generated_proto_source})
add_dependencies(knowrob_ameva_proto knowrob_ameva_generate_headers)
target_include_directories(knowrob_ameva_proto PUBLIC ${PROTOBUF_INCLUDE_DIRS})
target_link_libraries(knowrob_ameva_proto ${PROTOBUF_LIBRARIES})
ament_target_dependencies(knowrob_ameva_proto 
  rcl
  rclcpp
)

add_library(knowrob_ameva SHARED 
    src/ue_ws_conn.cpp 
    src/ue_viz.cpp 
    src/ue_control.cpp 
    src/ag_conn.cpp 
    src/kr_ws_server.cpp
)
target_include_directories(${PROJECT_NAME} PUBLIC ${SWIPL_INCLUDE_DIRS} ${LIBWEBSOCKETS_INCLUDE_DIRS} ${proto_gen_cpp_dir} ${CURLPP_INCLUDE_DIRS} include/${PROJECT_NAME})
target_link_libraries(${PROJECT_NAME} knowrob_ameva_proto ${rcl_LIBRARIES} ${rclcpp_LIBRARIES} ${LIBWEBSOCKETS_LIBRARIES} ${SWIPL_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${CURLPP_LIBRARIES})
add_dependencies(${PROJECT_NAME} knowrob_ameva_proto)
ament_target_dependencies(${PROJECT_NAME}
  rcl
  rclcpp
)

install(TARGETS knowrob_ameva
  DESTINATION lib/${PROJECT_NAME})
install(TARGETS knowrob_ameva_proto
  DESTINATION lib/${PROJECT_NAME})

install(
  DIRECTORY include
  DESTINATION include/${PROJECT_NAME}
)
install(
  PROGRAMS ${SWIPL_LIBRARIES}
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()