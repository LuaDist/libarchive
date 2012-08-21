#
#
project ( libarchive C )
#
cmake_minimum_required ( VERSION 2.8 )
set ( CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/build/cmake" )
if ( NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY )
  set ( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${libarchive_BINARY_DIR}/bin )
endif ( )
include ( cmake/dist.cmake )

#
# Version - read from 'version' file.
#
file ( STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/build/version _version )
string ( REGEX REPLACE "^([0-9])[0-9][0-9][0-9][0-9][0-9][0-9][a-z]?$" "\\1" _major 
  ${_version} )
string ( REGEX REPLACE "^[0-9]([0-9][0-9][0-9])[0-9][0-9][0-9][a-z]?$" "\\1" _minor 
  ${_version} )
string ( REGEX REPLACE "^[0-9][0-9][0-9][0-9]([0-9][0-9][0-9])[a-z]?$" "\\1" _revision 
  ${_version} )
string ( REGEX REPLACE "^[0-9][0-9][0-9][0-9][0-9][0-9][0-9]([a-z]?)$" "\\1" _quality 
  ${_version} )
set ( _version_number ${_major}${_minor}${_revision} )
string ( REGEX REPLACE "[0]*([^0][0-9]*)$" "\\1" _minor ${_minor} )
string ( REGEX REPLACE "[0]*([^0][0-9]*)$" "\\1" _revision ${_revision} )
#
set ( VERSION "${_major}.${_minor}.${_revision}${_quality}" )
set ( BSDCPIO_VERSION_STRING "${VERSION}" )
set ( BSDTAR_VERSION_STRING "${VERSION}" )
set ( LIBARCHIVE_VERSION_NUMBER "${_version_number}" )
set ( LIBARCHIVE_VERSION_STRING "${VERSION}" )
# Shared library number
set ( SOVERSION 8 )

# Enable CTest/CDash support
include ( CTest )

# Provide ADD_TEST_28 macro to approximate CMake 2.8 ADD_TEST(NAME).
# TODO: Require CMake 2.8 and drop this workaround (perhaps late 2010).
include ( AddTest28 )

option ( ENABLE_OPENSSL "Enable use of OpenSSL" ON )
option ( ENABLE_TAR "Enable tar building" ON )
option ( ENABLE_TAR_SHARED "Enable dynamic build of tar" ON )
option ( ENABLE_CPIO "Enable cpio building" ON )
option ( ENABLE_CPIO_SHARED "Enable dynamic build of cpio" ON )
option ( ENABLE_XATTR "Enable extended attribute support" ON )
option ( ENABLE_ACL "Enable ACL support" ON )
option ( ENABLE_TEST "Enable unit and regression tests" OFF )

if ( ENABLE_TEST )
  enable_testing ( )
endif ( ENABLE_TEST )
if ( WIN32 AND NOT CYGWIN )
  # Currently, dynamic build only.
  set ( ENABLE_TAR_SHARED ON )
  set ( ENABLE_CPIO_SHARED ON )
endif ( WIN32 AND NOT CYGWIN )

if ( WIN32 )
  set ( _WIN32_WINNT 0 x0500 CACHE INTERNAL "Setting _WIN32_WINNT to 0x0500 for Windows 2000 APIs" )
  set ( WINVER 0 x0500 CACHE INTERNAL "Setting WINVER to 0x0500 for Windows 2000 APIs" )
endif ( WIN32 )

#
include ( CheckCSourceRuns )
include ( CheckFileOffsetBits )
include ( CheckFuncs )
include ( CheckHeaderDirent )
include ( CheckIncludeFile )
include ( CheckIncludeFiles )
include ( CheckLibraryExists )
include ( CheckStructMember )
include ( CheckSymbolExists )
include ( CheckTypeExists )
include ( CheckTypeSize )

#
# Generate list.h
#
macro ( GENERATE_LIST_H _listfile _cmlist __list_sources )
set ( _argv ${ARGV} )
# Remove _listfile and _cmlist from _argv
list ( REMOVE_AT _argv 0 1 )
if ( NOT EXISTS "${_listfile}" OR ${_cmlist} IS_NEWER_THAN "${_listfile}" )

  message ( STATUS "Generating ${_listfile}" )
  file ( WRITE ${_listfile} "" )
  foreach ( testfile ${_argv} )
  if ( testfile MATCHES "^test_[^/]+[.]c$" )
    file ( STRINGS ${testfile} testvar REGEX "^DEFINE_TEST" )
    foreach ( deftest ${testvar} )
    file ( APPEND ${_listfile} "${deftest}\n" )
    endforeach ( deftest )
  endif ( testfile MATCHES "^test_[^/]+[.]c$" )
  endforeach ( testfile )

endif ( NOT EXISTS "${_listfile}" OR ${_cmlist} IS_NEWER_THAN "${_listfile}" )
endmacro ( GENERATE_LIST_H )
#
# Generate installation rules for man pages.
#
macro ( INSTALL_MAN __mans )
foreach ( _man ${ARGV} )
string ( REGEX REPLACE "^.+[.]([1-9])" "\\1" _mansect ${_man} )
install_doc ( ${_man} )
endforeach ( _man )
endmacro ( INSTALL_MAN __mans )

#
# Check compress/decompress libraries
#
if ( WIN32 AND NOT CMAKE_CL_64 AND NOT CYGWIN )
  # GnuWin32 is only for Win32, not Win64.
  set ( __GNUWIN32PATH "C:/Program Files/GnuWin32" )
endif ( WIN32 AND NOT CMAKE_CL_64 AND NOT CYGWIN )
if ( DEFINED __GNUWIN32PATH AND EXISTS "${__GNUWIN32PATH}" )
  # You have to add a path availabel DLL file into PATH environment variable.
  # Maybe DLL path is "C:/Program Files/GnuWin32/bin".
  # The zlib and the bzip2 Setup program have installed programs and DLLs into
  # "C:/Program Files/GnuWin32" by default.
  # This is convenience setting for Windows.
  set ( CMAKE_PREFIX_PATH ${__GNUWIN32PATH} $ ( CMAKE_PREFIX_PATH ) )
  #
  # If you didn't use Setup program or installed into nonstandard path,
  # cmake cannot find out your zlib or bzip2 libraries and include files,
  # you should execute cmake with  -DCMAKE_PREFIX_PATH option.
  #   e.g.
  #     cmake -DCMAKE_PREFIX_PATH=<your-GnuWin32-path> <path-to-source>
  #
  # If compiling error occured in zconf.h, You may need patch to zconf.h.
  #--- zconf.h.orig	2005-07-21 00:40:26.000000000
  #+++ zconf.h	2009-01-19 11:39:10.093750000
  #@@ -286,7 +286,7 @@
  # 
  # #if 1           /* HAVE_UNISTD_H -- this line is updated by ./configure */
  # #  include <sys/types.h> /* for off_t */
  #-#  include <unistd.h>    /* for SEEK_* and off_t */
  #+#  include <stdio.h>    /* for SEEK_* and off_t */
  # #  ifdef VMS
  # #    include <unixio.h>   /* for off_t */
  # #  endif
endif ( DEFINED __GNUWIN32PATH AND EXISTS "${__GNUWIN32PATH}" )

set ( ADDITIONAL_LIBS "" )
#
# Find ZLIB
#
find_package ( ZLIB )
if ( ZLIB_FOUND )
  set ( HAVE_LIBZ 1 )
  set ( HAVE_ZLIB_H 1 )
  include_directories ( ${ZLIB_INCLUDE_DIR} )
  list ( APPEND ADDITIONAL_LIBS ${ZLIB_LIBRARIES} )
endif ( ZLIB_FOUND )
mark_as_advanced ( CLEAR ZLIB_INCLUDE_DIR )
mark_as_advanced ( CLEAR ZLIB_LIBRARY )
#
# Find BZip2
#
find_package ( BZip2 )
if ( BZIP2_FOUND )
  set ( HAVE_LIBBZ2 1 )
  set ( HAVE_BZLIB_H 1 )
  include_directories ( ${BZIP2_INCLUDE_DIR} )
  list ( APPEND ADDITIONAL_LIBS ${BZIP2_LIBRARIES} )
endif ( BZIP2_FOUND )
mark_as_advanced ( CLEAR BZIP2_INCLUDE_DIR )
mark_as_advanced ( CLEAR BZIP2_LIBRARY )
#
# Find LZMA
#
find_package ( LZMA )
if ( LZMA_FOUND )
  set ( HAVE_LIBLZMA 1 )
  set ( HAVE_LZMA_H 1 )
  include_directories ( ${LZMA_INCLUDE_DIR} )
  list ( APPEND ADDITIONAL_LIBS ${LZMA_LIBRARIES} )
  mark_as_advanced ( CLEAR LZMA_INCLUDE_DIR )
  mark_as_advanced ( CLEAR LZMA_LIBRARY )
elseif ( LZMADEC_FOUND )
set ( HAVE_LIBLZMADEC 1 )
set ( HAVE_LZMADEC_H 1 )
include_directories ( ${LZMADEC_INCLUDE_DIR} )
list ( APPEND ADDITIONAL_LIBS ${LZMADEC_LIBRARIES} )
mark_as_advanced ( CLEAR LZMADEC_INCLUDE_DIR )
mark_as_advanced ( CLEAR LZMADEC_LIBRARY )
else ( LZMA_FOUND )
  mark_as_advanced ( CLEAR LZMA_INCLUDE_DIR )
  mark_as_advanced ( CLEAR LZMA_LIBRARY )
endif ( LZMA_FOUND )

#
# Check headers
#
check_header_dirent ( )

set ( INCLUDES "" )
macro ( LA_CHECK_INCLUDE_FILE header var )
check_include_files ( "${INCLUDES};${header}" ${var} )
if ( ${var} )
  set ( INCLUDES ${INCLUDES} ${header} )
endif ( ${var} )
endmacro ( LA_CHECK_INCLUDE_FILE )

# Few headers that must precede other headers
# Must precede sys/extattr.h on FreeBSD
la_check_include_file ( "sys/types.h" HAVE_SYS_TYPES_H )

# Alphabetize the rest unless there's a compelling reason
la_check_include_file ( "acl/libacl.h" HAVE_ACL_LIBACL_H )
la_check_include_file ( "attr/xattr.h" HAVE_ATTR_XATTR_H )
la_check_include_file ( "ctype.h" HAVE_CTYPE_H )
la_check_include_file ( "direct.h" HAVE_DIRECT_H )
la_check_include_file ( "dlfcn.h" HAVE_DLFCN_H )
la_check_include_file ( "errno.h" HAVE_ERRNO_H )
la_check_include_file ( "ext2fs/ext2_fs.h" HAVE_EXT2FS_EXT2_FS_H )
la_check_include_file ( "fcntl.h" HAVE_FCNTL_H )
la_check_include_file ( "grp.h" HAVE_GRP_H )
la_check_include_file ( "inttypes.h" HAVE_INTTYPES_H )
la_check_include_file ( "io.h" HAVE_IO_H )
la_check_include_file ( "langinfo.h" HAVE_LANGINFO_H )
la_check_include_file ( "limits.h" HAVE_LIMITS_H )
la_check_include_file ( "linux/fs.h" HAVE_LINUX_FS_H )
la_check_include_file ( "locale.h" HAVE_LOCALE_H )
la_check_include_file ( "memory.h" HAVE_MEMORY_H )
la_check_include_file ( "paths.h" HAVE_PATHS_H )
la_check_include_file ( "poll.h" HAVE_POLL_H )
la_check_include_file ( "process.h" HAVE_PROCESS_H )
la_check_include_file ( "pwd.h" HAVE_PWD_H )
la_check_include_file ( "regex.h" HAVE_REGEX_H )
la_check_include_file ( "signal.h" HAVE_SIGNAL_H )
la_check_include_file ( "stdarg.h" HAVE_STDARG_H )
la_check_include_file ( "stdint.h" HAVE_STDINT_H )
la_check_include_file ( "stdlib.h" HAVE_STDLIB_H )
la_check_include_file ( "string.h" HAVE_STRING_H )
la_check_include_file ( "strings.h" HAVE_STRINGS_H )
la_check_include_file ( "sys/acl.h" HAVE_SYS_ACL_H )
la_check_include_file ( "sys/cdefs.h" HAVE_SYS_CDEFS_H )
la_check_include_file ( "sys/extattr.h" HAVE_SYS_EXTATTR_H )
la_check_include_file ( "sys/ioctl.h" HAVE_SYS_IOCTL_H )
la_check_include_file ( "sys/mkdev.h" HAVE_SYS_MKDEV_H )
la_check_include_file ( "sys/param.h" HAVE_SYS_PARAM_H )
la_check_include_file ( "sys/poll.h" HAVE_SYS_POLL_H )
la_check_include_file ( "sys/select.h" HAVE_SYS_SELECT_H )
la_check_include_file ( "sys/stat.h" HAVE_SYS_STAT_H )
la_check_include_file ( "sys/time.h" HAVE_SYS_TIME_H )
la_check_include_file ( "sys/utime.h" HAVE_SYS_UTIME_H )
la_check_include_file ( "sys/wait.h" HAVE_SYS_WAIT_H )
la_check_include_file ( "time.h" HAVE_TIME_H )
la_check_include_file ( "unistd.h" HAVE_UNISTD_H )
la_check_include_file ( "utime.h" HAVE_UTIME_H )
la_check_include_file ( "wchar.h" HAVE_WCHAR_H )
la_check_include_file ( "wctype.h" HAVE_WCTYPE_H )
la_check_include_file ( "windows.h" HAVE_WINDOWS_H )

#
# Some headers require extra includes when they're available.
#

#
# Find OpenSSL
#
if ( ENABLE_OPENSSL )
  find_package ( OpenSSL )
else ( )
  set ( OPENSSL_FOUND 0 )
endif ( )
if ( OPENSSL_FOUND )
  include_directories ( ${OPENSSL_INCLUDE_DIR} )
  list ( APPEND ADDITIONAL_LIBS ${OPENSSL_LIBRARIES} )
else ( )
  # Block OpenSSL checks and override cached results.
  set ( HAVE_OPENSSL_MD5_H 0 )
  set ( HAVE_OPENSSL_RIPEMD_H 0 )
  set ( HAVE_OPENSSL_SHA_H 0 )
  set ( HAVE_OPENSSL_SHA256_INIT 0 )
  set ( HAVE_OPENSSL_SHA384_INIT 0 )
  set ( HAVE_OPENSSL_SHA512_INIT 0 )
endif ( )
#
# Check MD5/RMD160/SHA headers
#
la_check_include_file ( "md5.h" HAVE_MD5_H )
la_check_include_file ( "openssl/md5.h" HAVE_OPENSSL_MD5_H )
la_check_include_file ( "openssl/ripemd.h" HAVE_OPENSSL_RIPEMD_H )
la_check_include_file ( "openssl/sha.h" HAVE_OPENSSL_SHA_H )
la_check_include_file ( "ripemd.h" HAVE_RIPEMD_H )
la_check_include_file ( "rmd160.h" HAVE_RMD160_H )
la_check_include_file ( "sha.h" HAVE_SHA_H )
la_check_include_file ( "sha1.h" HAVE_SHA1_H )
la_check_include_file ( "sha2.h" HAVE_SHA2_H )
la_check_include_file ( "sha256.h" HAVE_SHA256_H )

#
# Find MD5/RMD160/SHA library
#
find_library ( CRYPTO_LIBRARY NAMES crypto )
if ( CRYPTO_LIBRARY )
  list ( APPEND ADDITIONAL_LIBS ${CRYPTO_LIBRARY} )
else ( CRYPTO_LIBRARY )
  if ( NOT OPENSSL_FOUND )
    find_library ( MD_LIBRARY NAMES md )
    if ( MD_LIBRARY )
      list ( APPEND ADDITIONAL_LIBS ${MD_LIBRARY} )
    endif ( MD_LIBRARY )
  endif ( NOT OPENSSL_FOUND )
endif ( CRYPTO_LIBRARY )
#
# Check MD5/RMD160/SHA functions
#
set ( CMAKE_REQUIRED_LIBRARIES ${ADDITIONAL_LIBS} )
if ( HAVE_MD5_H )
  check_symbol_exists ( MD5Init "md5.h" HAVE_MD5INIT )
endif ( HAVE_MD5_H )
if ( HAVE_RMD160_H )
  check_symbol_exists ( RMD160Init "rmd160.h" HAVE_RMD160INIT )
endif ( HAVE_RMD160_H )
if ( HAVE_SHA2_H )
  check_symbol_exists ( SHA256Init "sha2.h" HAVE_SHA256INIT )
  check_symbol_exists ( SHA384Init "sha2.h" HAVE_SHA384INIT )
  check_symbol_exists ( SHA512Init "sha2.h" HAVE_SHA512INIT )
  check_symbol_exists ( SHA256_Init "sha2.h" HAVE_SHA256_INIT )
  check_symbol_exists ( SHA384_Init "sha2.h" HAVE_SHA384_INIT )
  check_symbol_exists ( SHA512_Init "sha2.h" HAVE_SHA512_INIT )
elseif ( HAVE_OPENSSL_SHA_H )
check_symbol_exists ( SHA256_Init "openssl/sha.h" HAVE_OPENSSL_SHA256_INIT )
check_symbol_exists ( SHA384_Init "openssl/sha.h" HAVE_OPENSSL_SHA384_INIT )
check_symbol_exists ( SHA512_Init "openssl/sha.h" HAVE_OPENSSL_SHA512_INIT )
endif ( )
set ( CMAKE_REQUIRED_LIBRARIES "" )

#
# Find Libxml2
#
find_package ( LibXml2 )
if ( LIBXML2_FOUND )
  include_directories ( ${LIBXML2_INCLUDE_DIR} )
  list ( APPEND ADDITIONAL_LIBS ${LIBXML2_LIBRARIES} )
  set ( HAVE_LIBXML2 1 )
  # libxml2's include files use iconv.h
  # We need a directory path of iconv.h so that it won't fail to check
  # "libxml/xmlreader.h".
  find_path ( ICONV_INCLUDE_DIR iconv.h )
  include_directories ( ${ICONV_INCLUDE_DIR} )
  set ( CMAKE_REQUIRED_INCLUDES ${ICONV_INCLUDE_DIR} ${LIBXML2_INCLUDE_DIR} )
  check_include_files ( "libxml/xmlreader.h" HAVE_LIBXML_XMLREADER_H )
  set ( CMAKE_REQUIRED_INCLUDES "" )
else ( LIBXML2_FOUND )
  #
  # Find Expat
  #
  find_package ( EXPAT )
  if ( EXPAT_FOUND )
    include_directories ( ${EXPAT_INCLUDE_DIR} )
    list ( APPEND ADDITIONAL_LIBS ${EXPAT_LIBRARIES} )
    set ( HAVE_LIBEXPAT 1 )
    la_check_include_file ( "expat.h" HAVE_EXPAT_H )
  endif ( EXPAT_FOUND )
endif ( LIBXML2_FOUND )

#
# Check functions
#
check_symbol_exists ( CreateHardLinkA "windows.h" HAVE_CREATEHARDLINKA )
check_symbol_exists ( CreateHardLinkW "windows.h" HAVE_CREATEHARDLINKW )
check_symbol_exists ( _CrtSetReportMode "crtdbg.h" HAVE__CrtSetReportMode )
check_function_exists_glibc ( chflags HAVE_CHFLAGS )
check_function_exists_glibc ( chown HAVE_CHOWN )
check_function_exists_glibc ( chroot HAVE_CHROOT )
check_function_exists_glibc ( fchdir HAVE_FCHDIR )
check_function_exists_glibc ( fchflags HAVE_FCHFLAGS )
check_function_exists_glibc ( fchmod HAVE_FCHMOD )
check_function_exists_glibc ( fchown HAVE_FCHOWN )
check_function_exists_glibc ( fcntl HAVE_FCNTL )
check_function_exists_glibc ( fork HAVE_FORK )
check_function_exists_glibc ( fstat HAVE_FSTAT )
check_function_exists_glibc ( ftruncate HAVE_FTRUNCATE )
check_function_exists_glibc ( futimens HAVE_FUTIMENS )
check_function_exists_glibc ( futimes HAVE_FUTIMES )
check_function_exists_glibc ( geteuid HAVE_GETEUID )
check_function_exists_glibc ( getpid HAVE_GETPID )
check_function_exists_glibc ( lchflags HAVE_LCHFLAGS )
check_function_exists_glibc ( lchmod HAVE_LCHMOD )
check_function_exists_glibc ( lchown HAVE_LCHOWN )
check_function_exists_glibc ( link HAVE_LINK )
check_function_exists_glibc ( lstat HAVE_LSTAT )
check_function_exists_glibc ( lutimes HAVE_LUTIMES )
check_function_exists_glibc ( memmove HAVE_MEMMOVE )
check_function_exists_glibc ( mkdir HAVE_MKDIR )
check_function_exists_glibc ( mkfifo HAVE_MKFIFO )
check_function_exists_glibc ( mknod HAVE_MKNOD )
check_function_exists_glibc ( nl_langinfo HAVE_NL_LANGINFO )
check_function_exists_glibc ( pipe HAVE_PIPE )
check_function_exists_glibc ( poll HAVE_POLL )
check_function_exists_glibc ( readlink HAVE_READLINK )
check_function_exists_glibc ( select HAVE_SELECT )
check_function_exists_glibc ( setenv HAVE_SETENV )
check_function_exists_glibc ( setlocale HAVE_SETLOCALE )
check_function_exists_glibc ( sigaction HAVE_SIGACTION )
check_function_exists_glibc ( strchr HAVE_STRCHR )
check_function_exists_glibc ( strdup HAVE_STRDUP )
check_function_exists_glibc ( strerror HAVE_STRERROR )
check_function_exists_glibc ( strncpy_s HAVE_STRNCPY_S )
check_function_exists_glibc ( strrchr HAVE_STRRCHR )
check_function_exists_glibc ( symlink HAVE_SYMLINK )
check_function_exists_glibc ( timegm HAVE_TIMEGM )
check_function_exists_glibc ( tzset HAVE_TZSET )
check_function_exists_glibc ( unsetenv HAVE_UNSETENV )
check_function_exists_glibc ( utime HAVE_UTIME )
check_function_exists_glibc ( utimes HAVE_UTIMES )
check_function_exists_glibc ( utimensat HAVE_UTIMENSAT )
check_function_exists_glibc ( vfork HAVE_VFORK )
check_function_exists_glibc ( wcrtomb HAVE_WCRTOMB )
check_function_exists_glibc ( wcscpy HAVE_WCSCPY )
check_function_exists_glibc ( wcslen HAVE_WCSLEN )
check_function_exists_glibc ( wctomb HAVE_WCTOMB )
check_symbol_exists ( wmemcmp "wchar.h" HAVE_WMEMCMP )
check_symbol_exists ( wmemcpy "wchar.h" HAVE_WMEMCPY )

set ( CMAKE_REQUIRED_LIBRARIES "" )
check_symbol_exists ( fseeko "stdio.h" HAVE_FSEEKO )
check_symbol_exists ( strerror_r "string.h" HAVE_STRERROR_R )
check_symbol_exists ( strftime "time.h" HAVE_STRFTIME )
check_symbol_exists ( vprintf "stdio.h" HAVE_VPRINTF )
check_symbol_exists ( cygwin_conv_path "sys/cygwin.h" HAVE_CYGWIN_CONV_PATH )

check_symbol_exists ( major "sys/mkdev.h" MAJOR_IN_MKDEV )
check_symbol_exists ( major "sys/sysmacros.h" MAJOR_IN_SYSMACROS )

if ( HAVE_STRERROR_R )
  set ( HAVE_DECL_STRERROR_R 1 )
endif ( HAVE_STRERROR_R )

#
# Check defines
#
set ( headers "limits.h" )
if ( HAVE_STDINT_H )
  list ( APPEND headers "stdint.h" )
endif ( HAVE_STDINT_H )
if ( HAVE_INTTYPES_H )
  list ( APPEND headers "inttypes.h" )
endif ( HAVE_INTTYPES_H )
check_symbol_exists ( EFTYPE "errno.h" HAVE_EFTYPE )
check_symbol_exists ( EILSEQ "errno.h" HAVE_EILSEQ )
check_symbol_exists ( D_MD_ORDER "langinfo.h" HAVE_D_MD_ORDER )
check_symbol_exists ( optarg "unistd.h" HAVE_DECL_OPTARG )
check_symbol_exists ( optind "unistd.h" HAVE_DECL_OPTIND )
check_symbol_exists ( INT64_MAX "${headers}" HAVE_DECL_INT64_MAX )
check_symbol_exists ( INT64_MIN "${headers}" HAVE_DECL_INT64_MIN )
check_symbol_exists ( UINT32_MAX "${headers}" HAVE_DECL_UINT32_MAX )
check_symbol_exists ( UINT64_MAX "${headers}" HAVE_DECL_UINT64_MAX )
check_symbol_exists ( SIZE_MAX "${headers}" HAVE_DECL_SIZE_MAX )
check_symbol_exists ( SSIZE_MAX "limits.h" HAVE_DECL_SSIZE_MAX )

#
# Check struct members
#
# Check for birthtime in struct stat
check_struct_member ( "struct stat" st_birthtime "sys/types.h;sys/stat.h" HAVE_STRUCT_STAT_ST_BIRTHTIME )

# Check for high-resolution timestamps in struct stat
check_struct_member ( "struct stat" st_birthtimespec.tv_nsec "sys/types.h;sys/stat.h" 
  HAVE_STRUCT_STAT_ST_BIRTHTIMESPEC_TV_NSEC )
check_struct_member ( "struct stat" st_mtimespec.tv_nsec "sys/types.h;sys/stat.h" 
  HAVE_STRUCT_STAT_ST_MTIMESPEC_TV_NSEC )
check_struct_member ( "struct stat" st_mtim.tv_nsec "sys/types.h;sys/stat.h" HAVE_STRUCT_STAT_ST_MTIM_TV_NSEC )
check_struct_member ( "struct stat" st_mtime_n "sys/types.h;sys/stat.h" HAVE_STRUCT_STAT_ST_MTIME_N )
check_struct_member ( "struct stat" st_umtime "sys/types.h;sys/stat.h" HAVE_STRUCT_STAT_ST_UMTIME )
check_struct_member ( "struct stat" st_mtime_usec "sys/types.h;sys/stat.h" HAVE_STRUCT_STAT_ST_MTIME_USEC )
# Check for block size support in struct stat
check_struct_member ( "struct stat" st_blksize "sys/types.h;sys/stat.h" HAVE_STRUCT_STAT_ST_BLKSIZE )
# Check for st_flags in struct stat (BSD fflags)
check_struct_member ( "struct stat" st_flags "sys/types.h;sys/stat.h" HAVE_STRUCT_STAT_ST_FLAGS )
#
#
check_struct_member ( "struct tm" tm_sec "sys/types.h;sys/time.h;time.h" TIME_WITH_SYS_TIME )

#
# Check for integer types
#
# XXX There must be a way to make this simpler <sigh> XXXX
#
check_type_size ( "long long int" LONG_LONG_INT )
check_type_size ( "unsigned long long" UNSIGNED_LONG_LONG )
check_type_size ( "unsigned long long int" UNSIGNED_LONG_LONG_INT )

#
check_type_size ( dev_t DEV_T )
if ( NOT HAVE_DEV_T )
  if ( MSVC )
    set ( dev_t "unsigned int" )
  endif ( MSVC )
endif ( NOT HAVE_DEV_T )
#
check_type_size ( gid_t GID_T )
if ( NOT HAVE_GID_T )
  if ( WIN32 )
    set ( gid_t "short" )
  else ( WIN32 )
    set ( gid_t "unsigned int" )
  endif ( WIN32 )
endif ( NOT HAVE_GID_T )
#
check_type_size ( id_t ID_T )
if ( NOT HAVE_ID_T )
  if ( WIN32 )
    set ( id_t "short" )
  else ( WIN32 )
    set ( id_t "unsigned int" )
  endif ( WIN32 )
endif ( NOT HAVE_ID_T )
#
check_type_size ( int32_t INT32_T )
if ( NOT HAVE_INT32_T )
  set ( int32_t "int" )
endif ( NOT HAVE_INT32_T )
#
check_type_size ( int64_t INT64_T )
if ( NOT HAVE_INT64_T )
  if ( WIN32 )
    set ( int64_t __int64 )
  endif ( WIN32 )
endif ( NOT HAVE_INT64_T )
#
check_type_size ( intmax_t INTMAX_T )
if ( NOT HAVE_INTMAX_T )
  set ( intmax_t "int64_t" )
endif ( NOT HAVE_INTMAX_T )
#
check_type_size ( mode_t MODE_T )
if ( NOT HAVE_MODE_T )
  if ( WIN32 )
    set ( mode_t "unsigned short" )
  else ( WIN32 )
    set ( mode_t "int" )
  endif ( WIN32 )
endif ( NOT HAVE_MODE_T )
#
check_type_size ( off_t OFF_T )
if ( NOT HAVE_OFF_T )
  set ( off_t "__int64" )
endif ( NOT HAVE_OFF_T )
#
check_type_size ( size_t SIZE_T )
if ( NOT HAVE_SIZE_T )
  if ( "${CMAKE_SIZEOF_VOID_P}" EQUAL 8 )
    set ( size_t "uint64_t" )
  else ( "${CMAKE_SIZEOF_VOID_P}" EQUAL 8 )
    set ( size_t "uint32_t" )
  endif ( "${CMAKE_SIZEOF_VOID_P}" EQUAL 8 )
endif ( NOT HAVE_SIZE_T )
#
check_type_size ( ssize_t SSIZE_T )
if ( NOT HAVE_SSIZE_T )
  if ( "${CMAKE_SIZEOF_VOID_P}" EQUAL 8 )
    set ( ssize_t "int64_t" )
  else ( "${CMAKE_SIZEOF_VOID_P}" EQUAL 8 )
    set ( ssize_t "long" )
  endif ( "${CMAKE_SIZEOF_VOID_P}" EQUAL 8 )
endif ( NOT HAVE_SSIZE_T )
#
check_type_size ( uid_t UID_T )
if ( NOT HAVE_UID_T )
  if ( WIN32 )
    set ( uid_t "short" )
  else ( WIN32 )
    set ( uid_t "unsigned int" )
  endif ( WIN32 )
endif ( NOT HAVE_UID_T )
#
check_type_size ( pid_t PID_T )
if ( NOT HAVE_PID_T )
  if ( WIN32 )
    set ( pid_t "int" )
  else ( WIN32 )
    message ( FATAL_ERROR "pid_t doesn't exist on this platform?" )
  endif ( WIN32 )
endif ( NOT HAVE_PID_T )
#
check_type_size ( uint16_t UINT16_T )
if ( NOT HAVE_UINT16_T )
  set ( uint16_t "unsigned short" )
endif ( NOT HAVE_UINT16_T )
#
check_type_size ( uint32_t UINT32_T )
if ( NOT HAVE_UINT32_T )
  set ( uint32_t "unsigned int" )
endif ( NOT HAVE_UINT32_T )
#
check_type_size ( uint64_t UINT64_T )
if ( NOT HAVE_UINT64_T )
  if ( WIN32 )
    set ( uint64_t "unsigned __int64" )
  endif ( WIN32 )
endif ( NOT HAVE_UINT64_T )
#
check_type_size ( uintmax_t UINTMAX_T )
if ( NOT HAVE_UINTMAX_T )
  set ( uintmax_t "uint64_t" )
endif ( NOT HAVE_UINTMAX_T )
#
check_type_size ( intptr_t INTPTR_T )
if ( NOT HAVE_INTPTR_T )
  if ( "${CMAKE_SIZEOF_VOID_P}" EQUAL 8 )
    set ( intptr_t "int64_t" )
  else ( )
    set ( intptr_t "int32_t" )
  endif ( )
endif ( NOT HAVE_INTPTR_T )
#
check_type_size ( uintptr_t UINTPTR_T )
if ( NOT HAVE_UINTPTR_T )
  if ( "${CMAKE_SIZEOF_VOID_P}" EQUAL 8 )
    set ( uintptr_t "uint64_t" )
  else ( )
    set ( uintptr_t "uint32_t" )
  endif ( )
endif ( NOT HAVE_UINTPTR_T )
#
check_type_size ( wchar_t SIZEOF_WCHAR_T )
if ( HAVE_SIZEOF_WCHAR_T )
  set ( HAVE_WCHAR_T 1 )
endif ( HAVE_SIZEOF_WCHAR_T )
#
# Check if _FILE_OFFSET_BITS macro needed for large files
#
check_file_offset_bits ( )

#
# Check for Extended Attribute libraries, headers, and functions
#
if ( ENABLE_XATTR )
  la_check_include_file ( attr/xattr.h HAVE_ATTR_XATTR_H )
  la_check_include_file ( sys/xattr.h HAVE_SYS_XATTR_H )
  check_library_exists ( attr "setxattr" "" HAVE_ATTR_LIB )
  if ( HAVE_ATTR_LIB )
    set ( CMAKE_REQUIRED_LIBRARIES "attr" )
  endif ( HAVE_ATTR_LIB )
  check_function_exists_glibc ( extattr_get_file HAVE_EXTATTR_GET_FILE )
  check_function_exists_glibc ( extattr_list_file HAVE_EXTATTR_LIST_FILE )
  check_function_exists_glibc ( extattr_set_fd HAVE_EXTATTR_SET_FD )
  check_function_exists_glibc ( extattr_set_file HAVE_EXTATTR_SET_FILE )
  check_function_exists_glibc ( fsetxattr HAVE_FSETXATTR )
  check_function_exists_glibc ( getxattr HAVE_GETXATTR )
  check_function_exists_glibc ( lgetxattr HAVE_LGETXATTR )
  check_function_exists_glibc ( listxattr HAVE_LISTXATTR )
  check_function_exists_glibc ( llistxattr HAVE_LLISTXATTR )
  check_function_exists_glibc ( lsetxattr HAVE_LSETXATTR )
endif ( ENABLE_XATTR )

#
# Check for ACL libraries, headers, and functions
#
# The ACL support in libarchive is written against the POSIX1e draft,
# which was never officially approved and varies quite a bit across
# platforms.  Worse, some systems have completely non-POSIX acl functions,
# which makes the following checks rather more complex than I would like.
#
if ( ENABLE_ACL )
  check_library_exists ( acl "acl_get_file" "" HAVE_ACL_LIB )
  if ( HAVE_ACL_LIB )
    set ( CMAKE_REQUIRED_LIBRARIES "acl" )
    find_library ( ACL_LIBRARY NAMES acl )
    list ( APPEND ADDITIONAL_LIBS ${ACL_LIBRARY} )
  endif ( HAVE_ACL_LIB )
  #
  check_function_exists_glibc ( acl_create_entry HAVE_ACL_CREATE_ENTRY )
  check_function_exists_glibc ( acl_init HAVE_ACL_INIT )
  check_function_exists_glibc ( acl_set_fd HAVE_ACL_SET_FD )
  check_function_exists_glibc ( acl_set_fd_np HAVE_ACL_SET_FD_NP )
  check_function_exists_glibc ( acl_set_file HAVE_ACL_SET_FILE )
  check_type_exists ( acl_permset_t "${INCLUDES}" HAVE_ACL_PERMSET_T )

  # The "acl_get_perm()" function was omitted from the POSIX draft.
  # (It's a pretty obvious oversight; otherwise, there's no way to
  # test for specific permissions in a permset.)  Linux uses the obvious
  # name, FreeBSD adds _np to mark it as "non-Posix extension."
  # Test for both as a double-check that we really have POSIX-style ACL support.
  check_symbol_exists ( acl_get_perm "${INCLUDES}" HAVE_ACL_GET_PERM )
  check_symbol_exists ( acl_get_perm_np "${INCLUDES}" HAVE_ACL_GET_PERM_NP )
  check_symbol_exists ( acl_get_link "${INCLUDES}" HAVE_ACL_GET_LINK )
  check_symbol_exists ( acl_get_link_np "${INCLUDES}" HAVE_ACL_GET_LINK_NP )

  # MacOS has an acl.h that isn't POSIX.  It can be detected by
  # checking for ACL_USER
  check_symbol_exists ( ACL_USER "${INCLUDES}" HAVE_ACL_USER )
endif ( ENABLE_ACL )

# Generate "config.h" from "build/cmake/config.h.in"
configure_file ( ${CMAKE_CURRENT_SOURCE_DIR}/build/cmake/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h )
include_directories ( BEFORE ${CMAKE_CURRENT_BINARY_DIR} )
add_definitions ( -DHAVE_CONFIG_H )

#
# Register installation of PDF documents.
#
if ( WIN32 AND NOT CYGWIN )
  #
  # On Windows platform, It's better that we install PDF documents
  # on one's computer.
  # These PDF documents are available in the release package.
  #
  if ( EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/doc/pdf )
    install_doc ( ${CMAKE_CURRENT_SOURCE_DIR}/doc/pdf )
  endif ( EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/doc/pdf )
endif ( WIN32 AND NOT CYGWIN )
#
#
#
include_directories ( BEFORE ${CMAKE_CURRENT_SOURCE_DIR}/libarchive )
#
if ( MSVC )
  add_definitions ( -D_CRT_SECURE_NO_DEPRECATE )
endif ( MSVC )
# Especially for early development, we want to be a little
# aggressive about diagnosing build problems; this can get
# relaxed somewhat in final shipping versions.
if ( "CMAKE_C_COMPILER_ID" MATCHES "^GNU$" )
#  add_definitions ( -Wall -Werror )
endif ( "CMAKE_C_COMPILER_ID" MATCHES "^GNU$" )

if ( ENABLE_TEST )
  add_custom_target ( run_all_tests )
endif ( ENABLE_TEST )

add_subdirectory ( libarchive )
add_subdirectory ( tar )
add_subdirectory ( cpio )
