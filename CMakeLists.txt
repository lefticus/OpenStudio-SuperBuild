CMAKE_MINIMUM_REQUIRED( VERSION 2.8.7 )

PROJECT(OpenStudio)

SET(CMAKE_VERSION_MAJOR 1)
SET(CMAKE_VERSION_MINOR 2)
SET(CMAKE_VERSION_PATCH 1)

SET(CMAKE_VERSION_BUILD "Unknown" CACHE STRING "Build number")
FIND_PACKAGE(Git)
IF(GIT_FOUND)
  EXECUTE_PROCESS(COMMAND "${GIT_EXECUTABLE}" "rev-parse" "--short=10" "HEAD"
                  WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
                  TIMEOUT 10
                  RESULT_VARIABLE RESULT
                  OUTPUT_VARIABLE GIT_VERSION
                  ERROR_QUIET
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
  IF(${RESULT} EQUAL 0)
    SET(CMAKE_VERSION_BUILD ${GIT_VERSION} CACHE STRING "Build number" FORCE) # git sha                  
  ENDIF()
ENDIF()

# EnergyPlus Idd version
SET(ENERGYPLUS_VERSION "8.0" CACHE INTERNAL "EnergyPlus Version")


OPTION( NIGHTLY_BUILD "Use configurations for Nightly Build" OFF )

SET(OPENSTUDIO_VERSION "${CMAKE_VERSION_MAJOR}.${CMAKE_VERSION_MINOR}.${CMAKE_VERSION_PATCH}")

# Search for modules in the openstudiocore dir first to override cmake ones 
LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/openstudiocore/CMake" ${CMAKE_SOURCE_DIR})

# Build C++ documentation using Doxygen
# Requires: doxygen
OPTION( BUILD_DOCUMENTATION "Build Documenation" OFF )

IF(BUILD_DOCUMENTATION)
  Find_Package( Doxygen REQUIRED )
ENDIF()

# Build CSharp bindings
# Requires: SWIG
OPTION( BUILD_CSHARP_BINDINGS "Build CSharp bindings" OFF )

# Build Node V8 bindings
# Requires: SWIG
OPTION( BUILD_V8_BINDINGS "Build V8 bindings" OFF )

# Build Python bindings
# Requires: SWIG Python
OPTION( BUILD_PYTHON_BINDINGS "Build Python bindings" OFF )

# Build ctest testing
# Requires: EnergyPlus
OPTION( BUILD_TESTING "Build testing targets" OFF )

# Build package
# Requires: EnergyPlus
OPTION( BUILD_PACKAGE "Build package" OFF )

# Build test runner targets.
# This is a convenience for Visual Studio users
OPTION( ENABLE_TEST_RUNNER_TARGETS "Create test runner targets" OFF )

# Build with OpenSSL support
SET( BUILD_WITH_OPENSSL ON CACHE INTERNAL "Build With OpenSSL Support For SSH Connections" )
IF(UNIX)
  FIND_PACKAGE( OpenSSL )
  IF( NOT ${OPENSSL_FOUND} )
    MESSAGE(SEND_ERROR "OpenSSL could not be found, and is required for HTTPS communication")
    MESSAGE(SEND_ERROR "Please install OpenSSL development libraries using your package management system (possibly libssl-dev)")
  ELSE()
    MARK_AS_ADVANCED(LIB_EAY_DEBUG)
    MARK_AS_ADVANCED(LIB_EAY_RELEASE)
    MARK_AS_ADVANCED(SSL_EAY_DEBUG)
    MARK_AS_ADVANCED(SSL_EAY_RELEASE)
  ENDIF()
ELSEIF(WIN32)
  IF(CMAKE_CL_64)
    SET(OPENSSL_PATH "OpenSSL-Win64")
    IF(NOT EXISTS "${CMAKE_BINARY_DIR}/${OPENSSL_PATH}.zip")
      MESSAGE(STATUS "Downloading OpenSSL (x64)")
    ENDIF()
    FILE(DOWNLOAD "http://developer.nrel.gov/downloads/buildings/openstudio/src/OpenSSL-Win64.zip" "${CMAKE_BINARY_DIR}/${OPENSSL_PATH}.zip" TIMEOUT 120 INACTIVITY_TIMEOUT 120 SHOW_PROGRESS EXPECTED_MD5 974FDD1EDD3CBDCC54A0EEEE0DBB00CC)
  ELSE()
    SET(OPENSSL_PATH "OpenSSL-Win32")
    IF(NOT EXISTS "${CMAKE_BINARY_DIR}/${OPENSSL_PATH}.zip")
      MESSAGE(STATUS "Downloading OpenSSL (x86)")
    ENDIF()
    FILE(DOWNLOAD "http://developer.nrel.gov/downloads/buildings/openstudio/src/OpenSSL-Win32.zip" "${CMAKE_BINARY_DIR}/${OPENSSL_PATH}.zip" TIMEOUT 120 INACTIVITY_TIMEOUT 120 SHOW_PROGRESS EXPECTED_MD5 890EA14E30C5BE3C4F9298DBC516AA33)
  ENDIF()
  EXECUTE_PROCESS(COMMAND ${CMAKE_COMMAND} -E remove_directory "${CMAKE_BINARY_DIR}/${OPENSSL_PATH}")
  EXECUTE_PROCESS(COMMAND ${CMAKE_COMMAND} -E tar xfz "${CMAKE_BINARY_DIR}/${OPENSSL_PATH}.zip" WORKING_DIRECTORY "${CMAKE_BINARY_DIR}")
  SET(OPENSSL_ROOT_DIR "${CMAKE_BINARY_DIR}/${OPENSSL_PATH}")
ENDIF()



IF(MSVC)
  # Build with Multiple Processes
  SET(BUILD_WITH_MULTIPLE_PROCESSES ON CACHE BOOL "/MP compiler flag for full processor utilization")
  MARK_AS_ADVANCED( BUILD_WITH_MULTIPLE_PROCESSES )
ENDIF()

# EnergyPlus
FIND_PACKAGE(EnergyPlus "${ENERGYPLUS_VERSION}" REQUIRED)
MARK_AS_ADVANCED(
  ENERGYPLUS_EXE
  ENERGYPLUS_IDD
  ENERGYPLUS_WEATHER_DIR
)

# Package Radiance and EnergyPlus for convenience 
OPTION( PACKAGE_RADIANCE_AND_ENERGYPLUS "Package Radiance and EnergyPlus" OFF )
MARK_AS_ADVANCED(FORCE PACKAGE_RADIANCE_AND_ENERGYPLUS)

IF (PACKAGE_RADIANCE_AND_ENERGYPLUS)
  MESSAGE(WARNING "Packaging of Radiance and EnergyPlus is intended for internal use and deployment only")
  SET(ENERGYPLUS_LOCATION "" CACHE PATH "Location of EnergyPlus to Package")
  SET(RADIANCE_LOCATION "" CACHE PATH "Location of Radiance to Package")
ENDIF()

# Build Ruby gem
OPTION( BUILD_RUBY_GEM "Build Ruby Gem" OFF )

IF (BUILD_V8_BINDINGS)
  IF(WIN32)
    MESSAGE(SEND_ERROR "V8 is only tested and supported on Unix like systems")
  ENDIF()

  OPTION( BUILD_NODE_MODULES "Build V8 Bindings as Node Modules" ON )

  IF ( BUILD_NODE_MODULES )
    # Build Node for linking to Node modules
    OPTION( BUILD_NODE "Build Node" ON )
  ELSE()
    OPTION( BUILD_V8 "Build V8" ON )
  ENDIF()

ENDIF()

IF (BUILD_RUBY_GEM)
  # Build Ruby gem and package up all the libs
  OPTION( BUILD_RUBY_GEM_WITH_LIBS "Package all required libs with the Ruby Gem" ON )

  SET(BUILD_PACKAGE ON)
  SET(CPACK_BINARY_DEB OFF)
  SET(CPACK_BINARY_NSIS OFF)
  SET(CPACK_BINARY_RPM OFF)
  SET(CPACK_BINARY_STGZ OFF)
  SET(CPACK_BINARY_TBZ2 OFF)
  SET(CPACK_BINARY_TZ OFF)
  SET(CPACK_BINARY_TGZ ON)

  IF(CPACK_BINARY_DEB OR CPACK_BINARY_NSIS OR CPACK_BINARY_RPM OR CPACK_BINARY_STGZ
      OR CPACK_BINARY_TBZ2 OR CPACK_BINARY_TZ OR NOT CPACK_BINARY_TGZ OR NOT BUILD_PACKAGE)
    MESSAGE(FATAL_ERROR "To build the Ruby Gem please enable ONLY CPACK_BINARY_TGZ of the cpack binary options")
  ENDIF()

  MESSAGE("Enabling Ruby Gem overrides all of your packaging options. To build the Ruby Gem please type 'make package'")
ENDIF()

# Require that DAKOTA_FOUND
#
# Some code (certain analysisdriver tests) are conditionally compiled depending on 
# whether Dakota is found on the system. If this variable is set to true, CMake will give a configure
# error in the case that Dakota is not found.
#
# Dakota is an optional third party algorithm library that we use through its command line
# interface.
OPTION( REQUIRE_DAKOTA "Ensure Dakota (third party analysis tool) is found" OFF )
MARK_AS_ADVANCED( REQUIRE_DAKOTA )

IF(MSVC)
  # need to know if using the Express edition of Visual Studio, devenv is split into multiple programs
  OPTION( MSVC_IS_EXPRESS "using the Express editions of Visual Studio, must install both C++ and C# versions" FALSE )
  
  MARK_AS_ADVANCED(CMAKE_CONFIGURATION_TYPES)
ENDIF()

# Use PCH
OPTION( USE_PCH "Use precompiled headers" OFF )

FIND_PACKAGE (Qt4 4.8.0 COMPONENTS QtCore QtGui QtSql QtNetwork QtXml QtXmlPatterns QtWebKit)
MARK_AS_ADVANCED(QT_QTMOTIF_INCLUDE_DIR)
MARK_AS_ADVANCED(QT_QTMOTIF_LIBRARY_DEBUG)
MARK_AS_ADVANCED(QT_QTMOTIF_LIBRARY_RELEASE)

  # Build Qt
IF(QT_QMAKE_EXECUTABLE)
  OPTION( BUILD_QT "Build Qt" OFF)
ELSE()
  OPTION( BUILD_QT "Build Qt" ON)
ENDIF()

# Apple Sanity Check
IF(APPLE)
  IF (NOT CMAKE_OSX_DEPLOYMENT_TARGET)
    MESSAGE(FATAL_ERROR "OSX Deployment Target Must Be Set")
  ENDIF()

  IF (NOT CMAKE_OSX_SYSROOT)
    MESSAGE(FATAL_ERROR "OSX Sysroot must be set")
  ENDIF()

  IF (CMAKE_OSX_DEPLOYMENT_TARGET MATCHES ".*10\\.9.*" OR CMAKE_OSX_SYSROOT MATCHES ".*10\\.9.*")
    MESSAGE(WARNING "You are building against the 10.9 SDK which means ruby 2.0")
  ENDIF()

ENDIF()

include(ExternalProject)
include(ProcessorCount)

ProcessorCount(CPUCOUNT)
if(CPUCOUNT EQUAL 0)
  SET(CPUCOUNT "1")
endif()

MARK_AS_ADVANCED(Boost_DIR)
MARK_AS_ADVANCED(Boost_LIB_DIAGNOSTIC_DEFINITIONS)
MARK_AS_ADVANCED(ProcessorCount_cmd_getconf)
MARK_AS_ADVANCED(ProcessorCount_cmd_sysctl)

# Fix for OSX 10.9 Mavericks, allows osx_deployment_target=10.9
if(APPLE)
    set(apple-frameworks "-stdlib=libstdc++")
    set(CMAKE_SHARED_LINKER_FLAGS " ${apple-frameworks} ${CMAKE_SHARED_LINKER_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS " ${apple-frameworks} ${CMAKE_EXE_LINKER_FLAGS}")
    set(CMAKE_MODULE_LINKER_FLAGS " ${apple-frameworks} ${CMAKE_MODULE_LINKER_FLAGS}")
    set(CMAKE_CXX_FLAGS " -stdlib=libstdc++ ${CMAKE_CXX_FLAGS}")
endif()

# Boost
IF(APPLE)
  set(Boost_USE_STATIC_LIBS ON)  
ENDIF()
IF (WIN32)
  set(Boost_USE_STATIC_LIBS ON)  

  #uncomment all of this if we want to force dynamic libs on windows
#  set(Boost_USE_STATIC_LIBS OFF)  
#  ADD_DEFINITIONS( -DBOOST_THREAD_USE_DLL -DBOOST_THREAD_DYN_LINK -DBOOST_PROGRAM_OPTIONS_DYN_LINK  -DBOOST_REGEX_DYN_LINK  -DBOOST_FILESYSTEM_DYN_LINK -DBOOST_SYSTEM_DYN_LINK -DBOOST_SERIALIZATION_DYN_LINK  -DBOOST_DATE_TIME_DYN_LINK)
#  LINK_DIRECTORIES( ${Boost_LIBRARY_DIRS} )
#  IF (MSVC)
#    #Ignore dll specific warnings that are out of our hands to control, coming from external projects
#    ADD_DEFINITIONS( "/wd4251 /wd4275" )
#  ENDIF (MSVC)
ENDIF (WIN32)

SET( Boost_ADDITIONAL_VERSIONS "1.40" "1.47" "1.47.0")
FIND_PACKAGE( Boost 1.40.0 COMPONENTS filesystem regex program_options system thread date_time serialization)

IF(Boost_THREAD_FOUND)
  OPTION( BUILD_BOOST "Build Boost" OFF)
ELSE()
  OPTION( BUILD_BOOST "Build Boost" ON)
ENDIF()


FIND_PACKAGE( SWIG 1.3.40 )
MARK_AS_ADVANCED(  
  SWIG_DIR
  SWIG_EXECUTABLE
  SWIG_VERSION
)

IF(SWIG_FOUND AND NOT BUILD_V8_BINDINGS)
  OPTION(BUILD_SWIG "Build SWIG" OFF)
ELSE()
  OPTION(BUILD_SWIG "Build SWIG" ON)
ENDIF()

FIND_PROGRAM( RUBY_EXECUTABLE NAMES ruby PATHS $ENV{CMAKE_RUBY_PATH}  NO_SYSTEM_ENVIRONMENT_PATH )
FIND_PACKAGE( Ruby )


IF(WIN32 AND MSVC AND CMAKE_CL_64)
  MESSAGE(WARNING "64bit Support on Windows is EXPERIMENTAL. Ruby 2.0.0 will be built. The Sketchup plugin will not work. Expect problems with ruby support in general. Unless you've already built Boost and Qt you probably want to let this super build do it. There are lots of warnings, so we are disabling warnings as errors.") 
ENDIF()

IF (UNIX)
  IF(RUBY_FOUND)
    OPTION(BUILD_RUBY "Build Ruby" OFF)
  ELSE()
    OPTION(BUILD_RUBY "Build Ruby" ON)
  ENDIF()
ELSE()
  # Always provide ruby on Windows
  OPTION(BUILD_RUBY "Build Ruby" ON)
  MARK_AS_ADVANCED(BUILD_RUBY)
  
  IF(WIN32 AND MSVC)
    IF(CMAKE_CL_64)
      MESSAGE(WARNING "Cannot use 32bit mingw build of Ruby 2.0 with 64bit build of OpenStudio, forcing 64bit msvc build of Ruby 2.0") 
    ELSE()
      OPTION(BUILD_RUBY_MINGW32_2_0 "Use mingw32 Ruby 2.0" OFF)
      MARK_AS_ADVANCED(BUILD_RUBY_MINGW32_2_0)
    ENDIF()
  ENDIF() 
ENDIF()

IF ( NOT BUILD_RUBY AND NOT BUILD_SWIG )
  SET(RUBY_VER "${RUBY_VERSION_MAJOR}.${RUBY_VERSION_MINOR}.${RUBY_VERSION_PATCH}")

  IF(${RUBY_VER} VERSION_GREATER "1.8.8"
      AND ${SWIG_VERSION} VERSION_LESS "2.0.10")
    MESSAGE(SEND_ERROR "For Ruby >= 1.9.0, SWIG >= 2.0.10 is required. Use of the BUILD_SWIG option is suggested")
  ENDIF()
ENDIF()

IF(BUILD_WITH_MULTIPLE_PROCESSES)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
ENDIF()

IF(BUILD_NODE)
  # OSX can work with a static ruby library
  ExternalProject_Add(Node
    URL http://nodejs.org/dist/v0.10.15/node-v0.10.15.tar.gz
    CONFIGURE_COMMAND cd ${CMAKE_BINARY_DIR}/Node-prefix/src/Node && sh -c "./configure --prefix=${CMAKE_BINARY_DIR}/Node-prefix/src/Node-install"
    BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/Node-prefix/src/Node && $(MAKE)
    INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/Node-prefix/src/Node && $(MAKE) install && $(MAKE) install-includes && ../Node-install/bin/npm install jasmine-node node-gyp -g && ../Node-install/bin/node ../Node-install/bin/node-gyp install
    )
  SET(NODE_BIN_DIR ${CMAKE_BINARY_DIR}/Node-prefix/src/Node-install/bin/)
  SET(NODE_INCLUDE_DIR "$ENV{HOME}/.node-gyp/0.10.15")
ENDIF()

IF(BUILD_V8)
  ExternalProject_Add(V8
    SVN_REPOSITORY http://v8.googlecode.com/svn/branches/3.19
    CONFIGURE_COMMAND cd ${CMAKE_BINARY_DIR}/V8-prefix/src/V8 && $(MAKE) dependencies
    BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/V8-prefix/src/V8 && $(MAKE) x64
    INSTALL_COMMAND ""
    )
  SET(V8_INCLUDE_DIR ${CMAKE_BINARY_DIR}/V8-prefix/src/V8/include)
ENDIF()

IF(BUILD_RUBY)
  IF (APPLE)
    # OSX can work with a static ruby library
    ExternalProject_Add(Ruby 
      URL http://developer.nrel.gov/downloads/buildings/openstudio/src/ruby-1.8.7-p370.tar.gz
      URL_MD5 98b00bbd1cdde3116155edb6e555b781
      CONFIGURE_COMMAND cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby && sh -c "./configure --prefix=${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-install --without-tcl --without-tk  CFLAGS=\"-arch i386 -arch x86_64 -fPIC\" LDFLAGS=\"-arch i386 -arch x86_64\""
      BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby && $(MAKE)
      INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby && $(MAKE) install
      )
    SET(RUBY_EXECUTABLE ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-install/bin/ruby)
  ELSEIF (UNIX)
    # but linux cannot, it has a double free when unloading a ruby module
    ExternalProject_Add(Ruby 
      URL http://developer.nrel.gov/downloads/buildings/openstudio/src/ruby-1.8.7-p370.tar.gz
      URL_MD5 98b00bbd1cdde3116155edb6e555b781
      CONFIGURE_COMMAND cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby && sh -c "./configure --enable-shared --prefix=${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-install CFLAGS=\"-fPIC\" "
      BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby && $(MAKE)
      INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby && $(MAKE) install
      )
    SET(RUBY_EXECUTABLE ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-install/bin/ruby)
  ELSE()
    IF(CMAKE_CL_64)
      # Ruby 1.8 cannot be built for 64bit windows because of faulty assumptions in the code,
	  # and Ruby 2.0 isn't shipped in any way we can link to it, so for now we build our own
      ExternalProject_Add(Ruby
        URL http://developer.nrel.gov/downloads/buildings/openstudio/src/ruby-2.0.0-p0.zip
        URL_MD5 db5af5d6034646ad194cbdf6e50f49ee
        PATCH_COMMAND "${CMAKE_COMMAND}" -E copy ${CMAKE_SOURCE_DIR}/dependencies/mkexports.rb ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/win32/mkexports.rb
        CONFIGURE_COMMAND cmd /C "cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-build && ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/win32/configure.bat --prefix=${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-install --target=x64-mswin64 --disable-install-doc --disable-win95"
        BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-build && nmake
        INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-build && nmake install
        )
      SET(RUBY_EXECUTABLE ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-install/bin/ruby.exe )
      SET(RUBY_DLL x64-msvcr100-ruby200.dll)
      SET(RUBY_DLLPATH "${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-install/bin/${RUBY_DLL}")
      SET(RUBY_LIBRARY "${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby-install/lib/x64-msvcr100-ruby200.lib")
    ELSE()
      IF (BUILD_RUBY_MINGW32_2_0)
        # And windows requires mingw to build ruby, so we just unpackage it locally
        ExternalProject_Add(Ruby
          URL http://developer.nrel.gov/downloads/buildings/openstudio/src/ruby-2.0.0-aws-win32.zip
          PATCH_COMMAND "${CMAKE_COMMAND}" -E copy ${CMAKE_SOURCE_DIR}/dependencies/ruby/mingw32-ruby2.0/msvcrt-ruby200.lib  ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/lib COMMAND "${CMAKE_COMMAND}" -E copy ${CMAKE_SOURCE_DIR}/dependencies/ruby/mingw32-ruby2.0/config.h ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/include/ruby-2.0.0/i386-mingw32/ruby COMMAND "${CMAKE_COMMAND}" -E copy ${CMAKE_SOURCE_DIR}/dependencies/ruby/mingw32-ruby2.0/missing.h ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/include/ruby-2.0.0/ruby 
          CONFIGURE_COMMAND ""
          BUILD_COMMAND ""
          INSTALL_COMMAND ""
          )
        SET(RUBY_EXECUTABLE ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/bin/ruby.exe )
        SET(RUBY_DLL msvcrt-ruby200.dll)
        SET(RUBY_DLLPATH "${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/bin/${RUBY_DLL}")
        SET(RUBY_LIBRARY "${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/lib/msvcrt-ruby200.lib")
      ELSE()
        # And windows requires mingw to build ruby, so we just unpackage it locally
        ExternalProject_Add(Ruby
          URL http://developer.nrel.gov/downloads/buildings/openstudio/src/ruby-1.8.6-msvc-ssl.zip
          URL_MD5 d3fdb49726e63f123941f5f41f594149
          CONFIGURE_COMMAND ""
          BUILD_COMMAND ""
          INSTALL_COMMAND ""
          )
        SET(RUBY_EXECUTABLE ${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/bin/ruby.exe )
        SET(RUBY_DLL msvcrt-ruby18.dll)
        SET(RUBY_DLLPATH "${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/bin/${RUBY_DLL}")
        SET(RUBY_LIBRARY "${CMAKE_BINARY_DIR}/Ruby-prefix/src/Ruby/lib/msvcrt-ruby18.lib") 
      ENDIF()
    ENDIF()
    
    
  ENDIF()
ENDIF() 


IF(BUILD_SWIG)
  IF (UNIX)
    IF (BUILD_V8_BINDINGS)
      MESSAGE("Building SWIG for V8, the latest source will be cloned from git://github.com/lefticus/swig-v8.git. Autoconf tools are required")

      FIND_PROGRAM(autoconf "autoconf")

      IF (autoconf STREQUAL "autoconf_NOTFOUND")
        MESSAGE(FATAL_ERROR "autoconf is required to compile SWIG with V8 support")
      ELSE()
        GET_FILENAME_COMPONENT(autoconf_path "${autoconf}" PATH)
        MESSAGE("autoconf directory: ${autoconf_path}")
      ENDIF()



      # We patch it up with a version of pcre we provide to avoid having to have the requirement locally
      ExternalProject_Add( SWIG
        GIT_REPOSITORY git://github.com/lefticus/swig-v8.git
        GIT_TAG devel
        # URL http://github.com/lefticus/swig-v8/archive/devel.zip
        PATCH_COMMAND cp ${CMAKE_SOURCE_DIR}/dependencies/pcre-8.31.tar.bz2 ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-build && cd ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-build && ../SWIG/Tools/pcre-build.sh
        CONFIGURE_COMMAND cd ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG && /bin/sh -c "PATH=\$PATH:${autoconf_path} ./autogen.sh" && cd - && ../SWIG/configure --prefix=${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-install --disable-ccache
        BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-build && $(MAKE)
        INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-build && $(MAKE) install
        )
      SET(SWIG_EXECUTABLE ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-install/bin/swig )
    ELSE()
      # We patch it up with a version of pcre we provide to avoid having to have the requirement locally
      ExternalProject_Add( SWIG
        URL http://developer.nrel.gov/downloads/buildings/openstudio/src/swig-2.0.10.tar.gz
        URL_MD5 6d5e7ad05b4a404e5e85db9befb70c9a
        PATCH_COMMAND cp ${CMAKE_SOURCE_DIR}/dependencies/pcre-8.31.tar.bz2 ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-build && cd ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-build && ../SWIG/Tools/pcre-build.sh
        CONFIGURE_COMMAND ../SWIG/configure --prefix=${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-install
        BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-build && $(MAKE)
        INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-build && $(MAKE) install
        )
      SET(SWIG_EXECUTABLE ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG-install/bin/swig )
    ENDIF()
  ELSE()
    # SWIG requires MinGW to compile on windows, so we just copy in the prebuilt binary
    ExternalProject_Add( SWIG
      URL http://developer.nrel.gov/downloads/buildings/openstudio/src/swigwin-2.0.10.zip
      URL_MD5 cbb7006ecc912f056a2ec7f322fe59fb
      CONFIGURE_COMMAND ""
      BUILD_COMMAND ""
      INSTALL_COMMAND ""
      )
    SET(SWIG_EXECUTABLE ${CMAKE_BINARY_DIR}/SWIG-prefix/src/SWIG/swig.exe )
  ENDIF()
ENDIF() 

IF(BUILD_BOOST)
  IF(APPLE)
    STRING(REGEX MATCH "[0-9]+\\.[0-9]+" OSX_SDK_VERSION ${CMAKE_OSX_SYSROOT})
    # Added cxxflags="-stdlib=libstdc++" linkflags="-stdlib=libstdc++" for Boost 1.47.0 to compile when osx_deployment_target=10.9
    # These flags can be removed if Boost is updated to newer version (~1.55.0), may need --c++11 flag?
    ExternalProject_Add( Boost
      URL http://developer.nrel.gov/downloads/buildings/openstudio/src/boost_1_47_0.tar.gz
      URL_MD5 b0d9b288627b6a57d41d425d4f467592
      CONFIGURE_COMMAND cd ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost && ./bootstrap.sh
      PATCH_COMMAND patch -p1 < ${CMAKE_SOURCE_DIR}/dependencies/Boost/xcode_43.diff
      BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost && ./b2 cxxflags="-stdlib=libstdc++" linkflags="-stdlib=libstdc++" variant=release variant=debug address-model=32_64 macosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET} macosx-version=${OSX_SDK_VERSION}  architecture=x86 --layout=tagged  --without-python --without-math --prefix=${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost-install -j${CPUCOUNT} install
      INSTALL_COMMAND ""
      )
    SET(BOOST_ROOT ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost-install)
  ELSEIF(UNIX)
    ExternalProject_Add( Boost
      URL http://developer.nrel.gov/downloads/buildings/openstudio/src/boost_1_47_0.tar.gz
      URL_MD5 b0d9b288627b6a57d41d425d4f467592
      CONFIGURE_COMMAND cd ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost && ./bootstrap.sh
      BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost && ./b2 cxxflags=-fPIC link=static variant=debug variant=release --layout=tagged --without-python --without-math --prefix=${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost-install -j${CPUCOUNT} install
      INSTALL_COMMAND ""
      )
    SET(BOOST_ROOT ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost-install)
  ELSE()
    # The call to cmd and VS_UNICODE_OUTPUT is to fix an oddity where boost configuration complains about missing ICU when it shouldn't 
    # be trying to find it

    IF(CMAKE_CL_64)
      ExternalProject_Add( Boost
        URL http://developer.nrel.gov/downloads/buildings/openstudio/src/boost_1_47_0.tar.gz
        URL_MD5 b0d9b288627b6a57d41d425d4f467592
        PATCH_COMMAND cmd /C "cd ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost && bootstrap.bat"
        CONFIGURE_COMMAND ""
        BUILD_COMMAND cmd /C "set VS_UNICODE_OUTPUT=& cd ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost && b2 link=static address-model=64 --build-type=minimal  --without-python  --without-math --prefix=${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost-install -j${CPUCOUNT}" install
        INSTALL_COMMAND ""
        )
    ELSE()
      ExternalProject_Add( Boost
        URL http://developer.nrel.gov/downloads/buildings/openstudio/src/boost_1_47_0.tar.gz
        URL_MD5 b0d9b288627b6a57d41d425d4f467592
        PATCH_COMMAND cmd /C "cd ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost && bootstrap.bat"
        CONFIGURE_COMMAND ""
        BUILD_COMMAND cmd /C "set VS_UNICODE_OUTPUT=& cd ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost && b2 link=static address-model=32 --build-type=minimal  --without-python  --without-math --prefix=${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost-install -j${CPUCOUNT}" install
        INSTALL_COMMAND ""
        )
    ENDIF()

    SET(BOOST_ROOT ${CMAKE_BINARY_DIR}/Boost-prefix/src/Boost-install)
  ENDIF()
ENDIF()


IF(BUILD_BOOST)
  list(APPEND BoostLog_DEPENDS Boost)
ENDIF()

ExternalProject_Add( BoostLog
  DEPENDS ${BoostLog_DEPENDS}
  URL ${CMAKE_SOURCE_DIR}/dependencies/boost-log-1.1.tar.gz
  URL_MD5 ae76f0c29ca42ba0dd889a9932c44999
  PATCH_COMMAND "${CMAKE_COMMAND}" -E copy_directory ${CMAKE_SOURCE_DIR}/dependencies/BoostLog ${CMAKE_BINARY_DIR}/BoostLog-prefix/src/BoostLog
  INSTALL_COMMAND ""
  CMAKE_CACHE_ARGS
  -DBOOST_ROOT:STRING=${BOOST_ROOT}
  -DBOOST_LIBRARYDIR:STRING=${BOOST_ROOT}/lib
  -DCMAKE_OSX_ARCHITECTURES:STRING=${CMAKE_OSX_ARCHITECTURES}
  -DCMAKE_OSX_DEPLOYMENT_TARGET:STRING=${CMAKE_OSX_DEPLOYMENT_TARGET}
  -DCMAKE_OSX_SYSROOT:STRING=${CMAKE_OSX_SYSROOT}
  -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
  -DCMAKE_CXX_COMPILER:STRING=${CMAKE_CXX_COMPILER}
  -DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS}
  -DCMAKE_EXE_LINKER_FLAGS:STRING=${CMAKE_EXE_LINKER_FLAGS}
  -DCMAKE_MODULE_LINKER_FLAGS:STRING=${CMAKE_MODULE_LINKER_FLAGS}
  -DCMAKE_SHARED_LINKER_FLAGS:STRING=${CMAKE_SHARED_LINKER_FLAGS}
   )

ExternalProject_Add( GoogleTest
  URL ${CMAKE_SOURCE_DIR}/dependencies/gtest-1.6.0.tar.gz
  URL_MD5 69035f444dfd565466a7d22f67cafecd
  PATCH_COMMAND "${CMAKE_COMMAND}" -E copy_directory ${CMAKE_SOURCE_DIR}/dependencies/GoogleTest ${CMAKE_BINARY_DIR}/GoogleTest-prefix/src/GoogleTest
  INSTALL_COMMAND ""
  CMAKE_CACHE_ARGS
  -DCMAKE_OSX_ARCHITECTURES:STRING=${CMAKE_OSX_ARCHITECTURES}
  -DCMAKE_OSX_DEPLOYMENT_TARGET:STRING=${CMAKE_OSX_DEPLOYMENT_TARGET}
  -DCMAKE_OSX_SYSROOT:STRING=${CMAKE_OSX_SYSROOT}
  -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
  -DCMAKE_CXX_COMPILER:STRING=${CMAKE_CXX_COMPILER}
  -DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS}
  -DCMAKE_EXE_LINKER_FLAGS:STRING=${CMAKE_EXE_LINKER_FLAGS}
  -DCMAKE_MODULE_LINKER_FLAGS:STRING=${CMAKE_MODULE_LINKER_FLAGS}
  -DCMAKE_SHARED_LINKER_FLAGS:STRING=${CMAKE_SHARED_LINKER_FLAGS}
   )

SET(GTEST_INCLUDE_DIR ${CMAKE_BINARY_DIR}/GoogleTest-prefix/src/GoogleTest/include)
SET(GTEST_LIB_DIR ${CMAKE_BINARY_DIR}/GoogleTest-prefix/src/GoogleTest-build)

IF(BUILD_QT)
  IF(APPLE)
    # We're going to explicitly use the SDK specified in the configuration by the user
    SET(APPLE_SDK_COMMAND -sdk ${CMAKE_OSX_SYSROOT} )

    IF (CMAKE_OSX_SYSROOT MATCHES ".*10.7.sdk")
      SET(APPLE_PATCH_COMMAND "${CMAKE_COMMAND}" -E rename "${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt/src/3rdparty/webkit/WebKitLibraries/libWebKitSystemInterfaceMountainLion.a" "${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt/src/3rdparty/webkit/WebKitLibraries/libWebKitSystemInterfaceMountainLion.a.bak" COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt/src/3rdparty/webkit/WebKitLibraries/libWebKitSystemInterfaceLion.a" "${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt/src/3rdparty/webkit/WebKitLibraries/libWebKitSystemInterfaceMountainLion.a")
    ELSE()
      SET(APPLE_PATCH_COMMAND "")
    ENDIF()

    # At the moment we are building all of debug,release,x86 and x86_64
    # it would appear that calling "make" in some way that CMake can detect that we are calling
    # make (and attempt to pass parameters to the child make jobs) is breaking our ability to install
    # the below code circumvents that by calling make through a call to "sh"
    ExternalProject_Add( Qt
      URL http://developer.nrel.gov/downloads/buildings/openstudio/src/qt-everywhere-opensource-src-4.8.5.tar.gz
      URL_MD5 1864987bdbb2f58f8ae8b350dfdbe133
      CONFIGURE_COMMAND ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt/configure  ${APPLE_SDK_COMMAND} -debug-and-release -opensource -openssl -arch x86 -arch x86_64 -qt-sql-sqlite -plugin-sql-sqlite -nomake examples -nomake demos -nomake docs -no-phonon -no-phonon-backend -no-qt3support -confirm-license -prefix ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-install
      PATCH_COMMAND ${APPLE_PATCH_COMMAND}
      BUILD_COMMAND $(MAKE) -f ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-build/Makefile      
      INSTALL_COMMAND /bin/sh -c "cd ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-build && make -j1 install"
      )
  ELSEIF(UNIX)
    ExternalProject_Add( Qt
      URL http://developer.nrel.gov/downloads/buildings/openstudio/src/qt-everywhere-opensource-src-4.8.5.tar.gz
      URL_MD5 1864987bdbb2f58f8ae8b350dfdbe133
      CONFIGURE_COMMAND ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt/configure -debug-and-release -opensource -confirm-license -shared -qt-sql-sqlite -no-qt3support -openssl -nomake examples -nomake demos -nomake docs -plugindir ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-install/share/openstudio/qtplugins -qtlibinfix OpenStudio -prefix ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-install
      BUILD_COMMAND $(MAKE) -f ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-build/Makefile
      INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-build && $(MAKE) install
      )
  ELSE()

    ExternalProject_Add( Qt
      URL http://developer.nrel.gov/downloads/buildings/openstudio/src/qt-everywhere-opensource-src-4.8.5.tar.gz
      URL_MD5 1864987bdbb2f58f8ae8b350dfdbe133
      CONFIGURE_COMMAND cd ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt && configure -debug-and-release -opensource -confirm-license -shared -plugin-sql-sqlite -no-qt3support -nomake examples -nomake docs -nomake demos -mp -prefix ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-install -L ${OPENSSL_ROOT_DIR}/lib -I ${OPENSSL_ROOT_DIR}/include -openssl
      BUILD_COMMAND cd ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt && nmake 
      INSTALL_COMMAND cd ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt && nmake install
      )
  ENDIF()

  SET(QT_QMAKE_EXECUTABLE ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-install/bin/qmake)
ENDIF()


IF(BUILD_QT)
  list(APPEND QJSON_DEPENDS Qt)
ENDIF()

ExternalProject_Add(QJSON
  DEPENDS ${QJSON_DEPENDS}
  URL http://developer.nrel.gov/downloads/buildings/openstudio/src/qjson-0.8.1.tar.gz
  URL_MD5 4eef13da988edf8f91c260a3e1baeea9
  PATCH_COMMAND "${CMAKE_COMMAND}" -E copy ${CMAKE_SOURCE_DIR}/dependencies/QJSON/CMakeLists.txt  ${CMAKE_BINARY_DIR}/QJSON-prefix/src/QJSON/src COMMAND "${CMAKE_COMMAND}" -E copy ${CMAKE_SOURCE_DIR}/dependencies/QJSON/QJSONConfig.cmake.in ${CMAKE_BINARY_DIR}/QJSON-prefix/src/QJSON COMMAND "${CMAKE_COMMAND}" -E copy ${CMAKE_SOURCE_DIR}/dependencies/QJSON/qjson_export.h ${CMAKE_BINARY_DIR}/QJSON-prefix/src/QJSON/src

  CMAKE_CACHE_ARGS
  -DCMAKE_OSX_ARCHITECTURES:STRING=${CMAKE_OSX_ARCHITECTURES}
  -DCMAKE_OSX_DEPLOYMENT_TARGET:STRING=${CMAKE_OSX_DEPLOYMENT_TARGET}
  -DCMAKE_OSX_SYSROOT:STRING=${CMAKE_OSX_SYSROOT}
  -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
  -DCMAKE_CXX_COMPILER:STRING=${CMAKE_CXX_COMPILER}
  -DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS}
  -DCMAKE_EXE_LINKER_FLAGS:STRING=${CMAKE_EXE_LINKER_FLAGS}
  -DCMAKE_MODULE_LINKER_FLAGS:STRING=${CMAKE_MODULE_LINKER_FLAGS}
  -DCMAKE_SHARED_LINKER_FLAGS:STRING=${CMAKE_SHARED_LINKER_FLAGS}
  -DQT_QMAKE_EXECUTABLE:STRING=${QT_QMAKE_EXECUTABLE}
  -DCMAKE_INSTALL_PREFIX:STRING=${CMAKE_BINARY_DIR}/QJSON-prefix/src/QJSON-install
)


set(OpenStudioCore_DIR ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build)

set(BoostLog_DIR "${CMAKE_BINARY_DIR}/BoostLog-prefix/src/BoostLog-build")
set(QJSON_DIR "${CMAKE_BINARY_DIR}/QJSON-prefix/src/QJSON-build")

# Set up the dependencies for OpenStudioCore
SET(OpenStudioCore_DEPENDS BoostLog)
LIST(APPEND OpenStudioCore_DEPENDS GoogleTest)
LIST(APPEND OpenStudioCore_DEPENDS QJSON)

IF(BUILD_QT)
  LIST(APPEND OpenStudioCore_DEPENDS Qt)
ENDIF()

IF(BUILD_SWIG)
  LIST(APPEND OpenStudioCore_DEPENDS SWIG)
ENDIF()

IF(BUILD_BOOST)
  LIST(APPEND OpenStudioCore_DEPENDS Boost)
ENDIF()

IF(BUILD_RUBY)
  LIST(APPEND OpenStudioCore_DEPENDS Ruby)
ENDIF()

IF(BUILD_NODE)
  LIST(APPEND OpenStudioCore_DEPENDS Node)
ENDIF()

IF(BUILD_V8)
  LIST(APPEND OpenStudioCore_DEPENDS V8)
ENDIF()


ExternalProject_Add( OpenStudioCore
  DEPENDS ${OpenStudioCore_DEPENDS}
  SOURCE_DIR ${CMAKE_SOURCE_DIR}/openstudiocore
  CMAKE_CACHE_ARGS
    -DGTEST_LIB_DIR:PATH=${GTEST_LIB_DIR}
    -DGTEST_INCLUDE_DIR:PATH=${GTEST_INCLUDE_DIR}
    -DCMAKE_OSX_ARCHITECTURES:STRING=${CMAKE_OSX_ARCHITECTURES}
    -DCMAKE_OSX_DEPLOYMENT_TARGET:STRING=${CMAKE_OSX_DEPLOYMENT_TARGET}
    -DCMAKE_OSX_SYSROOT:STRING=${CMAKE_OSX_SYSROOT}
    -DENERGYPLUS_VERSION:STRING=${ENERGYPLUS_VERSION}
    -DBUILD_DOCUMENTATION:BOOL=${BUILD_DOCUMENTATION}
    -DBUILD_CSHARP_BINDINGS:BOOL=${BUILD_CSHARP_BINDINGS}
    -DBUILD_V8_BINDINGS:BOOL=${BUILD_V8_BINDINGS}
    -DBUILD_NODE_MODULES:BOOL=${BUILD_NODE_MODULES}
    -DBUILD_PYTHON_BINDINGS:BOOL=${BUILD_PYTHON_BINDINGS}
    -DBUILD_TESTING:BOOL=${BUILD_TESTING}
    -DBUILD_PACKAGE:BOOL=${BUILD_PACKAGE}
    -DENABLE_TEST_RUNNER_TARGETS:BOOL=${ENABLE_TEST_RUNNER_TARGETS}
    -DBUILD_WITH_MULTIPLE_PROCESSES:BOOL=${BUILD_WITH_MULTIPLE_PROCESSES}
    -DBUILD_WITH_OPENSSL:BOOL=${BUILD_WITH_OPENSSL}
    -DBUILD_RUBY_GEM:BOOL=${BUILD_RUBY_GEM}
    -DBUILD_RUBY_GEM_WITH_LIBS:BOOL=${BUILD_RUBY_GEM_WITH_LIBS}
    -DREQUIRE_DAKOTA:BOOL=${REQUIRE_DAKOTA}
    -DMSVC_IS_EXPRESS:BOOL=${MSVC_IS_EXPRESS}
    -DUSE_PCH:BOOL=${USE_PCH}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_CXX_COMPILER:STRING=${CMAKE_CXX_COMPILER}
    -DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS}
    -DCMAKE_C_FLAGS:STRING=${CMAKE_C_FLAGS}
    -DCMAKE_EXE_LINKER_FLAGS:STRING=${CMAKE_EXE_LINKER_FLAGS}
    -DCMAKE_MODULE_LINKER_FLAGS:STRING=${CMAKE_MODULE_LINKER_FLAGS}
    -DCMAKE_SHARED_LINKER_FLAGS:STRING=${CMAKE_SHARED_LINKER_FLAGS}
    -DCMAKE_INSTALL_PREFIX:STRING=${CMAKE_INSTALL_PREFIX}
    -DCMAKE_VERSION_MAJOR:STRING=${CMAKE_VERSION_MAJOR}
    -DCMAKE_VERSION_MINOR:STRING=${CMAKE_VERSION_MINOR}
    -DCMAKE_VERSION_PATCH:STRING=${CMAKE_VERSION_PATCH}
    -DCMAKE_VERSION_BUILD:STRING=${CMAKE_VERSION_BUILD}
    -DSUBVERSION_REVISION:STRING=${SUBVERSION_REVISION}
    -DBOOST_ROOT:STRING=${BOOST_ROOT}
    -DBOOST_LIBRARYDIR:STRING=${BOOST_ROOT}/lib
    -DQT_QMAKE_EXECUTABLE:STRING=${QT_QMAKE_EXECUTABLE}
    -DSWIG_EXECUTABLE:STRING=${SWIG_EXECUTABLE}
    -DRUBY_EXECUTABLE:STRING=${RUBY_EXECUTABLE}
    -DNODE_BIN_DIR:STRING=${NODE_BIN_DIR}
    -DNODE_INCLUDE_DIR:STRING=${NODE_INCLUDE_DIR}
    -DV8_INCLUDE_DIR:STRING=${V8_INCLUDE_DIR}
    -DRUBY_LIBRARY:STRING=${RUBY_LIBRARY}
    -DDOXYGEN_EXECUTABLE:STRING=${DOXYGEN_EXECUTABLE}
    -DDOXYGEN_DOT_EXECUTABLE:STRING=${DOXYGEN_DOT_EXECUTABLE}
    -DBoostLog_DIR:STRING=${BoostLog_DIR}
    -DQJSON_DIR:STRING=${QJSON_DIR}
    -DPACKAGE_RADIANCE_AND_ENERGYPLUS:BOOL=${PACKAGE_RADIANCE_AND_ENERGYPLUS}
    -DENERGYPLUS_LOCATION:PATH=${ENERGYPLUS_LOCATION}
    -DRADIANCE_LOCATION:PATH=${RADIANCE_LOCATION}
    -DOPENSSL_ROOT_DIR:PATH=${OPENSSL_ROOT_DIR}
    INSTALL_COMMAND ""
)

if (BUILD_QT AND MSVC)
  # Install Qt DLL's to Release directory. This is brute force-ish, it copies all files in the bin folder
  ExternalProject_Add_Step(OpenStudioCore InstallQtReleaseDLLs
     COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-install/bin ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/Release
     DEPENDERS configure
   )     
   
  # Install Qt DLL's to Debug directory. This is brute force-ish, it copies all files in the bin folder
  ExternalProject_Add_Step(OpenStudioCore InstallQtDebugDLLs
     COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_BINARY_DIR}/Qt-prefix/src/Qt-install/bin ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/Debug
     DEPENDERS configure
   )     
endif()

if (BUILD_RUBY AND MSVC)
  ExternalProject_Add_Step(OpenStudioCore MakeRubyReleaseFolder
     COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/Release
     DEPENDERS configure
   )     
  ExternalProject_Add_Step(OpenStudioCore MakeRubyDebugFolder
   COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/Debug
     DEPENDERS configure
   )     
  ExternalProject_Add_Step(OpenStudioCore MakeRubyRelWithDebInfoFolder
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/RelWithDebInfo
     DEPENDERS configure
   )     
  ExternalProject_Add_Step(OpenStudioCore MakeRubyMinSizeRelFolder
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/MinSizeRel
     DEPENDERS configure
   )     

   
  ExternalProject_Add_Step(OpenStudioCore MakeRubyReleaseDLL
    COMMAND ${CMAKE_COMMAND} -E copy "${RUBY_DLLPATH}" "${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/Release/${RUBY_DLL}"
     DEPENDERS configure
   )     
  ExternalProject_Add_Step(OpenStudioCore MakeRubyDebugDLL
    COMMAND ${CMAKE_COMMAND} -E copy "${RUBY_DLLPATH}" "${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/Debug/${RUBY_DLL}"
     DEPENDERS configure
   )     
  ExternalProject_Add_Step(OpenStudioCore MakeRubyRelWithDebInfoDLL
    COMMAND ${CMAKE_COMMAND} -E copy "${RUBY_DLLPATH}" "${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/RelWithDebInfo/${RUBY_DLL}"
     DEPENDERS configure
   )     
  ExternalProject_Add_Step(OpenStudioCore MakeRubyMinSizeRelDLL
    COMMAND ${CMAKE_COMMAND} -E copy "${RUBY_DLLPATH}" "${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products/MinSizeRel/${RUBY_DLL}"
     DEPENDERS configure
   )     
      
endif()


set(OPENSTUDIOCORE_BUILD_DIR ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build)
set(OPENSTUDIOCORE_LIB_DIR ${CMAKE_BINARY_DIR}/OpenStudioCore-prefix/src/OpenStudioCore-build/Products)
set(OPENSTUDIOCORE_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/openstudiocore/src)
set(OPENSTUDIOCORE_ROOT_DIR ${CMAKE_SOURCE_DIR}/openstudiocore/)

###############################################################################
# Use CPack
IF( BUILD_PACKAGE )
  INCLUDE(OpenStudioCPack.cmake )
ENDIF( BUILD_PACKAGE )
###############################################################################


