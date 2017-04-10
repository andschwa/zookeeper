cmake_minimum_required(VERSION 3.6)

project(zookeeper VERSION 3.5.3)
set(email user@zookeeper.apache.org)
set(description "zookeeper C client")

# general options
include_directories(include tests generated ${CMAKE_SOURCE_DIR} ${CMAKE_BINARY_DIR})
add_compile_options(-Wall)
if(NOT WIN32)
  add_compile_options(-Werror -fPIC)
endif()

# TODO: Enable /WX on Windows. Currently there are ~1000 warnings.
# TODO: Add Solaris support.
# TODO: Add debug configuration.

# Sync API option
option(WANT_SYNCAPI "Enables Sync API support" ON)
if(WANT_SYNCAPI)
  add_definitions(-DTHREADED)
endif()

# CppUnit option
option(WANT_CPPUNIT "Enables CppUnit" ON)

# header checks
function(to_have in out)
  string(TOUPPER ${in} str)
  string(REGEX REPLACE "/|\\." "_" str ${str})
  set(${out} "HAVE_${str}" PARENT_SCOPE)
endfunction()

# include file checks
foreach(f generated/zookeeper.jute.h generated/zookeeper.jute.c)
  if(EXISTS "${CMAKE_SOURCE_DIR}/${f}")
    to_have(${f} name)
    set(${name} 1)
  else()
    message(FATAL_ERROR
      "jute files are missing!\n"
      "Please run 'ant compile_jute' while in the zookeeper top level directory.")
  endif()
endforeach()

include(CheckIncludeFile)
set(check_headers
  arpa/inet.h
  dlfcn.h
  fcntl.h
  inttypes.h
  memory.h
  netdb.h
  netinet/in.h
  stdint.h
  stdlib.h
  string.h
  strings.h
  sys/socket.h
  sys/stat.h
  sys/time.h
  sys/types.h
  unistd.h
  sys/utsname.h)

foreach(f ${check_headers})
  to_have(${f} name)
  check_include_file(${f} ${name})
endforeach()

# function checks
include(CheckFunctionExists)
set(check_functions
  getcwd
  gethostbyname
  gethostname
  getlogin
  getpwuid_r
  gettimeofday
  getuid
  memmove
  memset
  poll
  socket
  strchr
  strdup
  strerror
  strtol)

foreach(fn ${check_functions})
  to_have(${fn} name)
  check_function_exists(${fn} ${name})
endforeach()

# library checks
to_have(librt name)
find_library(${name} rt)

# IPv6 check
include(CheckStructHasMember)
check_struct_has_member("struct sockaddr_in6" sin6_addr "netinet/in.h" ZOO_IPV6_ENABLED)

# configure
configure_file(cmake_config.h.in config.h)

# hashtable library
set(hashtable_sources src/hashtable/hashtable_itr.c src/hashtable/hashtable.c)
add_library(hashtable STATIC ${hashtable_sources})

# zookeeper library
set(zookeeper_sources
  src/zookeeper.c
  src/recordio.c
  generated/zookeeper.jute.c
  src/zk_log.c
  src/zk_hashtable.c
  src/addrvec.c)

if(WANT_SYNCAPI)
  list(APPEND zookeeper_sources src/mt_adaptor.c)
else()
  list(APPEND zookeeper_sources src/st_adaptor.c)
endif()

add_library(zookeeper STATIC ${zookeeper_sources})
target_link_libraries(zookeeper PUBLIC hashtable m rt)
target_link_libraries(zookeeper PRIVATE -Wno-undefined)
if(WANT_SYNCAPI)
  target_link_libraries(zookeeper PUBLIC pthread)
endif()
# TODO: Specify symbols to export.

# cli executable
add_executable(cli src/cli.c)
target_link_libraries(cli zookeeper)

# load_gen executable
if(WANT_SYNCAPI)
  add_executable(load_gen src/load_gen.c)
  target_compile_options(load_gen PRIVATE -Wno-unused-variable)
  target_link_libraries(load_gen zookeeper)
endif()

# tests
set(test_sources
  tests/TestDriver.cc
  tests/LibCMocks.cc
  tests/LibCSymTable.cc
  tests/MocksBase.cc
  tests/ZKMocks.cc
  tests/Util.cc
  tests/ThreadingUtil.cc
  tests/TestZookeeperInit.cc
  tests/TestZookeeperClose.cc
  tests/TestReconfig.cc
  tests/TestReconfigServer.cc
  tests/TestClientRetry.cc
  tests/TestOperations.cc
  tests/TestMulti.cc
  tests/TestWatchers.cc
  tests/TestClient.cc
  tests/ZooKeeperQuorumServer.cc
  tests/TestReadOnlyClient.cc)

if(WANT_SYNCAPI)
  list(APPEND test_sources tests/PthreadMocks.cc)
endif()

if(WANT_CPPUNIT)
  add_executable(zktest ${test_sources})
  target_compile_options(zktest PRIVATE
    -Wno-sign-compare
    -Wno-unused-variable
    -Wno-unused-but-set-variable
    -Wno-uninitialized
    -Wno-reorder)
  target_compile_definitions(zktest
    PRIVATE -DUSE_STATIC_LIB -DZKSERVER_CMD="${CMAKE_SOURCE_DIR}/tests/zkServer.sh")
  target_link_libraries(zktest zookeeper cppunit dl)

  file(STRINGS tests/wrappers.opt symbol_wrappers)
  if(WANT_SYNCAPI)
    file(STRINGS tests/wrappers-mt.opt symbol_wrappers_mt)
  endif()

  target_link_libraries(zktest ${symbol_wrappers} ${symbol_wrappers_mt})

  enable_testing()
  add_test(NAME zktest_runner COMMAND zktest)
  set_property(TEST zktest_runner PROPERTY ENVIRONMENT
    "ZKROOT=${CMAKE_SOURCE_DIR}/../.."
    "CLASSPATH=$CLASSPATH:$CLOVER_HOME/lib/clover.jar")
endif()
