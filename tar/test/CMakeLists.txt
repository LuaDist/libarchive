############################################
#
# How to build bsdtar_test
#
############################################
if ( ENABLE_TAR AND ENABLE_TEST )
  set ( bsdtar_test_SOURCES ../getdate.c main.c test.h test_0.c test_basic.c test_copy.c 
    test_empty_mtree.c test_getdate.c test_help.c test_option_T_upper.c test_option_q.c 
    test_option_r.c test_option_s.c test_patterns.c test_stdio.c test_strip_components.c 
    test_symlink_dir.c test_version.c test_windows.c )
  if ( WIN32 AND NOT CYGWIN )
    list ( APPEND bsdtar_test_SOURCES ../bsdtar_windows.c )
    list ( APPEND bsdtar_test_SOURCES ../bsdtar_windows.h )
  endif ( WIN32 AND NOT CYGWIN )

  #
  # Register target
  #
  add_executable ( bsdtar_test ${bsdtar_test_SOURCES} )
  set_property ( TARGET bsdtar_test PROPERTY COMPILE_DEFINITIONS LIST_H )

  #
  # Generate list.h by grepping DEFINE_TEST() lines out of the C sources.
  #
  generate_list_h ( ${CMAKE_CURRENT_BINARY_DIR}/list.h ${CMAKE_CURRENT_LIST_FILE} 
    ${bsdtar_test_SOURCES} )
  set_property ( DIRECTORY APPEND PROPERTY INCLUDE_DIRECTORIES ${CMAKE_CURRENT_BINARY_DIR} )

  # list.h has a line DEFINE_TEST(testname) for every
  # test.  We can use that to define the tests for cmake by
  # defining a DEFINE_TEST macro and reading list.h in.
  macro ( DEFINE_TEST _testname )
  add_test_28 ( NAME bsdtar_${_testname} COMMAND bsdtar_test -vv -p $<TARGET_FILE:bsdtar> 
    -r ${CMAKE_CURRENT_SOURCE_DIR} ${_testname} )
  endmacro ( DEFINE_TEST _testname )

  include ( ${CMAKE_CURRENT_BINARY_DIR}/list.h )

  # Experimental new test handling
  add_custom_target ( run_bsdtar_test COMMAND bsdtar_test -p ${BSDTAR} -r ${CMAKE_CURRENT_SOURCE_DIR} )
  add_dependencies ( run_bsdtar_test bsdtar )
  add_dependencies ( run_all_tests run_bsdtar_test )

endif ( ENABLE_TAR AND ENABLE_TEST )
