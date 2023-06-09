#
# INTEL CONFIDENTIAL
#
# Copyright 2019 Intel Corporation.
#
# This software and the related documents are Intel copyrighted materials, and
# your use of them is governed by the express license under which they were
# provided to you ("License"). Unless the License provides otherwise, you may
# not use, modify, copy, publish, distribute, disclose or transmit this software
# or the related documents without Intel's prior written permission.
#
# This software and the related documents are provided as is, with no express or
# implied warranties, other than those that are expressly stated in the License.
#


cmake_minimum_required (VERSION 3.6)
option (YOCTO_DEPENDENCIES "Use YOCTO dependencies system" OFF)
option (CRASHDUMP_BUILD_UT "Enable Building Unit tests" OFF)

# optional features
option (NVD_SECTION "Add NVD section to the crashdump contents." OFF)
option (TRIAGE_SECTION "Add triage section to the crashdump contents." OFF)
option (BAFI_NDA_OUTPUT
        "See docs/build_instructions.md before using this feature." OFF
)
option (BHS_EGS_BUILD
        "For Whitley build turn this OFF, see
        docs/build_instructions.md for more information." ON
)

project (crashdump)
include (ExternalProject)
if (${YOCTO_DEPENDENCIES})
    find_package (PkgConfig REQUIRED)
    pkg_check_modules (SAFEC REQUIRED libsafec)
    include_directories (${SAFEC_INCLUDE_DIRS})
    link_directories (${SAFEC_LIBRARY_DIRS})
endif ()

# Set the searching location for cmake 'include' locations
set (CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}")
# Include the sanitizer module
include (code-coverage)

set (CMAKE_CXX_STANDARD 20)
set (CMAKE_CXX_STANDARD_REQUIRED ON)
set (EXTERNAL_PACKAGES Boost cjson-project sdbusplus-project safeclib-project
                       libpeci-project
)

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU") # require at least gcc 8.0
    if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 8.0)
        message (FATAL_ERROR "GCC version must be at least 8.0")
    endif ()
endif ()

if (NOT ${YOCTO_DEPENDENCIES})

    externalproject_add (
        Boost
        URL https://downloads.yoctoproject.org/mirror/sources/boost_1_73_0.tar.bz2
        URL_MD5 9273c8c4576423562bbe84574b07b2bd
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
        INSTALL_COMMAND ""
        DOWNLOAD_COMMAND ""
    )
    externalproject_get_property (Boost SOURCE_DIR)
    include_directories (SYSTEM ${SOURCE_DIR})
    set (CMAKE_PREFIX_PATH ${SOURCE_DIR} ${CMAKE_PREFIX_PATH})

    # requires apt install autoconf-archive and autoconf
    externalproject_add (
        sdbusplus-project
        PREFIX ${CMAKE_BINARY_DIR}/sdbusplus-project
        GIT_REPOSITORY https://github.com/openbmc/sdbusplus.git
        GIT_TAG 6adfe948ee55ffde8457047042d0d55aa3d8b4a7
        SOURCE_DIR ${CMAKE_BINARY_DIR}/sdbusplus-src
        BINARY_DIR ${CMAKE_BINARY_DIR}/sdbusplus-build
        CONFIGURE_COMMAND ""
        BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/sdbusplus-src && meson build
                      -Ddefault_library=static && ninja -C build libsdbusplus.a
        INSTALL_COMMAND ""
        LOG_DOWNLOAD ON
    )
    include_directories (SYSTEM ${CMAKE_BINARY_DIR}/sdbusplus-src/include)
    link_directories (${CMAKE_BINARY_DIR}/sdbusplus-src/build)

    externalproject_add (
        safeclib-project
        GIT_REPOSITORY https://github.com/rurban/safeclib.git
        GIT_TAG 5da8464c092b2f234442824d3dbb49343e58bb16
        SOURCE_DIR "${CMAKE_BINARY_DIR}/safeclib-src"
        BINARY_DIR "${CMAKE_BINARY_DIR}/safeclib-build"
        INSTALL_DIR "${CMAKE_BINARY_DIR}/safeclib-install"
        UPDATE_COMMAND ""
        CONFIGURE_COMMAND
            cd ${CMAKE_BINARY_DIR}/safeclib-src/ && ./build-aux/autogen.sh &&
            ./configure --prefix=${CMAKE_BINARY_DIR}/safeclib-install
        BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/safeclib-src && make -j 10
        INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/safeclib-src && make install
        LOG_DOWNLOAD ON
    )
    include_directories (
        SYSTEM ${CMAKE_BINARY_DIR}/safeclib-install/include/libsafec
    )
    externalproject_get_property (safeclib-project INSTALL_DIR)
    add_library (safec SHARED IMPORTED)
    set_target_properties (
        safec PROPERTIES IMPORTED_LOCATION ${INSTALL_DIR}/lib/libsafec-3.6.0.so
    )

    externalproject_add (
        libpeci-project
        PREFIX ${CMAKE_BINARY_DIR}/libpeci-project
        GIT_REPOSITORY https://github.com/openbmc/libpeci.git
        GIT_TAG 7b11280d8e3113aecc4b9ce6e5d818268eb2122c
        INSTALL_COMMAND ""
        # For this step to work, meta-restricted repository needs to be clone at
        # same level as openbmc-crashdump
        PATCH_COMMAND
            git apply
            ${CMAKE_BINARY_DIR}/../../meta-restricted/recipes-core/libpeci/libpeci/0002-Add-PECI-Telemetry-command.patch
        LOG_DOWNLOAD ON
    )
    externalproject_get_property (libpeci-project SOURCE_DIR)
    include_directories (${SOURCE_DIR})
    externalproject_get_property (libpeci-project BINARY_DIR)
    add_library (peci SHARED IMPORTED)
    set_target_properties (
        peci PROPERTIES IMPORTED_LOCATION ${BINARY_DIR}/libpeci.so
    )

    externalproject_add (
        cjson-project
        GIT_REPOSITORY https://github.com/DaveGamble/cJSON.git
        GIT_TAG v1.7.12
        SOURCE_DIR "${CMAKE_BINARY_DIR}/cjson-src"
        BINARY_DIR "${CMAKE_BINARY_DIR}/cjson-build"
        INSTALL_DIR "${CMAKE_BINARY_DIR}/cjson-install"
        CONFIGURE_COMMAND
            cd ${CMAKE_BINARY_DIR}/cjson-src/ && cmake . -DENABLE_CJSON_UTILS=On
            -DENABLE_CJSON_TEST=Off
            -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/cjson-install
        BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/cjson-src && make -j 10
        INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/cjson-src && make install
        LOG_DOWNLOAD ON
    )
    include_directories (SYSTEM ${CMAKE_BINARY_DIR}/cjson-install/include/)
    externalproject_get_property (cjson-project INSTALL_DIR)
    add_library (cjson SHARED IMPORTED)
    set_target_properties (
        cjson PROPERTIES IMPORTED_LOCATION ${INSTALL_DIR}/lib/libcjson.so
    )

endif ()

if (NOT ${YOCTO_DEPENDENCIES})
    set (
        CMAKE_CXX_FLAGS
        "${CMAKE_CXX_FLAGS} -Wno-cpp -Wno-write-strings -Werror \
-Wall \
-Wuninitialized \
-Wunused-parameter \
-Wshift-negative-value \
-Wuninitialized \
-Wtype-limits \
-Wsign-compare \
-Wmissing-field-initializers \
-Wimplicit-fallthrough=3 \
-Wempty-body \
-Wignored-qualifiers \
-Wclobbered \
-Wcast-function-type \
"
    )
endif ()

if (${YOCTO_DEPENDENCIES})
    set (
        CMAKE_CXX_FLAGS
        "${CMAKE_CXX_FLAGS} -Wno-cpp -Wno-write-strings -Werror \
-Wall \
-Wuninitialized \
-Wunused-parameter \
-Wshift-negative-value \
-Wuninitialized \
-Wtype-limits \
-Wredundant-move \
-Wstring-compare \
-Wsign-compare \
-Wdeprecated-copy \
-Wmissing-field-initializers \
-Wimplicit-fallthrough=3 \
-Wempty-body \
-Wignored-qualifiers \
-Wclobbered \
-Wcast-function-type \
-DBHS_EGS_BUILD=NO"
    )
endif ()

set (
    SRC_FILES
    engine/TorDump.c
    engine/BigCore.c
    engine/utils.c
    engine/crashdump.c
    engine/cmdprocessor.c
    engine/logger.c
    engine/inputparser.c
    engine/validator.c
    engine/flow.c
)

if (${NVD_SECTION})
    set (SRC_FILES ${SRC_FILES} engine/nvd.c)
    add_definitions (-DNVD_SECTION)
    set (NVD_LIB fis)
endif ()

if (${TRIAGE_SECTION} OR ${BAFI_NDA_OUTPUT})
    set (SRC_FILES ${SRC_FILES} utils_triage.cpp)
    if (${TRIAGE_SECTION})
        add_definitions (-DTRIAGE_SECTION)
    endif ()
    if (${BAFI_NDA_OUTPUT})
        add_definitions (-DBAFI_NDA_OUTPUT)
    endif ()
endif ()
if (${BHS_EGS_BUILD})
    set (SRC_FILES ${SRC_FILES} engine/Crashlog.c
                                engine/base64Encode.c
    )
    add_definitions (-DBHS_EGS_BUILD)
endif ()

if (NOT ${CRASHDUMP_BUILD_UT})
    add_definitions (-DBUILD_MAIN)
    add_executable (crashdump crashdump.cpp utils_dbusplus.cpp ${SRC_FILES})

    if (NOT ${YOCTO_DEPENDENCIES})
        add_dependencies (crashdump ${EXTERNAL_PACKAGES})
        target_link_libraries (crashdump safec sdbusplus peci cjson)
    endif ()

    option (
        CRASHDUMP_PRINT_UNFORMATTED
        "Use cJSON_PrintUnformatted() to print the crashdump contents without
            whitespace to reduce the file size." OFF
    )

    option (OEMDATA_SECTION
            "Adds oemdata Section to the crashdump contents, for OEM usage."
            OFF
    )

    target_compile_definitions (
        crashdump
        PRIVATE $<$<BOOL:${CRASHDUMP_PRINT_UNFORMATTED}>:
                -DCRASHDUMP_PRINT_UNFORMATTED> $<$<BOOL:${OEMDATA_SECTION}>:
                -DOEMDATA_SECTION>
    )

    target_include_directories (crashdump PRIVATE ${CMAKE_SOURCE_DIR})
    set_property (TARGET crashdump PROPERTY C_STANDARD 99)
    target_link_libraries (
        crashdump
        cjson
        peci
        -lsystemd
        pthread
        -lstdc++fs
        sdbusplus
        ${SAFEC_LIBRARIES}
        ${NVD_LIB}
    )

    install (
        TARGETS crashdump
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib/static
    )
endif ()

install (
    FILES ${CMAKE_CURRENT_SOURCE_DIR}/crashdump_input_icx.json
          ${CMAKE_CURRENT_SOURCE_DIR}/telemetry_input_icx.json
          ${CMAKE_CURRENT_SOURCE_DIR}/crashdump_input_spr.json
          ${CMAKE_CURRENT_SOURCE_DIR}/telemetry_input_spr.json
          ${CMAKE_CURRENT_SOURCE_DIR}/crashdump_input_sprhbm.json
          ${CMAKE_CURRENT_SOURCE_DIR}/telemetry_input_sprhbm.json
    PERMISSIONS OWNER_READ
    DESTINATION /usr/share/crashdump/input/
)

find_package (Boost 1.73 REQUIRED)
include_directories (${BOOST_SRC_DIR})

add_definitions (-DBOOST_ERROR_CODE_HEADER_ONLY)
add_definitions (-DBOOST_SYSTEM_NO_DEPRECATED)
add_definitions (-DBOOST_ALL_NO_LIB)
add_definitions (-DBOOST_NO_RTTI)
add_definitions (-DBOOST_NO_TYPEID)

# Unit Tests
if (NOT ${YOCTO_DEPENDENCIES})
    if (${CRASHDUMP_BUILD_UT})
        externalproject_add (
            googletest-project
            GIT_REPOSITORY "https://github.com/google/googletest.git"
            GIT_TAG 6b74da4757a549563d7c37c8fae3e704662a043b
            CMAKE_ARGS
                -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/googletest-build
            SOURCE_DIR "${CMAKE_BINARY_DIR}/googletest-src"
            BINARY_DIR "${CMAKE_BINARY_DIR}/googletest-build"
        )
        externalproject_get_property (googletest-project BINARY_DIR)
        include_directories (${BINARY_DIR}/include)
        externalproject_get_property (googletest-project BINARY_DIR)
        add_library (gmock UNKNOWN IMPORTED)
        set_target_properties (
            gmock
            PROPERTIES "IMPORTED_LOCATION" "${BINARY_DIR}/lib/libgmock.a"
                       "IMPORTED_LINK_INTERFACE_LIBRARIES"
                       "${CMAKE_THREAD_LIBS_INIT}"
        )
        add_library (gtest UNKNOWN IMPORTED)
        set_target_properties (
            gtest
            PROPERTIES "IMPORTED_LOCATION" "${BINARY_DIR}/lib/libgtest.a"
                       "IMPORTED_LINK_INTERFACE_LIBRARIES"
                       "${CMAKE_THREAD_LIBS_INIT}"
        )

        enable_testing ()

        include (CTest)
        include_directories (${CMAKE_SOURCE_DIR})

        set (
            UT_FILES
            tests/mock/test_crashdump.cpp
            tests/test_utils.cpp
            tests/test_main.cpp
            tests/engine/utils_test.cpp
            tests/engine/cmdprocessor_test.cpp
            tests/engine/TorDump_test.cpp
            tests/engine/BigCore_test.cpp
            tests/engine/logger_test.cpp
            tests/engine/inputparser_test.cpp
            tests/engine/validator_test.cpp
            tests/engine/flow_test.cpp
            tests/engine/Crashlog_test.cpp
            tests/engine/MaxSize_test.cpp
            tests/engine/crashdump_test.cpp
        )

        add_definitions (-DCOMPILE_UNIT_TESTS)

        add_executable (
            crashdump_ut crashdump.cpp utils_dbusplus.cpp ${SRC_FILES}
                         ${UT_FILES}
        )
        add_dependencies (crashdump_ut ${EXTERNAL_PACKAGES} googletest-project)

        target_link_libraries (
            crashdump_ut
            PRIVATE cjson
                    peci
                    -lsystemd
                    pthread
                    -lstdc++fs
                    safec
                    sdbusplus
                    gmock
                    gtest
        )

        add_test (test_crashdump_ut crashdump_ut
                  "--gtest_output=xml:crashdump_test.xml"
        )

        crashdump_code_coverage (crashdump_ut)
    endif ()
endif ()

set (SERVICE_FILES
     ${PROJECT_SOURCE_DIR}/service_files/com.intel.crashdump.service
)

install (FILES ${SERVICE_FILES} DESTINATION /lib/systemd/system/)
