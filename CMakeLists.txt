cmake_minimum_required(VERSION 2.8.10.1)
project(QtBuilder)
  #
  # Enable C and CXX languages so CMAKE_SIZEOF_VOID_P indicates whether
  # this is a 64-bit build or not. (Otherwise, we could use 'NONE' for
  # languages in the project command because this project is built entirely
  # with ExternalProject custom build steps...)
  #

include(ExternalProject)
include(CTest)


function(qb_get_qt_version_info version var_tarball_name var_tarball_md5)
  string(REGEX REPLACE "[^A-Za-z0-9_]" "" var_suffix "${version}")

  set(tarball_basename "qt-everywhere-opensource-src-")

  set(qt_tgz_md5_485 "1864987bdbb2f58f8ae8b350dfdbe133")
  set(qt_tgz_md5_521 "a78408c887c04c34ce615da690e0b4c8")
  set(qt_tgz_md5_540 "e8654e4b37dd98039ba20da7a53877e6")

  set(tarball_name "${tarball_basename}${version}.tar.gz")
  set(tarball_md5 "${qt_tgz_md5_${var_suffix}}")

  message("version='${version}'")
  message("var_tarball_name='${var_tarball_name}'")
  message("var_tarball_md5='${var_tarball_md5}'")
  message("var_suffix='${var_suffix}'")
  message("tarball_name='${tarball_name}'")
  message("tarball_md5='${tarball_md5}'")

  set(${var_tarball_name} "${tarball_name}" PARENT_SCOPE)
  set(${var_tarball_md5} "${tarball_md5}" PARENT_SCOPE)
endfunction()


set(QB_VERSION "5.4.0" CACHE STRING
  "Version of Qt to download, build and install")

set(VersionValues
  4.8.5
  5.2.1
  5.4.0
)

set_property(CACHE QB_VERSION PROPERTY STRINGS ${VersionValues})

if(WIN32)
  set(QB_BASEDIR_DEFAULT "C:/Qt")
else()
  set(QB_BASEDIR_DEFAULT "$ENV{HOME}/Qt")
endif()

set(QB_BASEDIR "${QB_BASEDIR_DEFAULT}" CACHE PATH
  "Local base directory for downloading and building")

set(qt_server "http://download.qt-project.org")
set(qt_version "${QB_VERSION}")
qb_get_qt_version_info("${qt_version}" qt_tgz_name qt_tgz_md5)
string(REGEX REPLACE "^([0-9].[0-9]).*$" "\\1" qt_major_minor "${qt_version}")

# Full URL for source .tar.gz of Qt 5.4.0:
# http://download.qt-project.org/official_releases/qt/5.4/5.4.0/single/qt-everywhere-opensource-src-5.4.0.tar.gz

if(qt_version VERSION_LESS 5.0.0)
  set(qt_baseurl "${qt_server}/official_releases/qt/${qt_major_minor}/${qt_version}")
else()
  set(qt_baseurl "${qt_server}/official_releases/qt/${qt_major_minor}/${qt_version}/single")
endif()

set(qt_md5 "${qt_tgz_md5}")
set(qt_url "${qt_baseurl}/${qt_tgz_name}")

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${qt_tgz_name}")
  set(qt_url "${CMAKE_CURRENT_SOURCE_DIR}/${qt_tgz_name}")
endif()

set(suffix "unsupported")

if(MSVC12)
  set(suffix "vs12")
elseif(MSVC11)
  set(suffix "vs11")
elseif(MSVC10)
  set(suffix "vs10")
elseif(MSVC90)
  set(suffix "vs2008")
elseif(MSVC80)
  set(suffix "vs2005")
elseif(MSVC71)
  set(suffix "vs2003")
elseif(CMAKE_COMPILER_IS_GNUCC)
  set(suffix "gcc")
endif()

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(suffix "${suffix}-x64")
endif()

if(WIN32)
#  if(NOT CMAKE_GENERATOR MATCHES "Makefile")
#    message(FATAL_ERROR "
# error: building Qt on Windows requires using a Makefile generator
#
#  The Visual Studio generators are not supported.
#  Run cmake with -G \"NMake Makefiles\" or -G \"NMake Makefiles JOM\"
#  from a Visual Studio command prompt...
# ")
#  endif()

  #
  # Windows builds with nmake/jom are "in-source/in-place" builds.
  #   i.e. source == build == install dir
  #   and prefix is irrelevant...
  #
  set(qt_dir "${QB_BASEDIR}/${qt_version}-${suffix}")
  set(cfg_cmd "configure.bat")
  set(prefix_arg "")
  set(other_args -opensource -confirm-license
    -nomake examples
    -shared -release
  )
else()

  #
  # Linux/Mac builds with make honor --prefix. They do not necessarily have to
  # be "in-source/in-place" builds, although they can be... So we do in-source
  # here for consistency with the Windows builds.
  #
  set(qt_dir "${QB_BASEDIR}/${qt_version}-${suffix}")
  set(cfg_cmd "configure")
  set(prefix_arg --prefix=${qt_dir})
  set(other_args --opensource --confirm-license
    -nomake examples
    -shared -release
  )

  if(APPLE)
    set(other_args ${other_args} -cocoa)
    if("$ENV{QMAKE_MACOSX_DEPLOYMENT_TARGET}" STREQUAL "")
      message(AUTHOR_WARNING "Environment variable QMAKE_MACOSX_DEPLOYMENT_TARGET is not set. Current OS version will be the default deployment target.")
    endif()
  endif()
endif()

if(qt_version VERSION_LESS 5.0.0)
  set(qt_bin_dir "${qt_dir}/bin")
  set(qt_moc_v_output "Meta Object Compiler")
else()
  set(qt_bin_dir "${qt_dir}/qtbase/bin")
  set(qt_moc_v_output "moc ")
endif()

# CACHE so users can add-to/modify the Qt configure args if necessary:
#
set(QB_CONFIGURE_ARGS ${prefix_arg} ${other_args} CACHE STRING
  "Args for Qt configure step")

set(md5_args "")
if(qt_md5)
  set(md5_args URL_MD5 ${qt_md5})
endif()


file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/qt.gitignore" "#\n")


if(WIN32)
  set(build_cmd "jom")
else()
  set(build_cmd "$(MAKE)")
endif()


# Requirements for Qt 5.0 and up to build properly (at least on Windows with VS):
# Run in a VS command prompt, with jom, perl and python also added to the PATH.


ExternalProject_Add(qt
  URL ${qt_url}
  ${md5_args}
  DOWNLOAD_DIR ${QB_BASEDIR}/Downloads
  SOURCE_DIR ${qt_dir}
  BUILD_IN_SOURCE 1
  PATCH_COMMAND
    ${CMAKE_COMMAND}
      -E copy
      "${CMAKE_CURRENT_BINARY_DIR}/qt.gitignore"
      "<SOURCE_DIR>/.gitignore"
    COMMAND ${CMAKE_COMMAND}
      -E copy
      "${CMAKE_CURRENT_BINARY_DIR}/qt.gitignore"
      "<SOURCE_DIR>/qtbase/.gitignore"
  CONFIGURE_COMMAND <SOURCE_DIR>/${cfg_cmd} ${QB_CONFIGURE_ARGS}
  BUILD_COMMAND ${build_cmd}
  INSTALL_COMMAND ""
  STEP_TARGETS download
)


message(STATUS "suffix='${suffix}'")
message(STATUS "qt_dir='${qt_dir}'")
message(STATUS "qt_url='${qt_url}'")


add_test(SmokeTest-Qt-${qt_version}-moc "${qt_bin_dir}/moc" "-v")
add_test(SmokeTest-Qt-${qt_version}-qmake "${qt_bin_dir}/qmake" "-v")
add_test(SmokeTest-Qt-${qt_version}-uic "${qt_bin_dir}/uic" "-v")

set_property(TEST SmokeTest-Qt-${qt_version}-moc PROPERTY
  PASS_REGULAR_EXPRESSION "${qt_moc_v_output}")
