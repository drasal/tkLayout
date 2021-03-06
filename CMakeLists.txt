#############################################################################
#
# CMAKE file for building tkLayout software
#
#  Author: Z. Drasal, CERN
#
#############################################################################

#----------------------------------------------------------------------------
# CMAKE compatibility issues
CMAKE_MINIMUM_REQUIRED(VERSION 2.6 FATAL_ERROR)

#----------------------------------------------------------------------------
# Project name and version
PROJECT(tklayout)

#set( ${PROJECT_NAME}_MAJOR_VERSION 1 )
#set( ${PROJECT_NAME}_MINOR_VERSION 0 )

#----------------------------------------------------------------------------
# Project options
OPTION( INSTALL_DOC "Set to OFF to skip build/install Documentation" ON )

#----------------------------------------------------------------------------
# Advanced settings
# Library *nix style versioning
SET( ${PROJECT_NAME}_SOVERSION
    "${${PROJECT_NAME}_MAJOR_VERSION}.${${PROJECT_NAME}_MINOR_VERSION}" )

# Output directories
SET( EXECUTABLE_OUTPUT_PATH "${PROJECT_BINARY_DIR}/bin" CACHE PATH
    "EXECUTABLE_OUTPUT_PATH" FORCE )
SET( LIBRARY_OUTPUT_PATH "${PROJECT_BINARY_DIR}/lib" CACHE PATH
    "LIBRARY_OUTPUT_PATH" FORCE )
MARK_AS_ADVANCED( EXECUTABLE_OUTPUT_PATH LIBRARY_OUTPUT_PATH )

# Set default install prefix to project root directory
IF( CMAKE_INSTALL_PREFIX STREQUAL "/usr/local" )
    SET( CMAKE_INSTALL_PREFIX "${CMAKE_SOURCE_DIR}" )
ENDIF()


#----------------------------------------------------------------------------
# Project dependencies

#
# Get boost include & lib directory
#
SET( BOOST_INCLUDE_DIR "$ENV{BOOST_INCLUDE}" )

# Find all the libraries and save them 
SET( BOOST_LIBS_NAMES "boost_system$ENV{BOOST_SUFFIX}" 
                      "boost_filesystem$ENV{BOOST_SUFFIX}" 
                      "boost_program_options$ENV{BOOST_SUFFIX}" )
SET( BOOST_LIBS_DIR "$ENV{BOOST_LIB}" )

FOREACH( libname ${BOOST_LIBS_NAMES} )

        SET( BOOST_LIB_${libname} BOOST_LIB_${libname}-NOTFOUND )
        MARK_AS_ADVANCED( BOOST_LIB_${libname} )
        # MESSAGE( STATUS "${libname} >> ${BOOST_LIBS_DIR}")
        FIND_LIBRARY( BOOST_LIB_${libname} 
            NAMES ${libname} 
            PATHS ${BOOST_LIBS_DIR}
            PATH_SUFFIXES lib )
        # MESSAGE( STATUS ">>> ${libname} >> ${BOOST_LIB_${libname}}" )
        IF( NOT BOOST_LIB_${libname} )
            SET( BOOST_FINDLIB_FAILED TRUE )
            IF( NOT BOOST_FIND_QUIETLY )
                MESSAGE( STATUS "Check for BOOST: ${BOOST_LIBS_DIR}" 
                                " -- failed to find BOOST ${libname} library!!" )
            ENDIF()
        ELSE()
            LIST( APPEND BOOST_LIBS ${BOOST_LIB_${libname}} )
        ENDIF()
    ENDFOREACH()

#
# Get include directory using root-config
#
SET( ROOT_HOME "$ENV{ROOTSYS}/bin" )
EXEC_PROGRAM( "${ROOT_HOME}/root-config" "."
    ARGS --incdir
    OUTPUT_VARIABLE out_var
    RETURN_VALUE out_ret )

# Check if everything went OK
IF( out_ret)
    IF( NOT ROOT_FIND_QUIETLY )
        MESSAGE( STATUS "Check for ROOT: ${ROOT_HOME}/root-config --incdir"
                        " -- failed to find ROOT include directory!!")
    ENDIF()
    SET( ROOT_FINDINCL_FAILED TRUE )
    MARK_AS_ADVANCED( ROOT_FINDINCL_FAILED )
ELSE()
    SET( ROOT_FINDINCL_FAILED FALSE )
    SET( ROOT_INCLUDE_DIRS ${out_var} )
    MARK_AS_ADVANCED( ROOT_FINDINCL_FAILED ROOT_INCLUDE_DIRS )
ENDIF()

#
# Get libraries using root-config
#
SET( ROOT_LIBS_NAMES )

EXEC_PROGRAM( "${ROOT_HOME}/root-config" "."
    ARGS --noauxlibs --evelibs
    OUTPUT_VARIABLE out_var
    RETURN_VALUE out_ret )

# Check if everything went OK 
IF( out_ret) 
    IF( NOT ROOT_FIND_QUIETLY ) 
        MESSAGE( STATUS "Check for ROOT: ${ROOT_HOME}/root-config --noauxlibs --evelibs" 
                        " -- failed to find ROOT libraries!!") 
    ENDIF()                      
    SET( ROOT_FINDLIBS_FAILED TRUE ) 
    MARK_AS_ADVANCED( ROOT_FINDLIB_FAILED ) 
ELSE() 
    # Each argument will be separated by semicolon 
    SEPARATE_ARGUMENTS( out_var ) 
     
    # Get library path and remove -L compiler flag 
    LIST( GET out_var 0 libdir ) 
    LIST( REMOVE_AT out_var 0 ) 
    STRING( REGEX REPLACE "^-.(.*)$" "\\1" ROOT_LIBS_DIR "${libdir}") 
    MARK_AS_ADVANCED( ROOT_LIBS_DIR ) 
     
    # Extract library names 
    FOREACH( lib ${out_var} ) 
        STRING( REGEX REPLACE "^-.(.*)$" "\\1" libname "${lib}") 
        LIST( APPEND ROOT_LIBS_NAMES ${libname} ) 
    ENDFOREACH() 
     
    # Find all the libraries and save them 
    FOREACH( libname ${ROOT_LIBS_NAMES} "HistPainter") 
     
        SET( ROOT_LIB_${libname} ROOT_LIB_${libname}-NOTFOUND ) 
        MARK_AS_ADVANCED( ROOT_LIB_${libname} ) 
 
        FIND_LIBRARY( ROOT_LIB_${libname} 
            NAMES ${libname} 
            PATHS ${ROOT_LIBS_DIR} 
            PATH_SUFFIXES lib ) 
 
        IF( NOT ROOT_LIB_${libname} ) 
            SET( ROOT_FINDLIB_FAILED TRUE ) 
            IF( NOT ROOT_FIND_QUIETLY ) 
                MESSAGE( STATUS "Check for ROOT: ${ROOT_HOME}/root-config" 
                                " -- failed to find ROOT ${libname} library!!" ) 
            ENDIF() 
        ELSE() 
            #MESSAGE(STATUS ">>> ${ROOT_LIB_${libname}}")
            LIST( APPEND ROOT_LIBS ${ROOT_LIB_${libname}} ) 
        ENDIF() 
    ENDFOREACH() 
ENDIF() 

#----------------------------------------------------------------------------
# Include, source, dirs 
#
# Compiler flags
#SET ( CMAKE_CXX_COMPILER "g++" )
ADD_DEFINITIONS( "-Wl,--copy-dt-needed-entries" )
SET ( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14 -g -pedantic -Wall -Wno-reorder")
#SET ( CMAKE_EXE_LINKER_FLAGS "-Wl,--copy-dt-needed-entries" )

INCLUDE_DIRECTORIES( ${PROJECT_SOURCE_DIR}/source/Geometry/include 
                     ${PROJECT_SOURCE_DIR}/source/Analysis/include
                     ${PROJECT_SOURCE_DIR}/source/Extractors/include
                     ${PROJECT_SOURCE_DIR}/source/TinyXML/include
                     ${PROJECT_SOURCE_DIR}/source/Tools/include
                     ${PROJECT_SOURCE_DIR}/source/Tracking/include
                     ${PROJECT_SOURCE_DIR}/source/Web/include
                     ${BOOST_INCLUDE_DIR}
                     ${ROOT_INCLUDE_DIRS} )
FILE( GLOB exe_sources ${PROJECT_SOURCE_DIR}/source/*.cc )
FILE( GLOB obj_sources ${PROJECT_SOURCE_DIR}/source/Geometry/src/*.cc
                       ${PROJECT_SOURCE_DIR}/source/Analysis/src/*.cc 
                       ${PROJECT_SOURCE_DIR}/source/Extractors/src/*.cc
                       ${PROJECT_SOURCE_DIR}/source/Tools/src/*.cc 
                       ${PROJECT_SOURCE_DIR}/source/Tracking/src/*.cc
                       ${PROJECT_SOURCE_DIR}/source/Web/src/*.cc )
FILE( GLOB obj_headers ${PROJECT_SOURCE_DIR}/source/Geometry/include/*.h
                       ${PROJECT_SOURCE_DIR}/source/Analysis/include/*.h
                       ${PROJECT_SOURCE_DIR}/source/Extractors/include/*.h 
                       ${PROJECT_SOURCE_DIR}/source/Tools/include/*.h  
                       ${PROJECT_SOURCE_DIR}/source/Tracking/include/*.h 
                       ${PROJECT_SOURCE_DIR}/source/Web/include/*.h ) 
FILE( GLOB tinyxml_sources ${PROJECT_SOURCE_DIR}/source/TinyXML/src/*.cc )

# Get source files with main function defined
SET ( sources "" )
FOREACH( file ${exe_sources} )
 IF ( ${file} MATCHES "tklayout.cc" ) 
  SET( source_tklayout ${file} )
 ENDIF()
 IF ( ${file} MATCHES "setup.cc" ) 
  SET( source_setup ${file} )
 ENDIF()
 IF ( ${file} MATCHES "delphize.cc" )
  SET( source_delphize ${file} )
 ENDIF()
ENDFOREACH()

# Get files on which setup or delphize depend
FOREACH( file ${obj_sources} )
 IF( ${file} MATCHES "MainConfigHandler.cc" )
  SET( source_mainhandler ${file} )
 ENDIF()
 IF( ${file} MATCHES "GraphVizCreator.cc" )
  SET( source_vizgraphcreator ${file} )
 ENDIF()
 IF( ${file} MATCHES "MessageLogger.cc" )
  SET( source_messagelogger ${file} )
 ENDIF()
 IF( ${file} MATCHES "string_functions.cc" )
  SET( source_stringfunctions ${file} )
 ENDIF()
  IF( ${file} MATCHES "StringConverter.cc" )
  SET( source_stringconverter ${file} )
 ENDIF()
ENDFOREACH()

#----------------------------------------------------------------------------
# Set revision
# a custom target that is always built
ADD_CUSTOM_TARGET(revisiontag ALL)

# creates svnversion.h using cmake script
ADD_CUSTOM_COMMAND(TARGET revisiontag COMMAND ${CMAKE_COMMAND}
   -DSOURCE_DIR=${CMAKE_CURRENT_SOURCE_DIR} 
   -P ${CMAKE_CURRENT_SOURCE_DIR}/GetGitVersion.cmake)

#----------------------------------------------------------------------------
# add the executable, and link it to the geant4 libraries
#
#SET(CMAKE_INSTALL_RPATH ${BOOST_LIBS_DIR} ${ROOT_LIBS_DIR})
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)            # Set automatically rpath for dynamic linking of external libraries
SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib") # Set automatically rpath for dynamic linking of internally created & installed libraries

ADD_EXECUTABLE(tklayout ${source_tklayout} ${obj_sources} ${obj_headers} )
ADD_EXECUTABLE(setup.bin ${source_setup} ${source_vizgraphcreator} ${source_messagelogger} ${source_mainhandler} ${source_stringfunctions} ${source_stringconverter} ${obj_headers} )
ADD_EXECUTABLE(delphize ${source_delphize} )

ADD_LIBRARY(tinyXML SHARED ${tinyxml_sources})

# explicitly say that the executable depends on custom target
ADD_DEPENDENCIES(tklayout revisiontag)

TARGET_LINK_LIBRARIES(tklayout ${BOOST_LIBS} ${ROOT_LIBS} tinyXML )
TARGET_LINK_LIBRARIES(setup.bin ${BOOST_LIBS} )
TARGET_LINK_LIBRARIES(delphize ${BOOST_LIBS} ${ROOT_LIBS} )

#----------------------------------------------------------------------------
# Install the executable to 'bin' directory under CMAKE_INSTALL_PREFIX
#
INSTALL(TARGETS tklayout  RUNTIME DESTINATION bin)
INSTALL(TARGETS setup.bin RUNTIME DESTINATION bin)
INSTALL(TARGETS delphize  RUNTIME DESTINATION bin)
INSTALL(TARGETS tinyXML   ARCHIVE DESTINATION lib
                          LIBRARY DESTINATION lib
                          RUNTIME DESTINATION bin)

IF(CMAKE_HOST_UNIX)
    
    # Set the default version of GitRevision.cc file
    INSTALL( CODE "MESSAGE(STATUS \"Setting GitRevision.cc to default\" )
                   EXECUTE_PROCESS(COMMAND bash -c \"if [ -e GitRevision.orig.cc ]; then rm -f ${PROJECT_SOURCE_DIR}/source/Tools/src/GitRevision.cc; mv GitRevision.orig.cc ${PROJECT_SOURCE_DIR}/source/Tools/src/GitRevision.cc; fi\")" )
    
    # Intall symlink
    INSTALL( CODE "MESSAGE(STATUS \"Creating symlink $ENV{HOME}/bin/tklayout to ${PROJECT_SOURCE_DIR}/bin/tklayout... \" )
                   EXECUTE_PROCESS(COMMAND \"${CMAKE_COMMAND}\" -E create_symlink ${PROJECT_SOURCE_DIR}/bin/tklayout $ENV{HOME}/bin/tklayout)" )

    INSTALL( CODE "MESSAGE(STATUS \"Creating symlink $ENV{HOME}/bin/delphize to ${PROJECT_SOURCE_DIR}/bin/delphize... \" )
                   EXECUTE_PROCESS(COMMAND \"${CMAKE_COMMAND}\" -E create_symlink ${PROJECT_SOURCE_DIR}/bin/delphize $ENV{HOME}/bin/delphize)" )

    # Configure tkLayout
    INSTALL( CODE "MESSAGE(STATUS \"Configuring tkLayout software...\" )
                   MESSAGE(STATUS \" Target directories setup:\" )
                   EXECUTE_PROCESS(COMMAND bash -c \"if [ -e ${PROJECT_SOURCE_DIR}/bin/setup.bin ]; then ${PROJECT_SOURCE_DIR}/bin/setup.bin --dirNames; fi\")" ) 
    INSTALL( CODE "MESSAGE(STATUS \" Installation status:\" )
                   EXECUTE_PROCESS(COMMAND bash -c \"if ! ${PROJECT_SOURCE_DIR}/bin/setup.bin --checkDir; then echo \\\"Problem during installation\\\"; else echo \\\"Installation successful\\\"; fi\")" )

ENDIF()

#----------------------------------------------------------------------------
# Documentation
#
# Find Doxygen
FIND_PACKAGE( Doxygen )
IF( DOXYGEN_FOUND )

    # Add doc target
    ADD_CUSTOM_TARGET( doc ) 
    ADD_CUSTOM_COMMAND( TARGET doc
        COMMAND "${CMAKE_COMMAND}" -E copy_if_different
                "${PROJECT_SOURCE_DIR}/README.md" "${PROJECT_SOURCE_DIR}/doc/README.md"    
        COMMAND "${DOXYGEN_EXECUTABLE}" ARGS "Doxyfile"
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/doc"
        COMMENT "Building API Documentation in ${PROJECT_SOURCE_DIR}/doc... "
        VERBATIM )

ELSE()
    MESSAGE( STATUS "Doxygen not found in your system!!" )
    IF( INSTALL_DOC )
        MESSAGE( STATUS "INSTALL_DOC forced to OFF" )
        SET( INSTALL_DOC OFF )
    ENDIF()
ENDIF()

# Install documentation
#IF( INSTALL_DOC )
#    # make sure doxygen is executed (make doc) before make install
#    INSTALL( CODE "EXEC_PROGRAM(${CMAKE_BUILD_TOOL} ${PROJECT_BINARY_DIR} ARGS doc)" )
#    # install documentation
#    INSTALL( DIRECTORY "${PROJECT_SOURCE_DIR}/doc"
#            DESTINATION .
#            PATTERN "*CVS*" EXCLUDE )
#ENDIF()


#----------------------------------------------------------------------------
# Uninstall 
#
CONFIGURE_FILE(
    "${CMAKE_CURRENT_SOURCE_DIR}/CMakeUninstall.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

ADD_CUSTOM_TARGET(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
    
