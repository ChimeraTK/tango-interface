cmake_minimum_required(VERSION 3.16)

project(tango-interface)

set(${PROJECT_NAME}_MAJOR_VERSION 01)
set(${PROJECT_NAME}_MINOR_VERSION 00)
set(${PROJECT_NAME}_PATCH_VERSION 00)

include(${CMAKE_SOURCE_DIR}/cmake/set_version_numbers.cmake)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Modules)
include(${CMAKE_SOURCE_DIR}/cmake/set_default_flags.cmake)
include(${CMAKE_SOURCE_DIR}/cmake/set_default_build_to_release.cmake)

find_program(LSB_RELEASE lsb_release)

if (LSB_RELEASE)
  execute_process(COMMAND ${LSB_RELEASE} "-c" OUTPUT_VARIABLE CODENAME_RAW)
  string(REPLACE ":" ";" CODENAME_RAW ${CODENAME_RAW})
  list(GET CODENAME_RAW 1 CODENAME)
  string(STRIP ${CODENAME} CODENAME)
  if(CODENAME STREQUAL "n/a")
    # Hello, are you Tumbleweed
    execute_process(COMMAND ${LSB_RELEASE} "-d" OUTPUT_VARIABLE LSB_DESCRIPTION)
    if (LSB_DESCRIPTION MATCHES "umbleweed")
      set(CODENAME "tumbleweed")
    endif()
  endif()
else()
  set(CODENAME "Unknown")
endif()

message("Distribution codename is ${CODENAME}")
set(TANGO_GENERATOR Ninja)
if (CODENAME STREQUAL "bookworm")
  set(IDL_VERSION "5.1.0")
  set(ORB_VERSION "4.3.0")
  set(ORB_HASH "976045a2341f4e9a85068b21f4bd928993292933eeecefea372db09e0219eadd")
  set(TANGO_VERSION "9.3.7")
  set(TANGO_GENERATOR "Unix Makefiles")
else ()
  set(IDL_VERSION "6.0.2")
  set(ORB_VERSION "4.3.2")
  set(ORB_HASH "1c745330d01904afd7a1ed0a5896b9a6e53ac1a4b864a48503b93c7eecbf1fa8")
  set(TANGO_VERSION "10.1.0")
endif()

find_program(PATCHELF patchelf)

if(PATCHELF)
  execute_process(COMMAND ${PATCHELF} "--version" OUTPUT_VARIABLE PATCHELF_OUTPUT)
  string(REPLACE " " ";" PATCHELF_VERSION ${PATCHELF_OUTPUT})
  list(POP_FRONT PATCHELF_VERSION)

  if(PATCHELF_VERSION VERSION_LESS "0.18.0")
    set(PATCHELF "PATCHELF-NOTFOUND")
  else()
    message("${PATCHELF} is >= 0.18, using system version")
  endif()
endif()

if(NOT PATCHELF)
  include(ExternalProject)
  ExternalProject_Add(patchelf-external
    SOURCE_DIR ${PROJECT_BINARY_DIR}/patchelf
    URL https://github.com/NixOS/patchelf/releases/download/0.18.0/patchelf-0.18.0.tar.gz
    BUILD_BYPRODUCTS patchelf-external-prefix/bin/patchelf
    PATCH_COMMAND patch -p1 < ${CMAKE_CURRENT_SOURCE_DIR}/drop-unsupported-elf-tag.patch
    CONFIGURE_COMMAND ${PROJECT_BINARY_DIR}/patchelf/configure --prefix=<INSTALL_DIR> CFLAGS=-w CPPFLAGS=-w CXXFLAGS=-w
    BUILD_COMMAND ${MAKE}
  )
  ExternalProject_Get_property(patchelf-external install_dir)
  set(PATCHELF ${install_dir}/bin/patchelf)
  add_custom_target(patchelf-build DEPENDS patchelf-external)
else()
  add_custom_target(patchelf-build COMMAND echo "Nothing to do for patchelf.")
endif()

message("Patchelf is ${PATCHELF}")

message("Selected versions for ${CODENAME}")
message("IDL: ${IDL_VERSION}, ORB: ${ORB_VERSION} (with sha256 ${ORB_HASH}), tango: ${TANGO_VERSION}")

set(INTERMEDIATE_INSTALL_FOLDER ${CMAKE_BINARY_DIR}/temp_install)

include(ExternalProject)
ExternalProject_add(tango_idl_external
    GIT_REPOSITORY "https://gitlab.com/tango-controls/tango-idl.git"
    GIT_TAG "${IDL_VERSION}"
    CMAKE_GENERATOR Ninja
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} -DCMAKE_INSTALL_LIBDIR=lib
)

ExternalProject_add(omniORB_external
    URL "https://sourceforge.net/projects/omniorb/files/omniORB/omniORB-${ORB_VERSION}/omniORB-${ORB_VERSION}.tar.bz2"
    URL_HASH SHA256=${ORB_HASH}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_INSTALL_PREFIX} --libdir=${CMAKE_INSTALL_PREFIX}/lib
)

set(ENV{PYTHONPATH} "${INTERMEDIATE_INSTALL_FOLDER}${CMAKE_INSTALL_PREFIX}/python2.7/dist-packages:$ENV{PYTHONPOATH}:${tango_idl_path}/lib/pkgconfig")

# We drop cpp_test_suite for old tango since it depends on python being python2
ExternalProject_add(cpptango_external
    GIT_REPOSITORY "https://gitlab.com/tango-controls/cppTango.git"
    GIT_TAG "${TANGO_VERSION}"
    DEPENDS tango_idl_external omniORB_external
    PATCH_COMMAND "sed" "--in-place" "-e" "/add_subdirectory(\"cpp_test_suite\")'/d" <SOURCE_DIR>/CMakeLists.txt
    CMAKE_GENERATOR ${TANGO_GENERATOR}
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
        -DCMAKE_INSTALL_LIBDIR=lib
        -DOMNI_BASE=${CMAKE_INSTALL_PREFIX}
        -DBUILD_TESTING=Off
        -DTANGO_USE_TELEMETRY=Off
)

configure_file(patch_rpath.in patch_rpath @ONLY)

install(CODE [===[
    execute_process(COMMAND ${CMAKE_BINARY_DIR}/patch_rpath --set-rpath ${CMAKE_INSTALL_PREFIX}/lib ${DESTDIR}/${CMAKE_INSTALL_PREFIX}/lib/libtango.so)
]===])
# Fake install target
install(FILES README.md DESTINATION ${CMAKE_INSTALL_PREFIX}/share/doc/tango-interface)
