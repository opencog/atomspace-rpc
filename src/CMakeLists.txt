#Protobuff
set(protobuf_MODULE_COMPATIBLE TRUE)
find_package(protobuf REQUIRED)
message(STATUS "Using protobuf ${protobuf_VERSION}")

# gRPC
find_package(gRPC REQUIRED)
message(STATUS "Using gRPC ${gRPC_VERSION}")

#gRPC C++ plugin
get_target_property(gRPC_CPP_PLUGIN_EXECUTABLE gRPC::grpc_cpp_plugin IMPORTED_LOCATION_RELEASE)

#Proto file
get_filename_component(atom_proto "../specs/atom_server.proto" ABSOLUTE)
get_filename_component(atom_proto_path "${atom_proto}" PATH)
#Generated sources

protobuf_generate_cpp(atom_proto_srcs atom_proto_hdrs "${atom_proto}")
set(atom_grpc_srcs "${CMAKE_CURRENT_BINARY_DIR}/atom_server.grpc.pb.cc")
set(atom_grpc_hdrs "${CMAKE_CURRENT_BINARY_DIR}/atom_server.grpc.pb.h")

add_custom_command(
        OUTPUT "${atom_grpc_srcs}" "${atom_grpc_hdrs}"
        COMMAND protobuf::protoc
        ARGS --grpc_out "${CMAKE_CURRENT_BINARY_DIR}" -I "${atom_proto_path}"
        --plugin=protoc-gen-grpc="${gRPC_CPP_PLUGIN_EXECUTABLE}"
        "${atom_proto}"
        DEPENDS "${atom_proto}"
)

#Generated include directory
include_directories("${CMAKE_CURRENT_BINARY_DIR}")

set(SOURCES manager/AtomSpaceManager.cpp ${atom_grpc_srcs} ${atom_proto_srcs})

add_executable(atom_server AtomspaceService.cpp)
target_sources(atom_server PRIVATE ${SOURCES})

find_library(BIOSCIENCE_LIB
        NAME libbioscience-types.dylib libbioscience-types.so
        PATHS ${CMAKE_INSTALL_PREFIX}/lib/opencog
        NO_DEFAULT_PATH)

# Set default include paths.
include_directories(${CMAKE_CURRENT_SOURCE_DIR}
        ${COGUTIL_INCLUDE_DIR}
        ${ATOMSPACE_INCLUDE_DIR}
        ${GUILE_INCLUDE_DIR})


target_link_libraries(atom_server
        ${COGUTIL_LIBRARIES}
        ${ATOMSPACE_LIBRARIES}
        ${Boost_LIBRARIES}
        protobuf::libprotobuf
        gRPC::grpc++_unsecure)


IF(BIOSCIENCE_LIB)
    message(STATUS "AGI-Bio found.")
    target_link_libraries(atom_server ${BIOSCIENCE_LIB})
ENDIF()

install(TARGETS atom_server DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)

add_library(atom_service_client SHARED AtomServiceClient.cpp)

target_sources(atom_service_client PRIVATE manager/AtomSpaceManager.cpp ${atom_grpc_srcs} ${atom_proto_srcs})

target_link_libraries(atom_service_client
        ${COGUTIL_LIBRARIES}
        ${ATOMSPACE_LIBRARIES}
        ${GUILE_LIBRARIES}
        protobuf::libprotobuf
        gRPC::grpc++_unsecure)

IF(BIOSCIENCE_LIB)
    target_link_libraries(atom_service_client ${BIOSCIENCE_LIB})
ENDIF()

install(TARGETS atom_service_client
        DESTINATION "lib${LIB_DIR_SUFFIX}/opencog")

add_subdirectory(guile)