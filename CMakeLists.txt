CMAKE_MINIMUM_REQUIRED(VERSION 3.1)
PROJECT(Snappy VERSION 1.1.6 LANGUAGES C CXX)

OPTION (ENABLE_TESTS "Enable building tests and examples")
OPTION (ENABLE_SO_VERSION "Enable shared library version" ON)
OPTION (ENABLE_SHARED "Disable building shared" ON)
OPTION (ENABLE_STATIC "Disable building static")

IF (NOT ENABLE_SHARED AND NOT ENABLE_STATIC)
    MESSAGE (FATAL_ERROR "It is invalid that both ENABLE_SHARED and ENABLE_STATIC are OFF.")
ENDIF (NOT ENABLE_SHARED AND NOT ENABLE_STATIC)

SET(CMAKE_INCLUDE_CURRENT_DIR ON)

INCLUDE(CheckIncludeFiles)
INCLUDE(CheckLibraryExists)
INCLUDE(CheckCXXSourceCompiles)
INCLUDE(TestBigEndian)
INCLUDE(CMakePackageConfigHelpers)

TEST_BIG_ENDIAN(WORDS_BIG_ENDIAN)
IF (WORDS_BIG_ENDIAN)
    MESSAGE(STATUS "Building on big endian system")
    ADD_DEFINITIONS(-DWORDS_BIGENDIAN=1)
ENDIF (WORDS_BIG_ENDIAN)

CHECK_INCLUDE_FILES("byteswap.h" HAVE_BYTESWAP_H)
CHECK_INCLUDE_FILES("dlfcn.h" HAVE_DLFCN_H)
CHECK_INCLUDE_FILES("inttypes.h" HAVE_INTTYPES_H)
CHECK_INCLUDE_FILES("memory.h" HAVE_MEMORY_H)
CHECK_INCLUDE_FILES("stddef.h" HAVE_STDDEF_H)
CHECK_INCLUDE_FILES("stdint.h" HAVE_STDINT_H)
CHECK_INCLUDE_FILES("stdlib.h" HAVE_STDLIB_H)
CHECK_INCLUDE_FILES("strings.h" HAVE_STRINGS_H)
CHECK_INCLUDE_FILES("string.h" HAVE_STRING_H)
CHECK_INCLUDE_FILES("sys/byteswap.h" HAVE_SYS_BYTESWAP_H)
CHECK_INCLUDE_FILES("sys/endian.h" HAVE_SYS_ENDIAN_H)
CHECK_INCLUDE_FILES("sys/mman.h" HAVE_SYS_MMAN_H)
CHECK_INCLUDE_FILES("sys/resource.h" HAVE_SYS_RESOURCE_H)
CHECK_INCLUDE_FILES("sys/stat.h" HAVE_SYS_STAT_H)
CHECK_INCLUDE_FILES("sys/time.h" HAVE_SYS_TIME_H)
CHECK_INCLUDE_FILES("sys/types.h" HAVE_SYS_TYPES_H)
CHECK_INCLUDE_FILES("sys/uio.h" HAVE_SYS_UIO_H)
CHECK_INCLUDE_FILES("unistd.h" HAVE_UNISTD_H)
CHECK_INCLUDE_FILES("windows.h" HAVE_WINDOWS_H)

IF (NOT HAVE_SYS_UIO_H)
    SET(HAVE_SYS_UIO_H 0)
ENDIF (NOT HAVE_SYS_UIO_H)

IF (NOT HAVE_STDINT_H)
    SET(HAVE_STDINT_H 0)
ENDIF (NOT HAVE_STDINT_H)

IF (NOT HAVE_STDDEF_H)
    SET(HAVE_STDDEF_H 0)
ENDIF (NOT HAVE_STDDEF_H)

CHECK_CXX_SOURCE_COMPILES("int main(void) { return __builtin_expect(0, 1); }"
        HAVE_BUILTIN_EXPECT)

CHECK_CXX_SOURCE_COMPILES("int main(void) { return __builtin_ctzll(0); }"
        HAVE_BUILTIN_CTZ)

CONFIGURE_FILE(${Snappy_SOURCE_DIR}/cmake/config.h.in config.h)

# Configure snappy-stubs-public.h.in
SET(ac_cv_have_stdint_h ${HAVE_STDINT_H})
SET(ac_cv_have_stddef_h ${HAVE_STDDEF_H})
SET(ac_cv_have_sys_uio_h ${HAVE_SYS_UIO_H})
CONFIGURE_FILE(${Snappy_SOURCE_DIR}/snappy-stubs-public.h.in
               snappy-stubs-public.h)

IF (WIN32)
    ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS)
ENDIF (WIN32)

SET (snappy_SRCS
    snappy-c.cc
    snappy-c.h
    snappy-sinksource.cc
    snappy-sinksource.h
    snappy-stubs-internal.cc
    snappy-stubs-public.h
    snappy.cc
    snappy.h
)

# Define the main library.
IF (ENABLE_SHARED)
    ADD_LIBRARY(snappy SHARED ${snappy_SRCS})
    TARGET_COMPILE_DEFINITIONS(snappy PRIVATE -DHAVE_CONFIG_H)

    SET_TARGET_PROPERTIES(snappy PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)

    IF (ENABLE_SO_VERSION)
        SET_TARGET_PROPERTIES(snappy PROPERTIES VERSION ${PROJECT_VERSION}
                              SOVERSION ${PROJECT_VERSION_MAJOR})
    ENDIF (ENABLE_SO_VERSION)

    INSTALL(TARGETS snappy
            EXPORT SnappyTargets
            RUNTIME DESTINATION bin
            LIBRARY DESTINATION lib${LIB_SUFFIX}
            ARCHIVE DESTINATION lib${LIB_SUFFIX})
    INSTALL(EXPORT SnappyTargets NAMESPACE Snappy:: DESTINATION lib/cmake/Snappy)
ENDIF (ENABLE_SHARED)

IF (ENABLE_STATIC)
    ADD_LIBRARY (snappy_s STATIC ${snappy_SRCS})
    TARGET_COMPILE_DEFINITIONS(snappy_s PRIVATE -DHAVE_CONFIG_H)

    IF (UNIX)
        IF (CMAKE_COMPILER_IS_GNUCC)
            SET_TARGET_PROPERTIES (snappy_s PROPERTIES COMPILE_FLAGS "-fPIC")
        ENDIF (CMAKE_COMPILER_IS_GNUCC)

        IF (CMAKE_SYSTEM_NAME STREQUAL "AIX")
            SET (CMAKE_C_ARCHIVE_CREATE
                 "<CMAKE_AR> -X32_64 cr <TARGET> <LINK_FLAGS> <OBJECTS>")
        ENDIF (CMAKE_SYSTEM_NAME STREQUAL "AIX")
    ENDIF (UNIX)

    SET_TARGET_PROPERTIES (snappy_s PROPERTIES OUTPUT_NAME snappy)

    # at build time you may specify the cmake variable LIB_SUFFIX to handle
    # 64-bit systems which use 'lib64'
    INSTALL (TARGETS snappy_s ARCHIVE DESTINATION lib${LIB_SUFFIX})
ENDIF (ENABLE_STATIC)

INSTALL(FILES snappy.h
        snappy-c.h
        snappy-sinksource.h
        ${Snappy_BINARY_DIR}/snappy-stubs-public.h
        DESTINATION include)

SET(INCLUDE_INSTALL_DIR include)
SET(LIBRARY_INSTALL_DIR lib)
SET(BINARY_INSTALL_DIR bin)

CONFIGURE_PACKAGE_CONFIG_FILE(cmake/SnappyConfig.cmake.in
        ${Snappy_BINARY_DIR}/SnappyConfig.cmake
        INSTALL_DESTINATION lib/Snappy/cmake
        PATH_VARS INCLUDE_INSTALL_DIR LIBRARY_INSTALL_DIR BINARY_INSTALL_DIR
        )

WRITE_BASIC_PACKAGE_VERSION_FILE(${Snappy_BINARY_DIR}/SnappyConfigVersion.cmake
        COMPATIBILITY SameMajorVersion)
INSTALL(FILES ${Snappy_BINARY_DIR}/SnappyConfig.cmake
        ${Snappy_BINARY_DIR}/SnappyConfigVersion.cmake
        DESTINATION lib/cmake)


IF (ENABLE_TESTS)
    CHECK_LIBRARY_EXISTS(z zlibVersion "" HAVE_LIBZ)
    CHECK_LIBRARY_EXISTS(lzo2 lzo1x_1_15_compress "" HAVE_LIBLZO2)

    FIND_PACKAGE(GTest QUIET)
    IF(GTEST_FOUND)
        SET(HAVE_GTEST 1)
    ENDIF()

    FIND_PACKAGE(Gflags QUIET)
    IF(GFLAGS_FOUND)
        SET(HAVE_GFLAGS 1)
    ENDIF()

    ENABLE_TESTING()

    IF (HAVE_LIBZ)
        LIST(APPEND COMPRESSION_LIBS z)
    ENDIF (HAVE_LIBZ)

    IF (HAVE_LIBLZO2)
        LIST(APPEND COMPRESSION_LIBS lzo2)
    ENDIF (HAVE_LIBLZO2)

    IF (HAVE_LIBLZF)
        LIST(APPEND COMPRESSION_LIBS lzf)
    ENDIF (HAVE_LIBLZF)

    IF (HAVE_LIBQUICKLZ)
        LIST(APPEND COMPRESSION_LIBS quicklz)
    ENDIF (HAVE_LIBQUICKLZ)

    ADD_EXECUTABLE(snappy-unittest snappy_unittest.cc snappy-test.cc)
    TARGET_COMPILE_DEFINITIONS(snappy-unittest PRIVATE -DHAVE_CONFIG_H)
    TARGET_LINK_LIBRARIES(snappy-unittest snappy ${COMPRESSION_LIBS}
                          ${GFLAGS_LIBRARIES})
    TARGET_INCLUDE_DIRECTORIES(snappy-unittest BEFORE PRIVATE ${Snappy_SOURCE_DIR}
                               ${GTEST_INCLUDE_DIRS} ${GFLAGS_INCLUDE_DIRS})

    ADD_TEST(NAME snappy-unittest
             WORKING_DIRECTORY ${Snappy_SOURCE_DIR}
             COMMAND ${Snappy_BINARY_DIR}/snappy-unittest)
ENDIF (ENABLE_TESTS)
# vim: set ts=4 sw=4 sts=4 et:
