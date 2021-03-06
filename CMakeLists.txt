if(PACKAGE)
  cmake_minimum_required(VERSION 2.8.12)
else()
  cmake_minimum_required(VERSION 2.8)
endif()

message(STATUS "CMake version: ${CMAKE_VERSION}")

include(macros.cmake)

enable_testing()
set_variables() 
set_maxscale_version()

set(CMAKE_INSTALL_PREFIX "/usr/local/mariadb-maxscale" CACHE PATH "Prefix prepended to install directories.")
set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING "Choose the type of build, options are: None(CMAKE_CXX_FLAGS or CMAKE_C_FLAGS used) Debug Release RelWithDebInfo MinSizeRel.")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

project(MaxScale)

#Disabled for now pending evaluation
include(cmake/CheckPlatform.cmake) 

check_deps()
check_dirs()
find_package(Valgrind)
find_package(MySQLClient)
find_package(MySQL)
find_package(Pandoc)
find_package(TCMalloc)
find_package(Jemalloc)
# You can find the variables set by this in the FindCURL.cmake file
# which is a default module in CMake.
find_package(CURL) 
if(NOT CURL_FOUND)
  message(FATAL_ERROR "Failed to locate dependency: libcurl")
endif()

set(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_RPATH}:${CMAKE_INSTALL_PREFIX}/lib:${CMAKE_INSTALL_PREFIX}/modules)

# Make sure the release notes for this release are present if it is a stable one
if(${MAXSCALE_VERSION} MATCHES "-stable")
  file(GLOB ${CMAKE_SOURCE_DIR}/Documentation/Release-Notes RELEASE_NOTES *${MAXSCALE_VERSION_NUMERIC}*.md)
  list(LENGTH RELEASE_NOTES HAVE_NOTES)
  if( NOT HAVE_NOTES EQUAL 1)
    message(FATAL_ERROR "Could not find the release notes for this stable release: ${MAXSCALE_VERSION_NUMERIC}")
  endif()
endif()

file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/server/include)
configure_file(${CMAKE_SOURCE_DIR}/server/include/version.h.in ${CMAKE_BINARY_DIR}/server/include/version.h)
configure_file(${CMAKE_SOURCE_DIR}/server/test/maxscale_test.h.in ${CMAKE_BINARY_DIR}/server/include/maxscale_test.h)
configure_file(${CMAKE_SOURCE_DIR}/etc/postinst.in ${CMAKE_BINARY_DIR}/postinst)
configure_file(${CMAKE_SOURCE_DIR}/etc/postrm.in ${CMAKE_BINARY_DIR}/postrm)

set(FLAGS "-Wall -Wno-unused-variable -Wno-unused-function -fPIC" CACHE STRING "Compilation flags")
set(DEBUG_FLAGS "-ggdb -pthread -pipe -Wformat -fstack-protector --param=ssp-buffer-size=4" CACHE STRING "Debug compilation flags")

if(CMAKE_VERSION VERSION_GREATER 2.6)
  if((CMAKE_C_COMPILER_ID STREQUAL "GNU") AND (NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 4.2)))
    message(STATUS "C Compiler supports: -Werror=format-security")
    set(DEBUG_FLAGS "${DEBUG_FLAGS} -Werror=format-security" CACHE STRING "Debug compilation flags")
  endif()

  if((CMAKE_C_COMPILER_ID STREQUAL "GNU") AND (NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 4.6)))
    message(STATUS "C Compiler supports: -Wno-unused-but-set-variable")
    set(FLAGS "${FLAGS} -Wno-unused-but-set-variable " CACHE STRING "Compilation flags")
  endif()
endif()


IF(DEFINED OLEVEL )
  if((OLEVEL GREATER -1) AND (OLEVEL LESS 4) )
    set(FLAGS "${FLAGS} -O${OLEVEL}" CACHE STRING "Compilation flags"  FORCE)
	message(STATUS "Optimization level at: ${OLEVEL}")
  else()
	message(WARNING "Optimization level was set to a bad value, ignoring it. (Valid values are 0-3)")
  endif()
endif()

if(GCOV)
  set(FLAGS "${FLAGS} -fprofile-arcs -ftest-coverage" CACHE STRING "Compilation flags"  FORCE)
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lgcov")
endif()

if(FAKE_CODE)
  set(FLAGS "${FLAGS} -DFAKE_CODE" CACHE STRING "Compilation flags"  FORCE)
endif()

if(PROFILE)
  message(STATUS "Profiling executables")
  set(FLAGS "${FLAGS} -pg " CACHE STRING "Compilation flags" FORCE)
endif()

set(CMAKE_C_FLAGS "${FLAGS}")
set(CMAKE_C_FLAGS_DEBUG "${DEBUG_FLAGS} -DSS_DEBUG -DLOG_ASSERT")
set(CMAKE_C_FLAGS_RELEASE "")
set(CMAKE_C_FLAGS_RELWITHDEBINFO "-ggdb")

set(CMAKE_CXX_FLAGS "${FLAGS}")
set(CMAKE_CXX_FLAGS_DEBUG "${DEBUG_FLAGS} -DSS_DEBUG -DLOG_ASSERT")
set(CMAKE_CXX_FLAGS_RELEASE "")
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-ggdb")

subdirs(MYSQL_DIR_ALL ${MYSQL_DIR})
foreach(DIR ${MYSQL_DIR_ALL})
  include_directories(${DIR})
endforeach()

include_directories(${MYSQL_DIR}/..)
include_directories(utils)
include_directories(log_manager)
include_directories(query_classifier)
include_directories(server/include)
include_directories(server/include/maxscale)
include_directories(server/inih)
include_directories(server/modules/include)
include_directories(${CMAKE_BINARY_DIR}/server/include)
include_directories(${CURL_INCLUDE_DIRS})

add_subdirectory(plugins)
add_subdirectory(utils)
add_subdirectory(log_manager)
add_subdirectory(query_classifier)
add_subdirectory(server)
if(NOT WITHOUT_MAXADMIN)
  add_subdirectory(client)
endif()


execute_process(COMMAND perl ${CMAKE_SOURCE_DIR}/Documentation/format.pl 
${CMAKE_SOURCE_DIR}/Documentation/Changelog.md
${CMAKE_BINARY_DIR}/Changelog.txt)
execute_process(COMMAND perl ${CMAKE_SOURCE_DIR}/Documentation/format.pl 
${CMAKE_SOURCE_DIR}/Documentation/Release-Notes/MaxScale-1.1-Release-Notes.md
 ${CMAKE_BINARY_DIR}/ReleaseNotes.txt)
execute_process(COMMAND perl ${CMAKE_SOURCE_DIR}/Documentation/format.pl 
${CMAKE_SOURCE_DIR}/Documentation/Upgrading-To-MaxScale-1.1.0.md
 ${CMAKE_BINARY_DIR}/UpgradingToMaxScale110.txt)
install(FILES ${CMAKE_BINARY_DIR}/Changelog.txt DESTINATION .)
install(FILES ${CMAKE_BINARY_DIR}/ReleaseNotes.txt DESTINATION .)
install(FILES ${CMAKE_BINARY_DIR}/UpgradingToMaxScale110.txt DESTINATION .)

message(STATUS "Installing MaxScale to: ${CMAKE_INSTALL_PREFIX}/")

install(FILES server/MaxScale_template.cnf DESTINATION etc)
install(FILES server/MaxScale_BinlogServer_template.cnf DESTINATION etc)
install(FILES ${ERRMSG} DESTINATION mysql)
install(FILES ${CMAKE_SOURCE_DIR}/COPYRIGHT DESTINATION .)
install(FILES ${CMAKE_SOURCE_DIR}/README DESTINATION .)
install(FILES ${CMAKE_SOURCE_DIR}/LICENSE DESTINATION .)
install(DIRECTORY DESTINATION log)

# Install startup scripts and ldconfig files
if(WITH_SCRIPTS)
  configure_file(${CMAKE_SOURCE_DIR}/maxscale.conf.in ${CMAKE_BINARY_DIR}/maxscale.conf @ONLY)
  if(DEB_BASED)
    configure_file(${CMAKE_SOURCE_DIR}/etc/ubuntu/init.d/maxscale.in ${CMAKE_BINARY_DIR}/maxscale @ONLY)    
  else()
    configure_file(${CMAKE_SOURCE_DIR}/etc/init.d/maxscale.in ${CMAKE_BINARY_DIR}/maxscale @ONLY)    
  endif()
  if(PACKAGE)
    message(STATUS "maxscale.conf will unpack to: /etc/ld.so.conf.d")
    message(STATUS "startup scripts will unpack to to: /etc/init.d")
  else()
  install(FILES ${CMAKE_BINARY_DIR}/maxscale DESTINATION /etc/init.d 
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
  install(FILES ${CMAKE_BINARY_DIR}/maxscale.conf DESTINATION /etc/ld.so.conf.d 
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
  message(STATUS "Installing maxscale.conf to: /etc/ld.so.conf.d")
  message(STATUS "Installing startup scripts to: /etc/init.d")
  endif()
endif()

if(PACKAGE)
  install(FILES ${CMAKE_BINARY_DIR}/maxscale DESTINATION .
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
  install(FILES ${CMAKE_BINARY_DIR}/maxscale.conf DESTINATION .
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
  install(FILES ${CMAKE_BINARY_DIR}/postinst DESTINATION .
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
  install(FILES ${CMAKE_BINARY_DIR}/postrm DESTINATION .
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
  if(${CMAKE_VERSION} VERSION_LESS 2.8.12)
    message(WARNING "CMake version is ${CMAKE_VERSION}. Building of packages requires version 2.8.12 or greater.")
  else()
    # See if we are on a RPM-capable or DEB-capable system
    find_program(RPMBUILD rpmbuild)
    find_program(DEBBUILD dpkg-buildpackage)
    set(CPACK_GENERATOR "TGZ")
    if(NOT ( ${RPMBUILD} STREQUAL "RPMBUILD-NOTFOUND" ) )
      message(STATUS "Generating RPM packages")
      set(CPACK_GENERATOR "${CPACK_GENERATOR};RPM")
    endif()

    if(NOT ( ${DEBBUILD} STREQUAL "DEBBUILD-NOTFOUND" ) )
      set(CPACK_GENERATOR "${CPACK_GENERATOR};DEB")
      execute_process(COMMAND dpgk --print-architecture OUTPUT_VARIABLE DEB_ARCHITECTURE)
      set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE ${DEB_ARCHITECTURE})
      set (CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON) 
      message(STATUS "Generating DEB packages for ${DEB_ARCHITECTURE}")
    endif()

    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "MaxScale")
    set(CPACK_PACKAGE_VERSION_MAJOR "${MAXSCALE_VERSION_MAJOR}")
    set(CPACK_PACKAGE_VERSION_MINOR "${MAXSCALE_VERSION_MINOR}")
    set(CPACK_PACKAGE_VERSION_PATCH "${MAXSCALE_VERSION_PATCH}")
    set(CPACK_PACKAGE_CONTACT "MariaDB Corporation Ab")
    set(CPACK_PACKAGE_FILE_NAME "maxscale-${MAXSCALE_VERSION}")
    set(CPACK_PACKAGE_NAME "maxscale")
    set(CPACK_PACKAGE_VENDOR "MariaDB Corporation Ab")
    set(CPACK_PACKAGE_DESCRIPTION_FILE ${CMAKE_SOURCE_DIR}/etc/DESCRIPTION)
    set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
    set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_BINARY_DIR}/postinst;{CMAKE_BINARY_DIR}/postrm")
    set(CPACK_RPM_PACKAGE_RELEASE ${MAXSCALE_BUILD_NUMBER})
    set(CPACK_RPM_POST_INSTALL_SCRIPT_FILE ${CMAKE_BINARY_DIR}/postinst)
    set(CPACK_RPM_POST_UNINSTALL_SCRIPT_FILE ${CMAKE_BINARY_DIR}/postrm)
    set(CPACK_RPM_PACKAGE_NAME "maxscale")
    set(CPACK_RPM_PACKAGE_VENDOR "MariaDB Corporation Ab")
    set(CPACK_RPM_PACKAGE_LICENSE "GPLv2")
    set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "/etc /etc/ld.so.conf.d /etc/init.d /etc/rc.d/init.d")
    set(CPACK_RPM_SPEC_MORE_DEFINE "%define ignore \#")
    set(CPACK_RPM_USER_FILELIST "%ignore /etc/init.d")
    set(CPACK_RPM_USER_FILELIST "%ignore /etc/ld.so.conf.d")
    set(CPACK_RPM_USER_FILELIST "%ignore /etc")  
    include(CPack)
  endif()
endif()

add_custom_target(buildtests
  COMMAND ${CMAKE_COMMAND} -DDEPS_OK=Y -DBUILD_TESTS=Y -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR} -DWITH_SCRIPTS=N ${CMAKE_SOURCE_DIR}
  COMMAND make
  COMMENT "Building test suite..." VERBATIM
) 

add_custom_target(testall 
  COMMAND ${CMAKE_COMMAND} -DDEPS_OK=Y -DBUILD_TESTS=Y -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR} -DWITH_SCRIPTS=N ${CMAKE_SOURCE_DIR}
  COMMAND make install
  COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_SOURCE_DIR}/server/test/MaxScale_test.cnf ${CMAKE_BINARY_DIR}/etc/MaxScale.cnf
  COMMAND ${CMAKE_COMMAND} -P ${CMAKE_SOURCE_DIR}/cmake/testall.cmake
  COMMENT "Running full test suite..." VERBATIM)

add_custom_target(testcore
  COMMAND ${CMAKE_COMMAND} -DDEPS_OK=Y -DBUILD_TESTS=Y -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR} -DWITH_SCRIPTS=N ${CMAKE_SOURCE_DIR}
  COMMAND make install
  COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_SOURCE_DIR}/server/test/MaxScale_test.cnf ${CMAKE_BINARY_DIR}/etc/MaxScale.cnf
  COMMAND ctest -R Internal
  COMMENT "Running core test suite..." VERBATIM)

# uninstall target
# see http://www.cmake.org/Wiki/CMake_FAQ#Can_I_do_.22make_uninstall.22_with_CMake.3F
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

find_package(Doxygen)

if(DOXYGEN_FOUND)
  configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/doxygate.in"
    "${CMAKE_CURRENT_BINARY_DIR}/doxygate"
    IMMEDIATE @ONLY)

  add_custom_target(documentation
    COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/doxygate)
  
endif()

# Testall target with Valgrind
if(VALGRIND_FOUND)
add_custom_target(testall-valgrind
  COMMAND ${CMAKE_COMMAND} -DDEPS_OK=Y -DBUILD_TESTS=Y -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR} -DWITH_SCRIPTS=N ${CMAKE_SOURCE_DIR}
  COMMAND make install
  COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_SOURCE_DIR}/server/test/MaxScale_test.cnf ${CMAKE_BINARY_DIR}/etc/MaxScale.cnf
  COMMAND /bin/sh -c "valgrind --track-fds=yes  --leak-check=full --show-leak-kinds=all --log-file=${CMAKE_BINARY_DIR}/valgrind.log ${CMAKE_BINARY_DIR}/bin/maxscale -c ${CMAKE_BINARY_DIR} &>/dev/null"
  COMMAND /bin/sh -c "make test || echo \"Test results written to: ${CMAKE_BINARY_DIR}/Testing/Temporary/\""
  COMMAND killall maxscale
  COMMENT "Running full test suite with Valgrind..." VERBATIM)

endif()

add_custom_target(generate_pdf
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/Documentation ${CMAKE_BINARY_DIR}/Documentation
  COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_BINARY_DIR}/Documentation ${CMAKE_COMMAND} 
  -DBUILD_DIR=${CMAKE_BINARY_DIR}
  -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH} 
  -P generate-pdf.cmake
  COMMENT "Generating PDF files" VERBATIM) 

add_custom_target(generate_txt_release
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/Documentation ${CMAKE_BINARY_DIR}/Documentation
  COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_BINARY_DIR}/Documentation ${CMAKE_COMMAND} 
  -DBUILD_DIR=${CMAKE_BINARY_DIR}
  -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH} 
  -P generate-txt-release.cmake
  COMMENT "Generating TXT release notes" VERBATIM) 


add_custom_target(generate_html
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/Documentation ${CMAKE_BINARY_DIR}/Documentation
  COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_BINARY_DIR}/Documentation ${CMAKE_COMMAND} 
  -DBUILD_DIR=${CMAKE_BINARY_DIR}
  -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH} 
  -P generate-html.cmake
  COMMENT "Generating HTML files" VERBATIM) 

if(PACKAGE)
  message(STATUS "You can install startup scripts and system configuration files for MaxScale by running the 'postinst' shell script located at ${CMAKE_INSTALL_PREFIX}.")
  message(STATUS "To remove these installed files, run the 'postrm' shell script located in the same folder.")
endif()
