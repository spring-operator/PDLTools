# ======================================================================
# -- Greenplum Port
# ======================================================================

set(PORT "Greenplum")
string(TOUPPER ${PORT} PORT_UC)
string(TOLOWER ${PORT} PORT_LC)
set(PORT_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
get_filename_component(PORT_DIR_NAME "${PORT_SOURCE_DIR}" NAME)
set(PORT_DEPLOY_SCRIPT "${CMAKE_BINARY_DIR}/deploy/Component_${PORT}.cmake")

if(APPLE)
  # FIXME: This should be handled in a better way.
  # Greenplum is 32bit on the Mac.
  set(ADDITIONAL_GCC_FLAGS "-m32")
else(APPLE)
  unset(ADDITIONAL_GCC_FLAGS)
endif(APPLE)

list(APPEND CMAKE_MODULE_PATH
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
include(PostgreSQLUtils)
include(GreenplumUtils)

# ======================================================================
# 1. Specify files that will be compiled into the shared library, for
#    *all* versions of this port
# ======================================================================

include_directories("${CMAKE_CURRENT_SOURCE_DIR}")
set(PDLTOOLS_DBAL_SOURCES  ${PDLTOOLS_SOURCES})

add_sql_files(
    SQL_TARGET_FILES
    "./modules"
    "${CMAKE_CURRENT_BINARY_DIR}/modules"
)

add_custom_target(sqlFiles_${PORT_LC} ALL DEPENDS ${SQL_TARGET_FILES})


# ======================================================================
# 2. Install all SQL files
# ======================================================================

cpack_add_port_group_and_component_for_all_versions()

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/modules
  DESTINATION ports/${PORT_DIR_NAME}
  COMPONENT ${PORT_LC}_any
  REGEX "^(.*/)?\\.DS_Store\$" EXCLUDE)

# ======================================================================
# 3. Building the shared library is specific for each version of this
#    port
# ======================================================================

function(add_${PORT_LC}_library IN_PORT_VERSION)
  string(REPLACE "." "_" PORT_VERSION_UNDERSCORE "${IN_PORT_VERSION}")
  set(DBMS "${PORT_LC}_${PORT_VERSION_UNDERSCORE}")
  set(DBMS_UC "${PORT_UC}_${PORT_VERSION_UNDERSCORE}")

  # --------------------------------------------------------------------
  # 3.1. Compile C/C++ files
  # --------------------------------------------------------------------
  
  message ("   ------------------------------------------------------------")
  message ("   Adding ${PORT} to target list")
  message ("     ${DBMS_UC}_VERSION_MAJOR .. : ${${DBMS_UC}_VERSION_MAJOR}")
  message ("     ${DBMS_UC}_VERSION_MINOR .. : ${${DBMS_UC}_VERSION_MINOR}")
  message ("     ${DBMS_UC}_ARCHITECTURE ... : ${${DBMS_UC}_ARCHITECTURE}")
  message ("   ------------------------------------------------------------")
  
  add_executable(${DBMS} IMPORTED)
  set_target_properties(${DBMS} PROPERTIES IMPORTED_LOCATION "${${DBMS_UC}_EXECUTABLE}")
  
  include_directories(SYSTEM
    ${${DBMS_UC}_SERVER_INCLUDE_DIR}
    ${${DBMS_UC}_ADDITIONAL_INCLUDE_DIRS})
  
  add_pdltools_connector_library(pdltools_${DBMS}
    lib
    "${${DBMS_UC}_EXECUTABLE}"
    ${PDLTOOLS_DBAL_SOURCES})

  if(CMAKE_COMPILER_IS_GNUCXX)
    # The source code specifies that we are POSIX.1-2001 compliant:
    # On the Mac, this is done in $GP_HOME/include/pg_config_manual.h.
    # On Linux, gcc predefines _GNU_SOURCE, which leads to
    # _POSIX_C_SOURCE being defined in /usr/include/features.h
    # We therefore define HAVE_ERAND48, so that lrand48 is not redeclared
    # in $GP_HOME/include/postgresql/server/port.h.
    set_source_files_properties(${PDLTOOLS_DBAL_SOURCES} PROPERTIES
      COMPILE_FLAGS "-DHAVE_ERAND48 ${ADDITIONAL_GCC_FLAGS}")
    get_property(_OLD_VALUE TARGET pdltools_${DBMS} PROPERTY LINK_FLAGS)
    # If property has not been initialized, _OLD_VALUE is "_OLD_VALUE-NOTFOUND"
    set_target_properties(pdltools_${DBMS} PROPERTIES LINK_FLAGS "${_OLD_VALUE} ${ADDITIONAL_GCC_FLAGS}")
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "SunPro")
    # In syncbitvector.h, an array has zero length. This is non-standard,
    # but most compilers allow it. SunPro needs an extra option.
    set_source_files_properties(${PDLTOOLS_DBAL_SOURCES} PROPERTIES
      COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -features=zla")
  endif(CMAKE_COMPILER_IS_GNUCXX)
  
  # From the CMake documentation:
  # "Source file properties are visible only to targets added in the same
  # directory (CMakeLists.txt)." We therefore have to set them for each
  # version of this port!
  if(CMAKE_COMPILER_IS_GNUCC)
    # FIXME: Is there a portable (not just for gcc) way of including a header file?
    # Due to Greenplum bug MPP-13254, we need to include <sys/time.h>
    # before <postgres.h>
    # FIXME: In the C code, we have several places where strict aliasing
    # rules are violated. See this web page for some background:
    # http://dbp-consulting.com/tutorials/StrictAliasing.html
    # For now, we tell GCC that it cannot rely on strict aliasing rules.
    # Downside: We forgo some potential optimization.
    # The warning GCC would output without -fno-strict-aliasing is:
    # dereferencing type-punned pointer will break strict-aliasing rules
    set_source_files_properties(${LEGACY_C_FILES} PROPERTIES
      COMPILE_FLAGS "-fno-strict-aliasing -include sys/time.h ${ADDITIONAL_GCC_FLAGS}")
  endif(CMAKE_COMPILER_IS_GNUCC)
  
  set_source_files_properties(${LEGACY_C_FILES} PROPERTIES
    COMPILE_DEFINITIONS "NO_PG_MODULE_MAGIC")
  
  # sparse_vector.c indirectly includes openssl/ssl.h (via libpq/libpq-be.h)
  include_directories(SYSTEM ${${DBMS_UC}_CLIENT_INCLUDE_DIR})
  
  # --------------------------------------------------------------------
  # 3.2. Install shared library, Python files, and M4 header
  # --------------------------------------------------------------------
  
  cpack_add_version_component()

  install(TARGETS pdltools_${DBMS}
        LIBRARY DESTINATION ports/${PORT_DIR_NAME}/${IN_PORT_VERSION}/lib
        COMPONENT ${DBMS}
    )
  
  # --------------------------------------------------------------------
  # 4. Generate gppkg deployment scripts. Note: These are generated CMake
  #    scripts!
  # --------------------------------------------------------------------
  if(NOT (${IN_PORT_VERSION} VERSION_LESS "4.2"))
      add_gppkg(${IN_PORT_VERSION} Greenplum gpdb)
  endif(NOT (${IN_PORT_VERSION} VERSION_LESS "4.2"))


endfunction(add_${PORT_LC}_library)

# Add the current version as a target. This is a macro because it calls
# find_package, and we want the side effects (the defined variables) to be
# visible after the invocation

macro(add_current_${PORT_LC}_version)
  get_filename_component(_VERSION "${CMAKE_CURRENT_SOURCE_DIR}" NAME)
  string(REPLACE "." "_" _VERSION_UNDERSCORES ${_VERSION})
  
  find_package(${PORT}_${_VERSION_UNDERSCORES})
  if(${PORT_UC}_${_VERSION_UNDERSCORES}_FOUND)
    add_greenplum_library(${_VERSION})
  endif(${PORT_UC}_${_VERSION_UNDERSCORES}_FOUND)
endmacro(add_current_${PORT_LC}_version)

# ======================================================================
# 5. Build shared library and copy version-specific file for all
#    ${PORT_UC}_X_Y_PG_CONFIG macros defined by the user. If none has
#    been defined, try to find any version this
#    port.
# ======================================================================

determine_target_versions(VERSIONS)
foreach(VERSION ${VERSIONS})
  add_subdirectory(${VERSION})
endforeach(VERSION)
