cmake_minimum_required(VERSION 3.31)
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/.devcontainer/cmake)
include(preproject)
project(node VERSION 14.17.6)
set(webpro_REV NONE)
include(flags)
include(GNUInstallDirs)
set(XP_INSTALL_CMAKEDIR ${CMAKE_INSTALL_DATADIR}/cmake) # xpPackageDevel
include(ExternalProject)
set_property(DIRECTORY PROPERTY "EP_BASE" ${CMAKE_BINARY_DIR}/epbase)
find_package(Python)
if(NOT Python_FOUND)
  message(FATAL_ERROR "Unable to build node, required Python not found")
endif()
if(TRUE)#WIN32)
  xpFindPkg(PKGS nasm)
endif()
if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm")
  if(CMAKE_SIZEOF_VOID_P EQUAL 8) # 64-bit
    set(destcpu arm64)
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 4) # 32-bit
    set(destcpu arm)
  else()
    message(FATAL_ERROR "destcpu: unexpected")
  endif()
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64")
  set(destcpu arm64)
else()
  if(CMAKE_SIZEOF_VOID_P EQUAL 8) # 64-bit
    set(destcpu x64)
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 4) # 32-bit
    set(destcpu ia32)
  else()
    message(FATAL_ERROR "destcpu: unexpected")
  endif()
endif()
if(MSVC)
  set(XP_CONFIGURE vcbuild release ${destcpu})
elseif(UNIX)
  if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    set(destos linux)
  elseif(${CMAKE_SYSTEM_NAME} STREQUAL "Darwin")
    set(destos mac)
  else()
    message(FATAL_ERROR "destos: unexpected")
  endif()
  list(APPEND XP_CONFIGURE ${CMAKE_SOURCE_DIR}/configure
    --prefix=<INSTALL_DIR>
    --without-etw --without-dtrace
    # TODO: explore --tag=TAG custom build tag
    # TODO: explore --fully-static
    --dest-cpu=${destcpu} --dest-os=${destos}
    )
else()
  message(FATAL_ERROR "unsupported OS platform")
endif()
if(XP_BUILD_VERBOSE)
  xpVerboseListing("[CONFIGURE]" "${XP_CONFIGURE}")
endif()
set(cfg Release)
set(nodejs_DEPS)
set(XP_TARGET nodejs)
if(MSVC)
  # TRICKY: vcbuild doesn't seem to return a good exit status or something...
  #   MSVC thinks it always needs to build this (after what appears to be a
  #   successful build) -- and there can't be any external project steps
  #   after the step with vcbuild or they won't execute
  ExternalProject_Add(${XP_TARGET}vcbuild
    DOWNLOAD_COMMAND "" DOWNLOAD_DIR ${NULL_DIR}
    SOURCE_DIR ${CMAKE_SOURCE_DIR} INSTALL_DIR ${NULL_DIR}
    CONFIGURE_COMMAND ${XP_CONFIGURE}
    BUILD_IN_SOURCE 1 # <BINARY_DIR>==<SOURCE_DIR>
    )
  set_property(TARGET ${XP_TARGET}vcbuild PROPERTY FOLDER ${bld_folder})
  set(nodejs_DEPS ${XP_TARGET}vcbuild) # serialize the build
  set(binNode ${CMAKE_SOURCE_DIR}/${cfg}/node.exe)
  set(libNode ${CMAKE_SOURCE_DIR}/${cfg}/node.lib)
  set(XP_CONFIGURE ${CMAKE_COMMAND} -E echo "Configure MSVC...")
  set(XP_BUILD_CMD ${CMAKE_COMMAND} -E make_directory ${STAGE_DIR}/lib)
  set(XP_INSTALL_CMD ${CMAKE_COMMAND} -E copy ${libNode} ${STAGE_DIR}/lib)
elseif(UNIX)
  set(binNode ${CMAKE_SOURCE_DIR}/out/${cfg}/node)
  set(XP_BUILD_CMD)   # use default
  set(XP_INSTALL_CMD) # use default
endif()
ExternalProject_Add(${XP_TARGET} DEPENDS ${nodejs_DEPS}
  DOWNLOAD_COMMAND "" DOWNLOAD_DIR ${NULL_DIR}
  SOURCE_DIR ${CMAKE_SOURCE_DIR} INSTALL_DIR ${NULL_DIR}
  CONFIGURE_COMMAND ${XP_CONFIGURE}
  BUILD_COMMAND ${XP_BUILD_CMD}
  BUILD_IN_SOURCE 1 # <BINARY_DIR>==<SOURCE_DIR>
  INSTALL_COMMAND ${XP_INSTALL_CMD}
  )
ExternalProject_Add_Step(${XP_TARGET} copy_bin
  COMMAND ${CMAKE_COMMAND} -E make_directory ${STAGE_DIR}/bin
  COMMAND ${CMAKE_COMMAND} -E copy ${binNode} ${STAGE_DIR}/bin
  DEPENDEES install
  )
set_property(TARGET ${XP_TARGET} PROPERTY FOLDER ${bld_folder})
list(APPEND nodejs_DEPS ${XP_TARGET}) # serialize the build

function(copyheaders)
# copy headers to STAGE_DIR
set(nodejsHdrs ${CMAKE_SOURCE_DIR}/src/*.h)
set(uvDir ${CMAKE_SOURCE_DIR}/deps/uv/include)
set(v8Hdrs ${CMAKE_SOURCE_DIR}/deps/v8/include/*.h)
set(cppgcHdrs ${CMAKE_SOURCE_DIR}/deps/v8/include/cppgc/*.h)
set(XP_TARGET nodejs_stage)
if(NOT TARGET ${XP_TARGET})
  ExternalProject_Add(${XP_TARGET} DEPENDS ${nodejs_DEPS}
    DOWNLOAD_DIR ${NULL_DIR} BINARY_DIR ${NULL_DIR}
    SOURCE_DIR ${NULL_DIR} INSTALL_DIR ${STAGE_DIR}/include/node_${VER}/node
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory ${uvDir} <INSTALL_DIR>
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -Dsrc:STRING=${nodejsHdrs}
      -Ddst:STRING=<INSTALL_DIR> -P ${MODULES_DIR}/cmscopyfiles.cmake
    BUILD_COMMAND ${CMAKE_COMMAND} -Dsrc:STRING=${v8Hdrs}
      -Ddst:STRING=<INSTALL_DIR> -P ${MODULES_DIR}/cmscopyfiles.cmake
    INSTALL_COMMAND ${CMAKE_COMMAND} -Dsrc:STRING=${cppgcHdrs}
      -Ddst:STRING=<INSTALL_DIR>/cppgc -P ${MODULES_DIR}/cmscopyfiles.cmake
    )
  set_property(TARGET ${XP_TARGET} PROPERTY FOLDER ${bld_folder})
endif()
endfunction()

function(nasm)
if(WIN32 AND NOT UNIX)
  set(VER ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH})
  set(OS win64)
  ExternalProject_Add(${PROJECT_NAME}
    DOWNLOAD_EXTRACT_TIMESTAMP true
    URL https://www.nasm.us/pub/${PROJECT_NAME}/releasebuilds/${VER}/${OS}/${PROJECT_NAME}-${VER}-${OS}.zip
    URL_HASH SHA256=18918ac906e29417b936466e7a2517068206c8db8c04b9762a5befa18bfea5f0
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E copy_if_different
      <SOURCE_DIR>/nasm.exe <INSTALL_DIR>/bin/nasm.exe
    BUILD_COMMAND ${CMAKE_COMMAND} -E copy_if_different
      <SOURCE_DIR>/LICENSE <INSTALL_DIR>/share/LICENSE
    INSTALL_COMMAND ""
    )
  xpPackageDevel(EXE_PATH bin/nasm)
  ExternalProject_Get_Property(${PROJECT_NAME} INSTALL_DIR)
  install(DIRECTORY ${INSTALL_DIR}/ DESTINATION . USE_SOURCE_PERMISSIONS)
else()
  xpPackageDevel()
endif()
endfunction()
function(build_nodejs)
  set(NAME node)
  xpGetArgValue(${PRO_NODEJS} ARG VER VALUE VER)
  set(FIND_DEPS "set(nodeVer ${VER}) # for xpnode.cmake\n")
  set(TARGETS_FILE xpnode.cmake)
  string(TOUPPER ${NAME} PRJ)
  set(USE_VARS "set(${PRJ}_LIBRARIES xpro::${NAME})\n")
  set(USE_VARS "${USE_VARS}set(${PRJ}_EXE \${XP_ROOTDIR}/bin/${NAME}\${CMAKE_EXECUTABLE_SUFFIX})\n")
  set(USE_VARS "${USE_VARS}list(APPEND reqVars ${PRJ}_LIBRARIES ${PRJ}_EXE)\n")
  configure_file(${MODULES_DIR}/usexp.cmake.in
    ${STAGE_DIR}/share/cmake/usexp-${NAME}-config.cmake
    @ONLY NEWLINE_STYLE LF
    )
  configure_file(${MODULES_DIR}/xpnode.cmake ${STAGE_DIR}/share/cmake/ COPYONLY)
endfunction()
