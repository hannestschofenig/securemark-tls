cmake_minimum_required(VERSION 3.10.2)

cmake_policy(SET CMP0048 NEW)

project(sec-tls
	DESCRIPTION "Firmware for the SecureMark-TLS Self-Hosted Benchmark"
	LANGUAGES C
	VERSION 1.0.2)

option(SELFHOSTED "Set to '1' to build self-hosted binary, '0' by default makes a library" 1)
option(PSA_CRYPTO "Set to '1' to build benchmark with the PSA Crypto API; default is '0'" 1)

include(CMakeFiles.txt)
include(CheckIncludeFile)
include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckLibraryExists)
include(CheckSymbolExists)

include(ExternalProject)

include(cmake/splint.cmake)

# Cross compiling with cmake (a possible example)
#set(tools /Users/ptorelli/dev/gcc-arm-none-eabi-8-2018-q4-major)
#set(CMAKE_C_COMPILER ${tools}/bin/arm-none-eabi-gcc)
#set(CMAKE_RANLIB ${tools}/bin/arm-none-eabi-ranlib)
#set(CMAKE_AR ${tools}/bin/arm-none-eabi-ar)
#set(CMAKE_LINKER ${tools}/bin/arm-none-eabi-ld)
#set(CMAKE_STRIP ${tools}/bin/arm-none-eabi-strip)
#set(CMAKE_NM ${tools}/bin/arm-none-eabi-nm)
#set(CMAKE_OBJDUMP ${tools}/bin/arm-none-eabi-objdump)

#set(CMAKE_C_FLAGS "-pedantic -Wall -O3")

include_directories(../../monitor)
include_directories(../../monitor/th_api)
include_directories(../../profile)

if (SELFHOSTED)
    include_directories(profile/th_api)

    if (PSA_CRYPTO) 
		option(USE_PKCS11_HELPER_LIBRARY "Build mbed TLS with the pkcs11-helper library." OFF)
		option(ENABLE_ZLIB_SUPPORT "Build mbed TLS with zlib library." OFF)
		option(ENABLE_PROGRAMS "Build mbed TLS programs." OFF)
        option(ENABLE_TESTING "Build mbed TLS tests." OFF)
		
		# Enable PSA Crypto in the Mbed TLS Implementation
   	    add_definitions(-DMBEDTLS_PSA_CRYPTO_CONFIG=1)
#    	add_definitions(-DMBEDTLS_CONFIG_FILE="../../config/config-psa.h")

     	set(CMAKE_INSTALL_NAME_DIR ${CMAKE_INSTALL_FULL_LIBDIR}/lib)
        include_directories(../../mbedtls/include)
		add_subdirectory(../../mbedtls ./mbedtls)
		
		option(USE_STATIC_MBEDTLS_LIBRARY "Build mbed TLS static library." off)
		option(USE_SHARED_MBEDTLS_LIBRARY "Build mbed TLS shared library." on)
		
		if (CMAKE_SYSTEM MATCHES "OpenBSD")
			option(LINK_WITH_PTHREAD "Explicitly link mbed TLS library to pthread." on)
		endif()
		
		set(MBEDTLS_LIBRARIES "mbedtls" "mbedx509" "mbedcrypto")

        # The following three directives are used in the EEMBC benchmark.
		add_definitions(-DEE_CFG_SELFHOSTED=1)
		add_definitions(-DEE_CFG_QUIET=1)
		add_definitions(-DCRYPTO_PSA=1) 
		
		add_executable(sec-tls ${sectls_src_psa})

		target_link_libraries(sec-tls ${MBEDTLS_LIBRARIES})
		set_target_properties(sec-tls PROPERTIES INSTALL_RPATH mbedtls)
	else()
        include_directories(profile/th_api/redist)
		add_definitions(-DMBEDTLS_CONFIG_FILE="../../config/config-non-psa.h")
		add_definitions(-DEE_CFG_SELFHOSTED=1)
		add_definitions(-DEE_CFG_QUIET=1)
		add_definitions(-DCRYPTO_MBEDTLS=1)
		
		add_executable(sec-tls ${sectls_src})
	endif()		

else()
    include_directories(../../monitor/th_api)
    include_directories(../../profile/th_api)

	add_library(sec-tls ${sectls_src_no_main})

endif()

add_splint(sec-tls ${SOURCE})
