cmake_minimum_required(VERSION 2.6)

add_subdirectory (lua)
get_directory_property (CMAKE_LUA_INCLUDES DIRECTORY lua
    DEFINITION LUA_INCLUDE_DIRS)
get_directory_property (CMAKE_LUA_LIBRARIES DIRECTORY lua
    DEFINITION LUA_LIBRARIES)

project(domoticz)

SET(EXECUTABLE_OUTPUT_PATH "" CACHE INTERNAL 
  "Where to put the executables for Domoticz"
  )

#set(CMAKE_EXE_LINKER_FLAGS "-static")

# Macro for setting up precompiled headers. Usage:
#
#   add_precompiled_header(target header.h [FORCEINCLUDE])
#
# MSVC: A source file with the same name as the header must exist and
# be included in the target (E.g. header.cpp).
#
# MSVC: Add FORCEINCLUDE to automatically include the precompiled
# header file from every source file.
#
# GCC: The precompiled header is always automatically included from
# every header file.
MACRO(ADD_PRECOMPILED_HEADER _targetName _input)
  GET_FILENAME_COMPONENT(_inputWe ${_input} NAME_WE)
  SET(pch_source ${_inputWe}.cpp)
  FOREACH(arg ${ARGN})
    IF(arg STREQUAL FORCEINCLUDE)
      SET(FORCEINCLUDE ON)
    ELSE(arg STREQUAL FORCEINCLUDE)
      SET(FORCEINCLUDE OFF)
    ENDIF(arg STREQUAL FORCEINCLUDE)
  ENDFOREACH(arg)

  IF(MSVC)
    GET_TARGET_PROPERTY(sources ${_targetName} SOURCES)
    SET(_sourceFound FALSE)
    FOREACH(_source ${sources})
      SET(PCH_COMPILE_FLAGS "")
      IF(_source MATCHES \\.\(cc|cxx|cpp\)$)
	GET_FILENAME_COMPONENT(_sourceWe ${_source} NAME_WE)
	IF(_sourceWe STREQUAL ${_inputWe})
	  SET(PCH_COMPILE_FLAGS "${PCH_COMPILE_FLAGS} /Yc${_input}")
	  SET(_sourceFound TRUE)
	ELSE(_sourceWe STREQUAL ${_inputWe})
	  SET(PCH_COMPILE_FLAGS "${PCH_COMPILE_FLAGS} /Yu${_input}")
	  IF(FORCEINCLUDE)
	    SET(PCH_COMPILE_FLAGS "${PCH_COMPILE_FLAGS} /FI${_input}")
	  ENDIF(FORCEINCLUDE)
	ENDIF(_sourceWe STREQUAL ${_inputWe})
	SET_SOURCE_FILES_PROPERTIES(${_source} PROPERTIES COMPILE_FLAGS "${PCH_COMPILE_FLAGS}")
      ENDIF(_source MATCHES \\.\(cc|cxx|cpp\)$)
    ENDFOREACH()
    IF(NOT _sourceFound)
      MESSAGE(FATAL_ERROR "A source file for ${_input} was not found. Required for MSVC builds.")
    ENDIF(NOT _sourceFound)
  ENDIF(MSVC)

  IF(CMAKE_COMPILER_IS_GNUCXX)
	#set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-switch")
	#ADD_DEFINITIONS( -Wall -O0 -ggdb )
	#ADD_DEFINITIONS( -Wfatal-errors -Wformat=2 -Werror=format-security )

	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libgcc -static-libstdc++")
	SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -static-libgcc -static-libstdc++")
	SET(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "${CMAKE_SHARED_LIBRARY_LINK_C_FLAGS} -static-libgcc -static-libstdc++")
	SET(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "${CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS} -static-libgcc -static-libstdc++")

    GET_FILENAME_COMPONENT(_name ${_input} NAME)
    SET(_source "${CMAKE_CURRENT_SOURCE_DIR}/${_input}")
    SET(_outdir "${CMAKE_CURRENT_BINARY_DIR}/${_name}.gch")
    MAKE_DIRECTORY(${_outdir})
    SET(_output "${_outdir}/.c++")

    STRING(TOUPPER "CMAKE_CXX_FLAGS_${CMAKE_BUILD_TYPE}" _flags_var_name)
    SET(_compiler_FLAGS ${${_flags_var_name}})

    GET_DIRECTORY_PROPERTY(_directory_flags INCLUDE_DIRECTORIES)
    FOREACH(item ${_directory_flags})
      LIST(APPEND _compiler_FLAGS "-I${item}")
    ENDFOREACH(item)

    GET_DIRECTORY_PROPERTY(_directory_flags DEFINITIONS)
    LIST(APPEND _compiler_FLAGS ${_directory_flags})

    SEPARATE_ARGUMENTS(_compiler_FLAGS)
    MESSAGE("${CMAKE_CXX_COMPILER} -DPCHCOMPILE ${_compiler_FLAGS} -x c++-header -o {_output} ${_source}")
    ADD_CUSTOM_COMMAND(
      OUTPUT ${_output}
      COMMAND ${CMAKE_CXX_COMPILER} ${_compiler_FLAGS} -x c++-header -o ${_output} ${_source}
      DEPENDS ${_source} )
    ADD_CUSTOM_TARGET(${_targetName}_gch DEPENDS ${_output})
    ADD_DEPENDENCIES(${_targetName} ${_targetName}_gch)
    SET_TARGET_PROPERTIES(${_targetName} PROPERTIES COMPILE_FLAGS "-include ${_name} -Winvalid-pch")
  ENDIF(CMAKE_COMPILER_IS_GNUCXX)
ENDMACRO()

find_path(SQLITE3_INCLUDE_DIR sqlite3.h)
find_library(SQLITE3_LIBRARY sqlite3)
if(NOT SQLITE3_INCLUDE_DIR)
  message(SEND_ERROR "Failed to find sqlite3.h")
endif()
if(NOT SQLITE3_LIBRARY)
  message(SEND_ERROR "Failed to find the sqlite3 library")
endif()

FIND_PROGRAM(SVN_EXECUTABLE svn
  DOC "subversion command line client")

MACRO(Subversion_GET_REVISION dir variable)
  EXECUTE_PROCESS(COMMAND ${SVN_EXECUTABLE} info ${dir}
    OUTPUT_VARIABLE ${variable}
    OUTPUT_STRIP_TRAILING_WHITESPACE)
  STRING(REGEX REPLACE "^(.*\n)?Revision: ([^\n]+).*"
    "\\2" ${variable} "${${variable}}")
ENDMACRO(Subversion_GET_REVISION)

Subversion_GET_REVISION(. ProjectRevision)
MESSAGE(STATUS "Compiling Revision #${ProjectRevision}")

# The version number.
set (domoticz_VERSION_MAJOR 1)
set (domoticz_VERSION_MINOR 0)
set (domoticz_VERSION_PATCH ${ProjectRevision})

include_directories(${CMAKE_SOURCE_DIR}/main) 

# a custom target that is always built
ADD_CUSTOM_TARGET(revisiontag ALL)

# creates svnversion.h using cmake script
ADD_CUSTOM_COMMAND(TARGET revisiontag COMMAND ${CMAKE_COMMAND}
   -DSOURCE_DIR=${CMAKE_CURRENT_SOURCE_DIR} 
   -P ${CMAKE_CURRENT_SOURCE_DIR}/getsvn.cmake)

## Target
set(
domoticz_SRCS
main/stdafx.cpp
main/CmdLine.cpp
main/Camera.cpp
main/domoticz.cpp
main/EventSystem.cpp
main/Helper.cpp
main/localtime_r.cpp
main/Logger.cpp
main/mainworker.cpp
main/RFXNames.cpp
main/Scheduler.cpp
main/SQLHelper.cpp
main/SunRiseSet.cpp
main/WebServer.cpp
main/WindCalculation.cpp
main/DataPush.cpp
httpclient/HTTPClient.cpp
httpclient/UrlEncode.cpp
hardware/1Wire.cpp
hardware/1Wire/1WireByOWFS.cpp
hardware/1Wire/1WireByKernel.cpp
hardware/1Wire/1WireCommon.cpp
hardware/1Wire/1WireForWindows.cpp
hardware/ASyncSerial.cpp
hardware/ASyncTCP.cpp
hardware/BMP085.cpp
hardware/csocket.cpp
hardware/DavisLoggerSerial.cpp
hardware/DomoticzHardware.cpp
hardware/DomoticzTCP.cpp
hardware/Dummy.cpp
hardware/EcoDevices.cpp
hardware/EnOceanESP2.cpp
hardware/EnOceanESP3.cpp
hardware/evohome.cpp
hardware/ForecastIO.cpp
hardware/Gpio.cpp
hardware/GpioPin.cpp
hardware/HardwareMonitor.cpp
hardware/HarmonyHub.cpp
hardware/ICYThermostat.cpp
hardware/Limitless.cpp
hardware/Meteostick.cpp
hardware/MochadTCP.cpp
hardware/OpenZWave.cpp
hardware/openzwave/control_panel/ozwcp.cpp
hardware/openzwave/control_panel/zwavelib.cpp
hardware/openzwave/control_panel/tinyxml/tinystr.cpp
hardware/openzwave/control_panel/tinyxml/tinyxml.cpp
hardware/openzwave/control_panel/tinyxml/tinyxmlerror.cpp
hardware/openzwave/control_panel/tinyxml/tinyxmlparser.cpp
hardware/OTGWBase.cpp
hardware/OTGWSerial.cpp
hardware/OTGWTCP.cpp
hardware/P1MeterBase.cpp
hardware/P1MeterSerial.cpp
hardware/P1MeterTCP.cpp
hardware/PhilipsHue.cpp
hardware/PiFace.cpp
hardware/PVOutput_Input.cpp
hardware/Razberry.cpp
hardware/Rego6XXSerial.cpp
hardware/RFXComSerial.cpp
hardware/RFXComTCP.cpp
hardware/S0MeterBase.cpp
hardware/S0MeterSerial.cpp
hardware/SBFSpot.cpp
hardware/smbus.cpp
hardware/SolarEdgeBase.cpp
hardware/SolarEdgeTCP.cpp
hardware/TCPProxy/tcpproxy_server.cpp
hardware/TE923.cpp
hardware/TE923Tool.cpp
hardware/Teleinfo.cpp
hardware/ToonThermostat.cpp
hardware/VolcraftCO20.cpp
hardware/VolcraftCO20Tool.cpp
hardware/WOL.cpp
hardware/Wunderground.cpp
hardware/YouLess.cpp
hardware/ZWaveBase.cpp
smtpclient/SMTPClient.cpp
tcpserver/TCPClient.cpp
tcpserver/TCPServer.cpp
webserver/Base64.cpp
webserver/connection.cpp
webserver/connection_manager.cpp
webserver/cWebem.cpp
webserver/mime_types.cpp
webserver/reply.cpp
webserver/request_handler.cpp
webserver/request_parser.cpp
webserver/server.cpp
json/json_reader.cpp
json/json_value.cpp
json/json_writer.cpp
)
add_executable(domoticz ${domoticz_SRCS})

# explicitly say that the executable depends on the revisiontag
add_dependencies(domoticz revisiontag)


## Link libraries

option(USE_STATIC_BOOST "Build with static BOOST libraries" YES)
set(Boost_USE_STATIC_LIBS ${USE_STATIC_BOOST})
set(Boost_USE_MULTITHREADED ON)
unset(Boost_INCLUDE_DIR CACHE)
unset(Boost_LIBRARY_DIRS CACHE)
find_package(Boost REQUIRED COMPONENTS thread date_time system)
if(USE_STATIC_BOOST)
   message(STATUS "Linking against boost static libraries")
else(USE_STATIC_BOOST)
   message(STATUS "Linking against boost dynamic libraries")
endif(USE_STATIC_BOOST)

FIND_PACKAGE(ZLIB)
IF(ZLIB_FOUND)
  MESSAGE(STATUS "ZLIB libraries found at: ${ZLIB_LIBRARIES}")
  MESSAGE(STATUS "ZLIB includes found at: ${ZLIB_INCLUDE_DIRS}")
  INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIRS})
else()
  MESSAGE(FATAL_ERROR "ZLIB not found on your system, see install.txt how to get them installed. (for example 'sudo apt-get install zlib1g-dev')")
ENDIF(ZLIB_FOUND)

FIND_PACKAGE(CURL)
IF(CURL_FOUND)
  MESSAGE(STATUS "Curl libraries found at: ${CURL_LIBRARIES}")
  MESSAGE(STATUS "Curl includes found at: ${CURL_INCLUDE_DIRS}")
  INCLUDE_DIRECTORIES(${CURL_INCLUDE_DIRS})
else()
  MESSAGE(FATAL_ERROR "cURL not found on your system, see install.txt how to get them installed. (for example 'sudo apt-get install curl libcurl4-openssl-dev')")
ENDIF(CURL_FOUND)

find_path(LIBUSB_INCLUDE_DIR usb.h
   HINTS ${PC_LIBUSB_INCLUDEDIR} ${PC_LIBUSB_INCLUDE_DIRS})
find_library(LIBUSB_LIBRARY NAMES usb
   HINTS ${PC_LIBUSB_LIBDIR} ${PC_LIBUSB_LIBRARY_DIRS})
set(LIBUSB_LIBRARIES ${LIBUSB_LIBRARY})

find_package_handle_standard_args(LIBUSB  DEFAULT_MSG  LIBUSB_LIBRARIES LIBUSB_INCLUDE_DIR)
IF(LIBUSB_FOUND)
  MESSAGE(STATUS "LIBUSB found at: ${LIBUSB_LIBRARIES}")
  add_definitions(-DWITH_LIBUSB)
  target_link_libraries(domoticz ${LIBUSB_LIBRARIES})
else()
  MESSAGE(STATUS "==== LibUSB not found, support for TE923/Voltcraft disabled!")
ENDIF(LIBUSB_FOUND)

## support lua popen on Linux platforms
#IF(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
#	SET(OperatingSystem "Linux")
#	add_definitions(-DLUA_USE_LINUX)
#ENDIF(${CMAKE_SYSTEM_NAME} MATCHES "Linux")

# try to find open-zwave, if found, include support
find_library(OpenZWave NAMES libopenzwave.a HINTS "../open-zwave-read-only" "../open-zwave-read-only/cpp/build")
IF(OpenZWave)
  message(STATUS ${OpenZWave})
  target_link_libraries(domoticz ${OpenZWave})
  include_directories(${CMAKE_SOURCE_DIR}/hardware/openzwave) 
  add_definitions(-DWITH_OPENZWAVE)
  # open-zwave needs libudev
  IF(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(IOKIT_LIBRARY "-framework IOKit -framework CoreFoundation" CACHE FILEPATH "IOKit framework" FORCE)
    target_link_libraries(domoticz ${IOKIT_LIBRARY})
  else()
    find_library(UDEV NAMES libudev.a)
    IF(UDEV)
      message(STATUS ${UDEV})
      target_link_libraries(domoticz ${UDEV} -lrt)
    else()
      find_library(UDEV NAMES libudev.so)
      IF(UDEV)
         message(STATUS ${UDEV})
         target_link_libraries(domoticz ${UDEV} -lrt)
      else()
         MESSAGE(FATAL_ERROR "LIB UDEV not found on your system, see install.txt how to get them installed.\nsudo apt-get install libudev-dev")
      ENDIF(UDEV)
    ENDIF(UDEV)
  ENDIF()
else()
	MESSAGE(STATUS "==== OpenZWave not found, support disabled!")
ENDIF(OpenZWave)

# try to detect Raspberry Pi hardware and WiringPi library. If found, include GPIO support
find_path(RaspberryPi NAMES bcm_host.h PATHS "/opt/vc/include")
IF(RaspberryPi)
	message(STATUS "Building on Raspberry Pi... (bcm_host.h found in ${RaspberryPi})")
	find_path(WiringPi wiringPi.h)
	IF(WiringPi)
		message(STATUS "WiringPi include was found in ${WiringPi}...")
		find_library(WIRINGPI_LIBRARY wiringPi)
		IF(WIRINGPI_LIBRARY)
			message(STATUS "WiringPi library was found in ${WIRINGPI_LIBRARY}...")
			add_definitions(-DWITH_GPIO)
			target_link_libraries(domoticz ${WIRINGPI_LIBRARY})		
		else()
			message(STATUS "==== WiringPi library not found. GPIO support disabled.")
		message(STATUS "==== (Please follow http://wiringpi.com/download-and-install/ if you want to use GPIO.)")
		ENDIF()
	else()
		message(STATUS "==== WiringPi include file (wiringPi.h) not found. GPIO support disabled.")
		message(STATUS "==== (Please follow http://wiringpi.com/download-and-install/ if you want to use GPIO.)")
	ENDIF(WiringPi)
else()
	message(STATUS "Not building on Raspberry Pi. GPIO support disabled.")
ENDIF(RaspberryPi)

target_link_libraries(domoticz ${Boost_LIBRARIES} ${SQLITE3_LIBRARY} ${ZLIB_LIBRARIES} ${CURL_LIBRARIES} pthread lua ${CMAKE_DL_LIBS})

ADD_PRECOMPILED_HEADER(domoticz "main/stdafx.h")

# If it wasn't overridden from command line / cache.
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
	set(CMAKE_INSTALL_PREFIX "/opt/${CPACK_PACKAGE_NAME}" CACHE PATH "Install path prefix, prepended onto install directories." FORCE)
endif(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

install(TARGETS domoticz DESTINATION /opt/domoticz)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/www DESTINATION /opt/domoticz USE_SOURCE_PERMISSIONS)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/License.txt DESTINATION /opt/domoticz)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/History.txt DESTINATION /opt/domoticz)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/svnversion.h DESTINATION /opt/domoticz)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/scripts DESTINATION /opt/domoticz USE_SOURCE_PERMISSIONS)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/Config DESTINATION /opt/domoticz USE_SOURCE_PERMISSIONS)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/updatedomo DESTINATION /opt/domoticz PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ GROUP_EXECUTE GROUP_READ)

# build a CPack driven installer package
INCLUDE(InstallRequiredSystemLibraries)

SET(CPACK_GENERATOR "DEB")
SET(CPACK_PACKAGE_NAME "domoticz")
SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${domoticz}-A program for home automation (Domotica).")
SET(CPACK_PACKAGE_VENDOR "Domoticz.com")
SET(CPACK_PACKAGE_CONTACT "info@domoticz.com")
SET(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")

SET(CPACK_PACKAGE_VERSION_MAJOR "${domoticz_VERSION_MAJOR}")
SET(CPACK_PACKAGE_VERSION_MINOR "${domoticz_VERSION_MINOR}")
SET(CPACK_PACKAGE_VERSION_PATCH "${domoticz_VERSION_PATCH}")

SET(CPACK_PACKAGE_INSTALL_DIRECTORY "domoticz")
#SET (DOMOTICZ_VERSION_SHORT "${domoticz_VERSION_MAJOR}.${domoticz_VERSION_MINOR}.${domoticz_VERSION_PATCH}")
#SET(CPACK_PACKAGE_FILE_NAME "domoticz-${DOMOTICZ_VERSION_SHORT}-${CMAKE_SYSTEM_NAME}")

SET(CPACK_SOURCE_PACKAGE_FILE_NAME "${domoticz}-dev")
SET(CPACK_PACKAGING_INSTALL_PREFIX "/")
SET(CPACK_PACKAGE_DEFAULT_LOCATION "/opt/${CPACK_PACKAGE_NAME}")
SET(CPACK_PACKAGE_EXECUTABLES "domoticz;A program for home automation (Domotica).")

#set(CPACK_DEB_COMPONENT_INSTALL TRUE)
SET(CPACK_DEBIAN_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR})

INCLUDE(CPack)

INCLUDE(TestBigEndian)

TEST_BIG_ENDIAN(BIGENDIAN)
IF(${BIGENDIAN})
     ADD_DEFINITIONS(-DIS_BIG_ENDIAN)
ENDIF(${BIGENDIAN})
