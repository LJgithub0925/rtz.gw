cmake_minimum_required(VERSION 3.10)
project(rtz.gw C ASM)
set(RTZ_VERSION "0.2.7")

set(OPENSSL_ROOT_DIR 3rdparty/openssl-1.1.0j)
set(OPENSSL_INCLUDE_DIRS ${OPENSSL_ROOT_DIR}/include)
find_library(ssl_LIBRARY ssl PATHS ${OPENSSL_ROOT_DIR}/lib NO_DEFAULT_PATH)
find_library(crypto_LIBRARY crypto PATHS ${OPENSSL_ROOT_DIR}/lib NO_DEFAULT_PATH)
set(OPENSSL_LIBRARIES ${ssl_LIBRARY} ${crypto_LIBRARY})
find_library(srtp2_LIBRARY srtp2 PATHS 3rdparty/libsrtp-2.2.0/lib NO_DEFAULT_PATH)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBUNWIND REQUIRED libunwind-generic)
pkg_check_modules(SYSTEMD REQUIRED libsystemd)

add_subdirectory(3rdparty/cJSON-1.7.10)
#add_subdirectory(3rdparty/libaco-1.2.4)
#add_subdirectory(3rdparty/mxml-2.12)

option(WITH_HTTP_HOOKS "enable http hooks" OFF)
option(WITH_WSS "enable signal server WSS mode" OFF)
option(WITH_ASAN "enable address sanitizer" ON)
option(WITH_TSAN "enable thread sanitizer" OFF)
option(WITH_TCMALLOC "link with tcmalloc" OFF)
option(WITH_ZOOKEEPER "enable zookeeper integration" OFF)

add_definitions(-Wall -Wextra -Wno-unused-variable -Wno-unused-parameter -Wno-unused-function)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/libsrtp-2.2.0/include
    ${CMAKE_CURRENT_SOURCE_DIR}/src
#    ${aco_INCLUDE_DIRS}
    ${cJSON_INCLUDE_DIRS}
#    ${mxml_INCLUDE_DIRS}
    ${OPENSSL_INCLUDE_DIRS}
    ${LIBUNWIND_INCLUDE_DIRS}
    )
add_definitions(-DRTZ_VERSION="${RTZ_VERSION}")
add_definitions(-pthread -D_GNU_SOURCE -DTHREADED=1)
#add_definitions(-fno-omit-frame-pointer -fkeep-inline-functions -fno-inline)
set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -no-pie")
set(rtz_SRCS
#    src/hook/aco_hook_syscall.c
    src/media/sdp.c
    src/media/h26x.c
    src/media/codec_types.c
    src/media/g711.c
    src/media/rtp_demux.c
    src/media/rtp_mux.c
    src/media/fmp4_mux.c
    src/media/flac_util.c
	src/net/net_util.c
    src/net/nbuf.c
    src/net/tcp_chan.c
    src/net/rtmp_types.c
    src/net/rtmp_handshake.c
    src/net/rtmp_client.c
    src/net/http_types.c
    src/net/rtz_server.c
    src/net/stun.c
    src/net/ice.c
    src/net/dtls.c
    src/net/dtls_bio.c
    src/net/hls_server.c
    src/net/rtp.c
    src/net/rtcp.c
    src/net/tcp_chan_ssl.c
    src/net/http_hooks.c
    src/net/monitor_server.c
    src/net/drift_tracer.c
	src/cbuf.c
	src/sbuf.c
	src/event_loop.c
    src/cfg_util.c
	src/log.c
	src/main.c
	src/mpsc_queue.c
    src/algo/base64.c
    src/algo/md5.c
    src/algo/sha1.c
    src/algo/hmac_sha1.c
    src/algo/crc32.c
    src/timestamp.c
    src/apierror.c
    src/crash_util.c
    src/watchdog.c
    src/zk_util.c
    src/rtz_shard.c
    src/sched_util.c
	)
add_executable(rtz.gw ${rtz_SRCS})
if(WITH_WSS)
    add_definitions(-DWITH_WSS=1)
endif()
if(WITH_HTTP_HOOKS)
    add_definitions(-DWITH_HTTP_HOOKS=1)
endif()
if(WITH_ZOOKEEPER)
    find_library(zk_LIBRARY zookeeper_mt PATHS 3rdparty/zookeeper-3.5.4/lib NO_DEFAULT_PATH)
    add_definitions(-DWITH_ZOOKEEPER=1)
    include_directories(
        ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/zookeeper-3.5.4/include
        )
    target_link_libraries(rtz.gw ${zk_LIBRARY})
endif()
if(WITH_ASAN)
    add_definitions(-fsanitize=address)
    target_link_libraries(rtz.gw asan)
elseif(WITH_TSAN)
    add_definitions(-fsanitize=thread)
    target_link_libraries(rtz.gw tsan)
endif()
if(WITH_TCMALLOC)
    target_link_libraries(rtz.gw tcmalloc_and_profiler)
endif()

if(WITH_SYSTEMD)
    add_definitions(-DWITH_SYSTEMD=1)
    target_link_libraries(rtz.gw ${SYSTEMD_LIBRARIES})
endif()
target_link_libraries(rtz.gw
	${cJSON_LIBRARIES}
#	${aco_LIBRARIES}
#    ${mxml_LIBRARIES}
    ${OPENSSL_LIBRARIES}
    ${srtp2_LIBRARY}
    ${LIBUNWIND_STATIC_LIBRARIES}
	pthread
    m
    dl
	)
install(TARGETS rtz.gw DESTINATION /app/avideo)
install(FILES
    dist/rtz.pem
    dist/rtz.key
    dist/rtz.aliyun-tob.conf.sample
    DESTINATION /app/avideo)
install(FILES dist/70-rtz.gw.conf DESTINATION /etc/rsyslog.d)
install(FILES dist/rtz.gw@.service DESTINATION /etc/systemd/system)
install(FILES dist/rtz.logrotate DESTINATION /etc/logrotate.d RENAME rtz)

set(CPACK_GENERATOR "DEB")
set(CPACK_SOURCE_GENERATOR "TXZ")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Wang Hoi")
set(CPACK_PACKAGE_VERSION ${RTZ_VERSION})
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libgoogle-perftools4 (>= 2.4-0), libsystemd0")
set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_SOURCE_DIR}/dist/postinst;${CMAKE_CURRENT_SOURCE_DIR}/dist/prerm;${CMAKE_CURRENT_SOURCE_DIR}/dist/postrm;")
set(CPACK_SOURCE_IGNORE_FILES "/build/;/\.git/;/\.gitignore;/.*\.tmp")
include(CPack)

################################################################################

set(rtz_bench_SRCS
    src/bench/rtz_bench.c
    src/bench/rtz_client.c
    src/media/sdp.c
    src/media/h26x.c
    src/media/codec_types.c
    src/media/g711.c
    src/media/rtp_demux.c
    src/media/rtp_mux.c
    src/net/dtls.c
    src/net/dtls_bio.c
    src/net/ice.c
	src/net/net_util.c
    src/net/nbuf.c
    src/net/tcp_chan.c
    src/net/tcp_chan_ssl.c
    src/net/http_types.c
    src/net/stun.c
    src/net/rtp.c
    src/net/rtcp.c
	src/cbuf.c
	src/sbuf.c
	src/event_loop.c
    src/cfg_util.c
    src/crash_util.c
	src/log.c
	src/mpsc_queue.c
    src/algo/base64.c
    src/algo/md5.c
    src/algo/sha1.c
    src/algo/hmac_sha1.c
    src/algo/crc32.c
    src/timestamp.c
    src/sched_util.c
	)
add_executable(rtz_bench ${rtz_bench_SRCS})
if(WITH_ASAN)
    add_definitions(-fsanitize=address)
    target_link_libraries(rtz_bench asan)
elseif(WITH_TSAN)
    add_definitions(-fsanitize=thread)
    target_link_libraries(rtz_bench tsan)
endif()
if(WITH_TCMALLOC)
    target_link_libraries(rtz_bench tcmalloc_and_profiler)
endif()
target_link_libraries(rtz_bench
	${cJSON_LIBRARIES}
#	${aco_LIBRARIES}
#    ${mxml_LIBRARIES}
    ${OPENSSL_LIBRARIES}
    ${srtp2_LIBRARY}
    ${LIBUNWIND_STATIC_LIBRARIES}
	pthread
    m
    dl
	)
