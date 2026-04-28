cmake_minimum_required(VERSION 3.22)

project(BoringSSLAndroid LANGUAGES C CXX ASM)

option(ENABLE_LTO "Build BoringSSL with link-time optimization" OFF)

set(BORINGSSL_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")

include("${BORINGSSL_SOURCE_DIR}/gen/sources.cmake")

set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

function(boringssl_resolve_sources out_var)
    set(resolved_sources)
    foreach(source IN LISTS ARGN)
        if(IS_ABSOLUTE "${source}")
            list(APPEND resolved_sources "${source}")
        else()
            list(APPEND resolved_sources "${BORINGSSL_SOURCE_DIR}/${source}")
        endif()
    endforeach()
    set(${out_var} ${resolved_sources} PARENT_SCOPE)
endfunction()

function(boringssl_apply_common_options target)
    target_include_directories(${target}
        PRIVATE
            "${BORINGSSL_SOURCE_DIR}"
            "${BORINGSSL_SOURCE_DIR}/include")
    target_compile_definitions(${target}
        PRIVATE
            BORINGSSL_IMPLEMENTATION
            OPENSSL_SMALL)
    target_compile_options(${target}
        PRIVATE
            $<$<COMPILE_LANGUAGE:C,CXX>:-fno-strict-aliasing>
            $<$<COMPILE_LANGUAGE:C,CXX>:-fno-common>
            $<$<COMPILE_LANGUAGE:C,CXX>:-fvisibility=hidden>
            $<$<COMPILE_LANGUAGE:C,CXX>:-Wno-builtin-macro-redefined>
            $<$<COMPILE_LANGUAGE:C,CXX>:-D__FILE__=__FILE_NAME__>
            $<$<COMPILE_LANGUAGE:ASM>:-Wa,--noexecstack>)
    set_target_properties(${target} PROPERTIES POSITION_INDEPENDENT_CODE ON)

    if(ENABLE_LTO)
        target_compile_options(${target}
            PRIVATE
                $<$<COMPILE_LANGUAGE:C,CXX>:-flto>)
    endif()
endfunction()

function(boringssl_apply_link_options target)
    target_link_options(${target}
        PRIVATE
            -Wl,--icf=all)
    if(ENABLE_LTO)
        target_link_options(${target}
            PRIVATE
                -flto)
    endif()
endfunction()

function(boringssl_package_executable target)
    set_target_properties(${target}
        PROPERTIES
            RUNTIME_OUTPUT_NAME "lib${target}"
            SUFFIX ".so")
endfunction()

boringssl_resolve_sources(CRYPTO_ANDROID_SOURCES
    ${CRYPTO_SOURCES}
    ${BCM_SOURCES}
    ${CRYPTO_SOURCES_ASM}
    ${BCM_SOURCES_ASM}
    ${TEST_SUPPORT_SOURCES_ASM})
boringssl_resolve_sources(SSL_ANDROID_SOURCES ${SSL_SOURCES})
boringssl_resolve_sources(BSSL_ANDROID_SOURCES ${BSSL_SOURCES})
boringssl_resolve_sources(TEST_SUPPORT_ANDROID_SOURCES ${TEST_SUPPORT_SOURCES})
boringssl_resolve_sources(CRYPTO_TEST_ANDROID_SOURCES ${CRYPTO_TEST_SOURCES})
boringssl_resolve_sources(SSL_TEST_ANDROID_SOURCES ${SSL_TEST_SOURCES})

add_library(crypto_static STATIC ${CRYPTO_ANDROID_SOURCES})
boringssl_apply_common_options(crypto_static)
target_include_directories(crypto_static
    PRIVATE
        "${BORINGSSL_SOURCE_DIR}/crypto"
    PUBLIC
        "${BORINGSSL_SOURCE_DIR}/include")

add_library(ssl_static STATIC ${SSL_ANDROID_SOURCES})
boringssl_apply_common_options(ssl_static)
target_include_directories(ssl_static
    PUBLIC
        "${BORINGSSL_SOURCE_DIR}/include")
target_link_libraries(ssl_static PUBLIC crypto_static)

add_library(gtest_main STATIC
    "${BORINGSSL_SOURCE_DIR}/third_party/googletest/googletest/src/gtest-all.cc"
    "${BORINGSSL_SOURCE_DIR}/third_party/googletest/googlemock/src/gmock-all.cc")
target_include_directories(gtest_main
    PUBLIC
        "${BORINGSSL_SOURCE_DIR}/third_party/googletest/googletest/include"
        "${BORINGSSL_SOURCE_DIR}/third_party/googletest/googlemock/include"
    PRIVATE
        "${BORINGSSL_SOURCE_DIR}/third_party/googletest/googletest"
        "${BORINGSSL_SOURCE_DIR}/third_party/googletest/googlemock")

add_library(test_support STATIC ${TEST_SUPPORT_ANDROID_SOURCES})
boringssl_apply_common_options(test_support)
target_link_libraries(test_support PRIVATE gtest_main crypto_static)

add_executable(crypto_test ${CRYPTO_TEST_ANDROID_SOURCES})
boringssl_apply_common_options(crypto_test)
boringssl_apply_link_options(crypto_test)
boringssl_package_executable(crypto_test)
target_link_libraries(crypto_test PRIVATE test_support gtest_main crypto_static)

add_executable(ssl_test ${SSL_TEST_ANDROID_SOURCES})
boringssl_apply_common_options(ssl_test)
boringssl_apply_link_options(ssl_test)
boringssl_package_executable(ssl_test)
target_link_libraries(ssl_test PRIVATE test_support gtest_main ssl_static)

add_executable(bssl ${BSSL_ANDROID_SOURCES})
boringssl_apply_common_options(bssl)
boringssl_apply_link_options(bssl)
boringssl_package_executable(bssl)
target_link_libraries(bssl PRIVATE ssl_static)
