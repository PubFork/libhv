cmake_minimum_required(VERSION 3.0)

project(hv VERSION 1.20)

# TARGET_TYPE = SHARED or STATIC
set(TARGET_TYPE SHARED)

# see config.mk
option(WITH_PROTOCOL "compile protocol" ON)

option(WITH_HTTP "compile http" ON)
option(WITH_HTTP_SERVER "compile http/server" ON)
option(WITH_HTTP_CLIENT "compile http/client" ON)

# WITH_CONSUL need WITH_HTTP_CLIENT=ON
option(WITH_CONSUL "compile consul" OFF)

option(ENABLE_IPV6 "ipv6" OFF)
option(ENABLE_UDS "Unix Domain Socket" OFF)
option(ENABLE_WINDUMP "Windows MiniDumpWriteDump" OFF)
option(USE_MULTIMAP "MultiMap" OFF)

option(WITH_CURL "with curl library" OFF)
option(WITH_NGHTTP2 "with nghttp2 library" OFF)
option(WITH_OPENSSL "with openssl library" OFF)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMake;${CMAKE_MODULE_PATH}")
include(utils)
include(vars)

# see configure
# Checks for header files
check_header("stdint.h")
check_header("stdbool.h")
check_header("sys/types.h")
check_header("sys/stat.h")
check_header("sys/time.h")
check_header("fcntl.h")
check_header("pthread.h")

# Checks for functions
set(CMAKE_REQUIRED_LIBRARIES "pthread")
check_function("gettid" "unistd.h")
check_function("strlcpy" "string.h")
check_function("strlcat" "string.h")
check_function("clock_gettime" "time.h")
check_function("gettimeofday" "sys/time.h")
check_function("pthread_spin_lock" "pthread.h")
check_function("pthread_mutex_timedlock" "pthread.h")
check_function("sem_timedwait" "semaphore.h")

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/hconfig.h.in ${CMAKE_CURRENT_SOURCE_DIR}/hconfig.h)

# see Makefile.in
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED True)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

set(INCDIR include)
set(SRCDIR src)
set(LIBDIR lib)
set(BINDIR bin)
set(LIBRARY_OUTPUT_PATH ${LIBDIR})
set(EXECUTABLE_OUTPUT_PATH ${BINDIR})

set(INCDIRS . include 3rd/include)
set(LIBDIRS . lib 3rd/lib)
include_directories(${INCDIRS} ${SRCDIR})
link_directories(${LIBDIRS})

if(ENABLE_IPV6)
    add_definitions(-DENABLE_IPV6)
endif()

if(ENABLE_UDS)
    add_definitions(-DENABLE_UDS)
endif()

if(USE_MULTIMAP)
    add_definitions(-DUSE_MULTIMAP)
endif()

if(WITH_CURL)
    add_definitions(-DWITH_CURL)
    set(LIBS ${LIBS} curl)
    if(WIN32)
        set(LIBS ${LIBS} wldap32 advapi32 crypt32)
    endif()
endif()

if(WITH_NGHTTP2)
    add_definitions(-DWITH_NGHTTP2)
    set(LIBS ${LIBS} nghttp2)
endif()

if(WITH_OPENSSL)
    add_definitions(-DWITH_OPENSSL)
    set(LIBS ${LIBS} ssl crypto)
endif()

if(WIN32)
    add_definitions(-D_WIN32_WINNT=0x0600)
    set(LIBS ${LIBS} winmm iphlpapi ws2_32)
    if(ENABLE_WINDUMP)
        add_definitions(-DENABLE_WINDUMP)
        set(LIBS ${LIBS} dbghelp)
    endif()
endif()

if(UNIX)
    set(LIBS ${LIBS} pthread m dl)
    if(CMAKE_COMPILER_IS_GNUCC)
        set(LIBS ${LIBS} rt)
    endif()
endif()

if(ANDROID)
    set(LIBS ${LIBS} log)
endif()

# see Makefile
set(ALL_SRCDIRS . base utils event protocol http http/client http/server consul examples)
set(LIBHV_SRCDIRS . base utils event)
set(LIBHV_HEADERS hv.h hconfig.h)
set(LIBHV_HEADERS ${LIBHV_HEADERS} ${BASE_HEADERS} ${UTILS_HEADERS} ${EVENT_HEADERS})

if(WITH_PROTOCOL)
    set(LIBHV_HEADERS ${LIBHV_HEADERS} ${PROTOCOL_HEADERS})
    set(LIBHV_SRCDIRS ${LIBHV_SRCDIRS} protocol)
endif()

if(WITH_HTTP)
    set(LIBHV_HEADERS ${LIBHV_HEADERS} ${HTTP_HEADERS})
    set(LIBHV_SRCDIRS ${LIBHV_SRCDIRS} http)
    if(WITH_HTTP_SERVER)
        set(LIBHV_HEADERS ${LIBHV_HEADERS} ${HTTP_SERVER_HEADERS})
        set(LIBHV_SRCDIRS ${LIBHV_SRCDIRS} http/server)
    endif()
    if(WITH_HTTP_CLIENT)
        set(LIBHV_HEADERS ${LIBHV_HEADERS} ${HTTP_CLIENT_HEADERS})
        set(LIBHV_SRCDIRS ${LIBHV_SRCDIRS} http/client)
        if(WITH_CONSUL)
            set(LIBHV_HEADERS ${LIBHV_HEADERS} ${CONSUL_HEADERS})
            set(LIBHV_SRCDIRS ${LIBHV_SRCDIRS} consul)
        endif()
    endif()
endif()

list_source_directories(LIBHV_SRCS ${LIBHV_SRCDIRS})
add_library(hv ${TARGET_TYPE} ${LIBHV_SRCS})
target_include_directories(hv PRIVATE ${LIBHV_SRCDIRS})
target_link_libraries(hv ${LIBS})

install(FILES ${LIBHV_HEADERS} DESTINATION include/hv)
install(TARGETS hv)

add_subdirectory(unittest)
add_subdirectory(examples)

add_custom_target(libhv   DEPENDS hv)
add_custom_target(default DEPENDS hv)
