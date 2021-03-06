set(FDB_C_SRCS
  fdb_c.cpp
  foundationdb/fdb_c.h
  ThreadCleanup.cpp)

file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/foundationdb)

set(asm_file ${CMAKE_CURRENT_BINARY_DIR}/fdb_c.g.S)

set(platform "linux")
if(APPLE)
  set(platform "osx")
elseif(WIN32)
  set(platform "windows")
  set(asm_file ${CMAKE_CURRENT_BINARY_DIR}/fdb_c.g.asm)
endif()

add_custom_command(OUTPUT ${asm_file} ${CMAKE_CURRENT_BINARY_DIR}/fdb_c_function_pointers.g.h
  COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/generate_asm.py ${platform}
      ${CMAKE_CURRENT_SOURCE_DIR}/fdb_c.cpp
      ${asm_file}
      ${CMAKE_CURRENT_BINARY_DIR}/fdb_c_function_pointers.g.h
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/generate_asm.py ${CMAKE_CURRENT_SOURCE_DIR}/fdb_c.cpp
  COMMENT "Generate C bindings")
add_custom_target(fdb_c_generated DEPENDS ${asm_file}
      ${CMAKE_CURRENT_BINARY_DIR}/fdb_c_function_pointers.g.h)

vexillographer_compile(TARGET fdb_c_options LANG c OUT ${CMAKE_CURRENT_BINARY_DIR}/foundationdb/fdb_c_options.g.h
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/foundationdb/fdb_c_options.g.h)

include(GenerateExportHeader)

add_library(fdb_c SHARED ${FDB_C_SRCS} ${asm_file})
add_dependencies(fdb_c fdb_c_generated fdb_c_options)
target_link_libraries(fdb_c PUBLIC fdbclient)
target_include_directories(fdb_c PUBLIC
  ${CMAKE_CURRENT_BINARY_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_BINARY_DIR}/foundationdb)
if(WIN32)
  enable_language(ASM_MASM)
  set_property(SOURCE ${asm_file} PROPERTY LANGUAGE ASM_MASM)
endif()
# TODO: re-enable once the old vcxproj-based build system is removed.
#generate_export_header(fdb_c EXPORT_MACRO_NAME "DLLEXPORT"
#  EXPORT_FILE_NAME ${CMAKE_CURRENT_BINARY_DIR}/foundationdb/fdb_c_export.h)
install(TARGETS fdb_c
  EXPORT fdbc
  DESTINATION ${FDB_LIB_DIR}
  COMPONENT clients)
install(
  FILES foundationdb/fdb_c.h
        ${CMAKE_CURRENT_BINARY_DIR}/foundationdb/fdb_c_options.g.h
        ${CMAKE_SOURCE_DIR}/fdbclient/vexillographer/fdb.options
  DESTINATION ${FDB_INCLUDE_INSTALL_DIR}/foundationdb COMPONENT clients)
#install(EXPORT fdbc DESTINATION ${FDB_LIB_DIR}/foundationdb COMPONENT clients)
