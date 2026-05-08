project(lib)

file(GLOB SRCS "*.c")

file(GLOB HEADERS "peer.h" "peer_connection.h" "peer_signaling.h")

add_library(peer
  ${SRCS}
  ${HTTP_SOURCES}
  ${MQTT_SOURCES}
  ${MQTT_SERIALIZER_SOURCES}
)

include_directories(peer PUBLIC
  ${HTTP_INCLUDE_PUBLIC_DIRS}
  ${MQTT_INCLUDE_PUBLIC_DIRS}
)

add_dependencies(peer cjson mbedtls usrsctp srtp2)

target_link_libraries(peer ${DEP_LIBS} srtp2)

set_target_properties(peer PROPERTIES PUBLIC_HEADER "${HEADERS}")

install(TARGETS peer
  LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib/
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_PREFIX}/include/
)

install(EXPORT peer DESTINATION lib/cmake/peer FILE peerConfig.cmake)
install(FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/../include/peer.h
  ${CMAKE_CURRENT_SOURCE_DIR}/../include/peer_connection.h
  ${CMAKE_CURRENT_SOURCE_DIR}/../include/peer_signaling.h DESTINATION ${CMAKE_INSTALL_PREFIX}/include/)
install(TARGETS ${DEP_LIBS} EXPORT peer
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})