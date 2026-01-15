cmake_minimum_required( VERSION 3.15...99.99)

project ( mulle-clang-cpack NONE) # VERSION 4.0.0.0)

if( "$ENV{VERSION}" STREQUAL "")
   message( FATAL_ERROR "Environment variable VERSION not set")
endif()

set( PROJECT_VERSION $ENV{VERSION})
set( VERSION $ENV{VERSION})
set( RC $ENV{RC})

set( NAME "${VERSION}${RC}")

#
# just some libraries
#
install( DIRECTORY "opt/mulle-clang-project/${NAME}" DESTINATION "/opt/mulle-clang-project" USE_SOURCE_PERMISSIONS)

#
# create a symlink for binary
#
if(CMAKE_HOST_UNIX)
#   message( STATUS "Create symlinks")
#   execute_process(COMMAND ln -s ../../opt/mulle-clang/${NAME}/bin/clang-cpp mulle-clang-cpp)
   execute_process(COMMAND ln -s ../../opt/mulle-clang-project/${NAME}/bin/mulle-scan-build mulle-scan-build)
   execute_process(COMMAND ln -s ../../opt/mulle-clang-project/${NAME}/bin/mulle-nm mulle-nm)
   execute_process(COMMAND ln -s ../../opt/mulle-clang-project/${NAME}/bin/mulle-clang mulle-clang)
#   execute_process(COMMAND ln -s ../../opt/mulle-clang/${NAME}/bin/lldb mulle-lldb)
#   execute_process(COMMAND ln -s ../../opt/mulle-clang/${NAME}/bin/lldb-mi mulle-lldb-mi)

   message( STATUS "Install symlinks in ${CMAKE_BINARY_DIR}")
   install( FILES ${CMAKE_BINARY_DIR}/mulle-clang DESTINATION "/usr/bin")
   install( FILES ${CMAKE_BINARY_DIR}/mulle-scan-build DESTINATION "/usr/bin")
   install( FILES ${CMAKE_BINARY_DIR}/mulle-nm DESTINATION "/usr/bin")
#   install( FILES ${CMAKE_BINARY_DIR}/mulle-lldb DESTINATION "/usr/bin")
#   install( FILES ${CMAKE_BINARY_DIR}/mulle-lldb-mi DESTINATION "/usr/bin")
endif(CMAKE_HOST_UNIX)

#
# CPack and project specific stuff
#
######
set( CPACK_PACKAGE_NAME "mulle-clang")
set( CPACK_PACKAGE_VERSION "${NAME}")
set( CPACK_STRIP_FILES true)

# stuff needed for Debian
# memo: its impossible to check for chosen generator here

set( CPACK_PACKAGE_CONTACT "Nat! <nat-cpack@mulle-kybernetik.com>")

set( CPACK_DEBIAN_PACKAGE_HOMEPAGE "https://github.com/mulle-cc/mulle-clang-project")
set( CPACK_DEBIAN_PACKAGE_DESCRIPTION "mulle-clang, the mulle-objc compiler based on clang")

set( CPACK_RPM_PACKAGE_VENDOR "Mulle kybernetiK")
set( CPACK_RPM_PACKAGE_DESCRIPTION "${CPACK_DEBIAN_PACKAGE_DESCRIPTION}")
set( CPACK_RPM_PACKAGE_GROUP "Development/Tools")
set( CPACK_RPM_PACKAGE_LICENSE "Apache-2.0 with LLVM-exception")
set( CPACK_RPM_PACKAGE_URL "https://github.com/mulle-cc/mulle-clang-project")

######
#
# must be after CPACK_PACKAGE_VERSION
include( CPack)

