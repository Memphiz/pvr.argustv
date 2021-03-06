PROJECT(pvr.argustv)

CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

include(AddonVersion.cmake)
addon_version(pvr.argustv ARGUSTV)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR})

ENABLE_LANGUAGE(CXX)

OPTION(PACKAGE_ZIP "Package Zip file?" OFF)

IF(WIN32)
  SET(BINDING_FILE ${XBMC_BINDINGS}.zip)
  message (STATUS "downloading XBMC bindings: " ${BINDING_FILE})
  file(DOWNLOAD http://mirrors.xbmc.org/build-deps/win32/${BINDING_FILE} ${CMAKE_BINARY_DIR}/downloads/${BINDING_FILE} STATUS STATUSLIST SHOW_PROGRESS)
  LIST(GET STATUSLIST 0 VALUE)
  IF(${VALUE} STRGREATER "0")
    LIST(GET STATUSLIST 1 VALUE)
    message (STATUS "failed to download XBMC bindings: " ${VALUE})
  ENDIF(${VALUE} STRGREATER "0")
  execute_process(
   COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_BINARY_DIR}/downloads/${BINDING_FILE}
  )
  FIND_PATH(XBMC_INCLUDE_DIR xbmc/xbmc_pvr_dll.h ${CMAKE_BINARY_DIR}/include)
  INCLUDE_DIRECTORIES(${XBMC_INCLUDE_DIR})
  ADD_DEFINITIONS(-DTARGET_WINDOWS)
ELSE(WIN32)
  FIND_PATH(XBMC_INCLUDE_DIR xbmc/xbmc_pvr_dll.h)
  FIND_PACKAGE(xbmcplatform REQUIRED)
  SET(INCLUDES ${xbmcplatform_INCLUDE_DIRS}
               ${XBMC_INCLUDE_DIR}
               ${PROJECT_SOURCE_DIR}/src)
  ADD_DEFINITIONS(-DTARGET_DARWIN -DTARGET_DARWIN_OSX)
ENDIF(WIN32)

SET(DEPLIBS ${xbmcplatform_LIBRARIES} tsreader)

FIND_PACKAGE(JsonCpp)
IF(JSONCPP_FOUND)
  LIST(APPEND DEPLIBS ${JSONCPP_LIBRARY})
  LIST(APPEND INCLUDES ${JSONCPP_INCLUDE_DIR})
ELSE(JSONCPP_FOUND)
  MESSAGE(STATUS "Building JsonCpp")
  INCLUDE(ExternalProject)
  ExternalProject_ADD(JsonCpp
                      URL http://garr.dl.sourceforge.net/project/jsoncpp/jsoncpp/0.5.0/jsoncpp-src-0.5.0.tar.gz
                      PREFIX JsonCpp
                      PATCH_COMMAND patch -p0 < ${PROJECT_SOURCE_DIR}/01-jsoncpp-add-cmake-buildsystem.patch
                      CMAKE_ARGS -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE} -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>)
  LIST(APPEND INCLUDES ${CMAKE_BINARY_DIR}/JsonCpp/include/jsoncpp)
  LIST(APPEND DEPLIBS ${CMAKE_BINARY_DIR}/JsonCpp/lib/libjsoncpp.a)
ENDIF(JSONCPP_FOUND)

INCLUDE_DIRECTORIES(${INCLUDES})

SET(SOURCES src/activerecording.cpp
            src/argustvrpc.cpp
            src/channel.cpp
            src/client.cpp
            src/epg.cpp
            src/guideprogram.cpp
            src/KeepAliveThread.cpp
            src/pvrclient-argustv.cpp
            src/recording.cpp
            src/recordinggroup.cpp
            src/recordingsummary.cpp
            src/tools.cpp
            src/upcomingrecording.cpp
            src/uri.cpp
            src/utils.cpp)

ADD_SUBDIRECTORY(src/lib/tsreader)

ADD_LIBRARY(pvr.argustv ${SOURCES})
ADD_DEPENDENCIES(pvr.argustv tsreader)
IF(NOT WIN32)
  TARGET_LINK_LIBRARIES(pvr.argustv ${DEPLIBS})
  set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "-s")
  set(CMAKE_MODULE_LINKER_FLAGS_RELEASE "-s")
ENDIF(NOT WIN32)
SET_TARGET_PROPERTIES(pvr.argustv PROPERTIES VERSION ${ARGUSTV_VERSION}
                                          SOVERSION 13.0
                                          PREFIX "")

IF(PACKAGE_ZIP)
  # Pack files together to create an archive
  INSTALL(DIRECTORY ${PROJECT_NAME} DESTINATION ./)
  IF(WIN32)
    INSTALL(PROGRAMS ${CMAKE_BINARY_DIR}/pvr.argustv.dll DESTINATION pvr.argustv)
  ELSE(WIN32)
    INSTALL(TARGETS pvr.argustv DESTINATION pvr.argustv)
  ENDIF(WIN32)
  SET(CPACK_GENERATOR "ZIP")
  SET(CPACK_INCLUDE_TOPLEVEL_DIRECTORY OFF)
  SET(CPACK_PACKAGE_FILE_NAME ${CMAKE_PROJECT_NAME}-${ARGUSTV_VERSION})
  INCLUDE(CPack)
ELSE(PACKAGE_ZIP)
  INSTALL(TARGETS pvr.argustv DESTINATION lib/xbmc/addons/pvr.argustv)
  INSTALL(DIRECTORY pvr.argustv DESTINATION share/xbmc/addons)
ENDIF(PACKAGE_ZIP)
