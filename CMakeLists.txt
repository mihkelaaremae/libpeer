cmake_minimum_required(VERSION 3.16)
cmake_policy(VERSION 3.16)

include(${CMAKE_CURRENT_LIST_DIR}/third_party/coreHTTP/httpFilePaths.cmake)
include(${CMAKE_CURRENT_LIST_DIR}/third_party/coreMQTT/mqttFilePaths.cmake)

if (DEFINED ENV{IDF_PATH})
  file(GLOB ESP32_CODES "./src/*.c")
  idf_component_register(
   SRCS ${ESP32_CODES} ${HTTP_SOURCES} ${MQTT_SOURCES} ${MQTT_SERIALIZER_SOURCES}
   INCLUDE_DIRS "./include" ${HTTP_INCLUDE_PUBLIC_DIRS} ${MQTT_INCLUDE_PUBLIC_DIRS}
   PRIV_INCLUDE_DIRS "./src"
   REQUIRES mbedtls srtp json esp_netif
  )
  add_definitions("-DCONFIG_USE_LWIP=1" "-DCONFIG_USE_USRSCTP=0" "-DCONFIG_AUDIO_BUFFER_SIZE=8096" "-DCONFIG_DATA_BUFFER_SIZE=102400" "-D__BYTE_ORDER=__LITTLE_ENDIAN" "-DHTTP_DO_NOT_USE_CUSTOM_CONFIG" "-DMQTT_DO_NOT_USE_CUSTOM_CONFIG")
  return()
endif()

project(peer VERSION 0.0.1)

option(ENABLE_TESTS "Enable tests" OFF)
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(ADDRESS_SANITIZER "Build with AddressSanitizer." OFF)
option(MEMORY_SANITIZER "Build with MemorySanitizer." OFF)
option(THREAD_SANITIZER "Build with ThreadSanitizer." OFF)
option(UNDEFINED_BEHAVIOR_SANITIZER "Build with UndefinedBehaviorSanitizer." OFF)

include(ExternalProject)

include_directories(${CMAKE_CURRENT_BINARY_DIR}/dist/include ${CMAKE_CURRENT_BINARY_DIR}/dist/include/cjson)
link_directories(${CMAKE_CURRENT_BINARY_DIR}/dist/lib)

set(DEP_LIBS "srtp2" "usrsctp" "mbedtls" "mbedcrypto" "mbedx509" "cjson")
# Extended debug information (symbols, source code, and macro definitions)
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g3")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g3")

function(enableSanitizer SANITIZER)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -g -fsanitize=${SANITIZER} -fno-omit-frame-pointer -fno-optimize-sibling-calls" PARENT_SCOPE)
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=${SANITIZER}" PARENT_SCOPE)
endfunction()

if(ADDRESS_SANITIZER)
  enableSanitizer("address")
endif()

if(MEMORY_SANITIZER)
  enableSanitizer("memory")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-memory-track-origins")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize-memory-track-origins")
endif()

if(THREAD_SANITIZER)
  enableSanitizer("thread")
endif()

if(UNDEFINED_BEHAVIOR_SANITIZER)
  enableSanitizer("undefined")
endif()

add_definitions("-Wunused-variable -Werror=sequence-point -Werror=pointer-sign -Werror=return-type -Werror=sizeof-pointer-memaccess -Wincompatible-pointer-types -DHTTP_DO_NOT_USE_CUSTOM_CONFIG -DMQTT_DO_NOT_USE_CUSTOM_CONFIG")

file(READ ${CMAKE_CURRENT_SOURCE_DIR}/third_party/mbedtls/include/mbedtls/mbedtls_config.h INPUT_CONTENT)
string(REPLACE "//#define MBEDTLS_SSL_DTLS_SRTP" "#define MBEDTLS_SSL_DTLS_SRTP" MODIFIED_CONTENT ${INPUT_CONTENT})
file(WRITE ${CMAKE_CURRENT_SOURCE_DIR}/third_party/mbedtls/include/mbedtls/mbedtls_config.h ${MODIFIED_CONTENT})

option(LIBSRTP_TEST_APPS OFF)
option(ENABLE_CJSON_UNINSTALL OFF)
option(ENABLE_LOCALES OFF)
option(ENABLE_CJSON_TEST OFF)
option(BUILD_SHARED_LIBS OFF)
option(ENABLE_TESTING OFF)
option(ENABLE_PROGRAMS OFF)
option(TEST_APPS OFF)
option(sctp_build_programs OFF)

add_subdirectory(third_party/cJSON)
add_subdirectory(third_party/mbedtls)
add_subdirectory(third_party/libsrtp)
add_subdirectory(third_party/usrsctp)

target_compile_options(cjson PRIVATE -w)
target_compile_options(srtp2 PRIVATE -fPIC -w)
target_compile_options(usrsctp PRIVATE -fPIC -w)
target_compile_options(mbedtls PRIVATE -fPIC -w)
target_compile_options(mbedcrypto PRIVATE -fPIC -w)
target_compile_options(mbedx509 PRIVATE -fPIC -w)

set_target_properties(srtp2 PROPERTIES INTERFACE_INCLUDE_DIRECTORIES "") 
set_target_properties(usrsctp PROPERTIES INTERFACE_INCLUDE_DIRECTORIES "") 
target_include_directories(srtp2 PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/third_party/libsrtp/include/>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/third_party/libsrtp/crypto/include>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_PREFIX}/include>
  )
target_include_directories(usrsctp PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/third_party/usrsctp/usrsctplib>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_PREFIX}/include>
  )
target_include_directories(cjson PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/third_party/cJSON/>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_PREFIX}/include>
  )

add_subdirectory(src)
add_subdirectory(examples)

if(ENABLE_TESTS)
  add_subdirectory(tests)
endif()
