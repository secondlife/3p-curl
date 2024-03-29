set(EXE_NAME curl)

set(tool_hugehelp_path "${CURL_BINARY_DIR}/src/tool_hugehelp.c")

if(USE_MANUAL)
  # Use the C locale to ensure that only ASCII characters appear in the
  # embedded text. NROFF and MANOPT are set in the parent CMakeLists.txt
  add_custom_command(
    OUTPUT ${tool_hugehelp_path}
    COMMAND ${CMAKE_COMMAND} -E echo "#include \"tool_setup.h\"" > ${tool_hugehelp_path}
    COMMAND ${CMAKE_COMMAND} -E echo "#ifndef HAVE_LIBZ" >> ${tool_hugehelp_path}
    COMMAND env LC_ALL=C "${NROFF}" ${NROFF_MANOPT}
            "${CURL_BINARY_DIR}/docs/curl.1" |
            "${PERL_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl"
            "${CURL_SOURCE_DIR}/docs/MANUAL" >> ${tool_hugehelp_path}
    COMMAND ${CMAKE_COMMAND} -E echo "#else" >> ${tool_hugehelp_path}
    COMMAND env LC_ALL=C "${NROFF}" ${NROFF_MANOPT}
            "${CURL_BINARY_DIR}/docs/curl.1" |
            "${PERL_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl" -c
            "${CURL_SOURCE_DIR}/docs/MANUAL" >> ${tool_hugehelp_path}
    COMMAND ${CMAKE_COMMAND} -E echo "#endif /* HAVE_LIBZ */" >> ${tool_hugehelp_path}
    DEPENDS
      "${CURL_SOURCE_DIR}/docs/MANUAL"
      generate-curl.1
      "${CURL_BINARY_DIR}/docs/curl.1"
      "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl"
      "${CMAKE_CURRENT_SOURCE_DIR}/tool_hugehelp.h"
    VERBATIM)
else()
  add_custom_command(
    OUTPUT ${tool_hugehelp_path}
    COMMAND ${CMAKE_COMMAND} -E echo "/* built-in manual is disabled, blank function */" > ${tool_hugehelp_path}
    COMMAND ${CMAKE_COMMAND} -E echo "#include \"tool_hugehelp.h\"" >> ${tool_hugehelp_path}
    COMMAND ${CMAKE_COMMAND} -E echo "void hugehelp(void) {}" >> ${tool_hugehelp_path}
    DEPENDS
      "${CMAKE_CURRENT_SOURCE_DIR}/tool_hugehelp.h"
    VERBATIM)

endif()

# This tool_hugehelp target that depends on generated ${tool_hugehelp_path}
# ensures that when tool_hugehelp is required, the correct add_custom_command
# above will run.
add_custom_target(tool_hugehelp DEPENDS ${tool_hugehelp_path})

transform_makefile_inc("Makefile.inc" "${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake")
include(${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake)

if(MSVC)
  list(APPEND CURL_SOURCE curl.rc)
endif()

# CURL_FILES comes from Makefile.inc
add_executable(
  ${EXE_NAME}
  ${CURL_FILES}
)
# This add_dependencies ensures that ${tool_hugehelp_path} will be generated
# before we try to build the executable. Otherwise CMake tries it in the other
# order, producing a "no such source file" error.
add_dependencies(${EXE_NAME} tool_hugehelp)

source_group("curlX source files" FILES ${CURLX_CFILES})
source_group("curl source files" FILES ${CURL_CFILES})
source_group("curl header files" FILES ${CURL_HFILES})

include_directories(
  ${CURL_SOURCE_DIR}/lib        # To be able to reach "curl_setup_once.h"
  ${CURL_BINARY_DIR}/lib        # To be able to reach "curl_config.h"
  ${CURL_BINARY_DIR}/include    # To be able to reach "curl/curlbuild.h"
  # This is needed as tool_hugehelp.c is generated in the binary dir
  ${CURL_SOURCE_DIR}/src        # To be able to reach "tool_hugehelp.h"
  )

#Build curl executable
target_link_libraries( ${EXE_NAME} libcurl ${CURL_LIBS})

################################################################################

#SET_TARGET_PROPERTIES(${EXE_NAME} ARCHIVE_OUTPUT_DIRECTORY "blah blah blah")
#SET_TARGET_PROPERTIES(${EXE_NAME} RUNTIME_OUTPUT_DIRECTORY "blah blah blah")
#SET_TARGET_PROPERTIES(${EXE_NAME} LIBRARY_OUTPUT_DIRECTORY "blah blah blah")

# Add the postfix to the executable since it is not added automatically as for modules and shared libraries
set_target_properties(${EXE_NAME} PROPERTIES
  DEBUG_POSTFIX "${CMAKE_DEBUG_POSTFIX}")

#INCLUDE(ModuleInstall OPTIONAL)

install(TARGETS ${EXE_NAME} DESTINATION bin)
