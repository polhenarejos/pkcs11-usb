cmake_minimum_required (VERSION 2.4)

if(COMMAND cmake_policy)
      cmake_policy(SET CMP0003 NEW)
endif(COMMAND cmake_policy)

project (pkcs11 C)

set(PKCS11_USB_SRCS gck-rpc-module.c gck-rpc-message.c gck-rpc-util.c egg-buffer.c gck-rpc-tls-psk.c)

add_definitions(-Wall)
add_library(pkcs11-usb SHARED ${PKCS11_USB_SRCS})

# Disable console when building Win32 binary in release mode
if (WIN32)
  if("${CMAKE_BUILD_TYPE}" MATCHES "^Rel.*")
    set(GUI_TYPE WIN32)
  endif()
endif()

find_package(PkgConfig)
pkg_check_modules(PC_LIBPCSC libpcsclite)
set(PCSC_DEFINITIONS ${PC_LIBPCSC_CFLAGS_OTHER})

find_path(LIBPCSC_INCLUDE_DIR winscard.h
          HINTS ${PC_LIBPCSC_INCLUDEDIR} ${PC_LIBPCSC_INCLUDE_DIRS}
          PATH_SUFFIXES PCSC )
          
find_library(LIBPCSC_LIBRARY NAMES pcsclite
             HINTS ${PC_LIBPCSC_LIBDIR} ${PC_LIBPCSC_LIBRARY_DIRS} )

include(FindPackageHandleStandardArgs)
# handle the QUIETLY and REQUIRED arguments and set LIBPCSC_FOUND to TRUE
# if all listed variables are TRUE
find_package_handle_standard_args(LIBPCSC  DEFAULT_MSG
                                  LIBPCSC_LIBRARY LIBPCSC_INCLUDE_DIR)
mark_as_advanced(LIBPCSC_INCLUDE_DIR LIBPCSC_LIBRARY )

set(LIBPCSC_LIBRARIES ${LIBPCSC_LIBRARY} )
set(LIBPCSC_INCLUDE_DIRS ${LIBPCSC_INCLUDE_DIR} )

if (LIBPCSC_FOUND)
  include_directories(${LIBPCSC_INCLUDE_DIRS})
  message(STATUS "PCSC include dir='${LIBPCSC_INCLUDE_DIRS}'")
  target_link_libraries (pkcs11-usb ${LIBPCSC_LIBRARIES})
endif (LIBPCSC_FOUND)

set_target_properties(pkcs11-usb PROPERTIES VERSION 0.1 SOVERSION 0)

if (WIN32)
  include_directories("ext/include")

  add_library(dl STATIC IMPORTED)
  set_property(TARGET dl PROPERTY
               IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/ext/lib/libdl.a)
  target_link_libraries (pkcs11-usb ws2_32)
endif (WIN32)

target_link_libraries (pkcs11-usb pthread ssl crypto)

install_targets (/lib pkcs11-usb)

add_custom_command(
   OUTPUT syscall-names.h 
   COMMAND ${CMAKE_SOURCE_DIR}/mksyscalls.sh
   WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
