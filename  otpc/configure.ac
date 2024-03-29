# configure.ac
AC_INIT(otpd, 3.1.0, frank@tri-dsystems.com)

AC_PREREQ([2.59])
AC_COPYRIGHT([Copyright 2005-2008 TRI-D Systems, Inc.])
AC_REVISION([$Revision: #1 $])

AC_CONFIG_SRCDIR([main.c])
PACKAGE_DATE="2008-02-11"

# 64-bit support (to append to --with lib paths)
AC_ARG_WITH(lib64,
            AS_HELP_STRING([--with-lib64=DIR], [additional suffix for 64-bit libs]),
            [ LIB64="$withval" ])
# for now we only support openssl (for DES, MD4, MD5, SHA1, SSL)
AC_ARG_WITH(openssl,
            AS_HELP_STRING([--with-openssl=DIR], [openssl directory]),
            [ CFLAGS="$CFLAGS -I$withval/include"
              LDFLAGS="$LDFLAGS -L$withval/lib$LIB64 -R$withval/lib$LIB64" ])
# make sure user knows trid isn't available for their platform
AC_ARG_WITH(trid,
          AS_HELP_STRING([--without-trid], [don't include TRI-D token support]))
# libldap
AC_ARG_WITH(included-libldap,
            AS_HELP_STRING([--with-included-libldap], [use included libldap]))

# Various CHECK macros set $DEFS, but we need to set $DEFINES to allow
# our Makefile to pass them as compiler -D options, without carrying the
# extra baggage included in $DEFS.  If we end up with lots of these,
# we'll move to config.h, but for now this is why you'll see strange
# constructs which set $DEFINES.

# Solaris needs _REENTRANT
DEFINES="$DEFINES -D_REENTRANT"

# Need to know the target for -gstabs test and trid.o
AC_CANONICAL_TARGET

##
## Checks for programs.
##
AC_PROG_CC
# http://sources.redhat.com/bugzilla/show_bug.cgi?id=144
if test "$CC" = "gcc" -o "$CC" = "$target-gcc" ; then
    case $target in
        sparc-sun-solaris*) CFLAGS="`echo $CFLAGS | sed s/-g/-gstabs/`" ;;
    esac
fi
# make sure TLS is supported (e.g., sunfreeware.com gcc is misconfigured)
AC_MSG_CHECKING(for thread-local storage support)
AC_COMPILE_IFELSE(
  [AC_LANG_PROGRAM([[static __thread int foo;]])],
  [AC_MSG_RESULT(yes)],
  [AC_MSG_RESULT(no)
   AC_MSG_ERROR(toolchain is either too old or misconfigured)])

AC_PROG_LEX
# lex assumes stdin and stdout are constants, but this is not true for glibc.
# For glibc platforms, we need flex.  Let's just require it everywhere.
# This isn't really needed unless config.l is changed, but requiring this
# anyway should have minimal impact.  It's not sufficient to test for
# recency of config.l vs scanner.c (to skip this test) because config.l
# could be updated after configure is run.
if test "$LEX" = "lex" ; then
    AC_MSG_ERROR(flex is required)
fi
AC_PROG_YACC
AC_PROG_INSTALL

##
## Checks for typedefs, structures, and compiler characteristics.
##
AC_C_CONST

##
## Checks for header files.
##
AC_HEADER_STDC
AC_CHECK_HEADER(inttypes.h, ,AC_MSG_ERROR(please update your system))
AC_CHECK_HEADER(openssl/md4.h, ,AC_MSG_ERROR(unable to find openssl headers))

##
## Checks for libraries.
##
AC_CHECK_LIB(pthread, pthread_create, ,
             AC_MSG_ERROR(libpthread is required))
AC_CHECK_LIB(socket, socket, SOCKET_LIBS=-lsocket)
# For now, we use only unix domain sockets
DEFINES="$DEFINES -DUSE_SOCKET"
# Solaris needs -lnsl for gethostbyname() et al.
AC_CHECK_LIB(nsl, gethostbyname)
# Solaris needs -lresolv for hstrerror()
AC_CHECK_LIB(resolv, hstrerror)
dnl dlsym(), for cyclades, no longer supported.
dnl to add it back, need to also pass -rdynamic to $CC iff using GNU ld
dnl see cl 2449 for ld test
dnl # dlsym() needed for config.y, otp.c
dnl AC_CHECK_LIB(dl, dlsym)
AC_CHECK_LIB(crypto, MD4, ,
             AC_MSG_ERROR(OpenSSL(libcrypto) is required))
# Solaris needs -lrt for nanosleep()
AC_CHECK_LIB(rt, nanosleep)

##
## Checks for library functions.
##
AC_CHECK_FUNCS([closefrom], DEFINES="$DEFINES -DHAVE_CLOSEFROM")

# Test for clock_gettime(), but only on Linux, because we don't want
# to add -lrt on Solaris (Solaris has clock_gettime, but we don't use it.)
# Need to update this when we support more than Solaris and Linux.
AC_PREPROC_IFELSE(
  [AC_LANG_PROGRAM([[#ifndef __linux__
                     #error not linux
                     #endif]])],
  [AC_SEARCH_LIBS(clock_gettime, [rt])],)
if test "${ac_cv_search_clock_gettime-no}" != no; then
  DEFINES="$DEFINES -DHAVE_CLOCK_GETTIME"
fi

AC_CHECK_FUNCS([pset_info], DEFINES="$DEFINES -DHAVE_PSET_INFO")

# use the correct tri-d module
if test -f cardops/trid.c ; then
  :
elif test -f cardops/trid.o.$target ; then
  cp cardops/trid.o.$target cardops/trid.o
  chmod +w cardops/trid.o
elif test "$with_trid" != "no" ; then
  AC_MSG_ERROR([TRI-D support for $target not found; use --without-trid to disable])
fi

if test "${with_included_libldap+set}" != "set"; then
  # use system ldap by default, but fallback to included ldap
  AC_CHECK_LIB(ldap, ldap_initialize, use_included_libldap=no,
                                      use_included_libldap=yes)
  if test "$use_included_libldap" = "no"; then
    AC_CHECK_LIB(ldap, ldap_start_tls, , use_included_libldap=yes)
  fi
  if test "$use_included_libldap" = "no"; then
    # this is really just to add -lber to LIBS; we already know we have openldap
    AC_CHECK_LIB(lber, ber_free, , use_included_libldap=yes)
  fi
elif test "$with_included_libldap" = "no"; then
  # same tests as above, but fail on error
  AC_CHECK_LIB(ldap, ldap_initialize, ,
               AC_MSG_ERROR([OpenLDAP is required]))
  # empty (:) action to avoid adding -lldap to LIBS twice
  AC_CHECK_LIB(ldap, ldap_start_tls, :,
               AC_MSG_ERROR([OpenLDAP >= 2.3.4 is required]))
  # this is really just to add -lber to LIBS; we already know we have openldap
  AC_CHECK_LIB(lber, ber_free, ,
               AC_MSG_ERROR([OpenLDAP is required]))
  use_included_libldap=no
else
  use_included_libldap=yes
fi

# Choose included libldap (default) or system libldap.
if test "$use_included_libldap" = "yes"; then
dnl  # we already tested for libcrypto, this is just here as a reminder
dnl  AC_CHECK_LIB(crypto, RSA_generate_key, ,
dnl               AC_MSG_ERROR(OpenSSL(libcrypto) is required))
  AC_CHECK_LIB(ssl, SSL_library_init, ,
               AC_MSG_ERROR(OpenSSL(libssl) is required))

  USE_LDAP=included
  LDAP_DIR=openldap-2.3.32
  CPPFLAGS="$CPPFLAGS -I\$(srcdir)/\$(LDAP_DIR)/include"
  LDAP_LIBS="\$(srcdir)/\$(LDAP_DIR)/libraries/libldap/.libs/libldap.a \$(srcdir)/\$(LDAP_DIR)/libraries/liblber/.libs/liblber.a"

  # Fix up the openldap configure script to not require non-lib portions.
  # We could just ship it already modified, but for future upgrade it's better
  # to have 3rd party source as pristine as possible.
  # This "patch" is safe to run multiple times.  Keep it that way!
  chmod +w $LDAP_DIR/configure
  perl -i -pe 's: (doc|clients|servers|tests|libraries/(lib(ldap_r|rewrite)))/[[^\s"]]+::g if /^\s*ac_config_files=/' $LDAP_DIR/configure
  # " close quote to improve syntax highlighting
  perl -i -pe 's/^/: / if m:^chmod \+x tests/run:' $LDAP_DIR/configure
  perl -i -pe 's:".*:junk.c: if /^BACKENDSC/' $LDAP_DIR/configure
  perl -i -pe 's:".*:junk.c: if /^OVERLAYSC/' $LDAP_DIR/configure
  # don't print this confusing message
  perl -i -pe 's/^/: / if m:^echo Please run:' $LDAP_DIR/configure

  # don't install ldap.conf
  # Makefile is written in such a way that we can't just pass args to avoid it
  M=$LDAP_DIR/libraries/libldap/Makefile.in
  chmod +w $M
  # can't ':' this one out because it starts with a special Make char (-)
  perl -i -ne 'print unless /^\s+-..MKDIR.*sysconfdir/' $M
  # ':' out the actual install
  perl -i -pe 's/(?<!: )(..INSTALL)/: $1/ if /sysconfdir/' $M

  # disable some openldap features
  enable_backends=no; export enable_backends
  enable_null=yes; export enable_null
  with_cyrus_sasl=no; export with_cyrus_sasl
  enable_shared=no; export enable_shared

  # would have used $LDAP_VERSION above but AC_C_S is not allowed to take $vars
  AC_CONFIG_SUBDIRS(openldap-2.3.32)
else
  USE_LDAP=system
fi

##
AC_CONFIG_FILES([Makefile])
# We set all defines explicitly because DEFS has a lot of extra garbage
AC_SUBST(DEFINES)
AC_SUBST(PACKAGE_DATE)
AC_SUBST(USE_LDAP)
AC_SUBST(LDAP_DIR)
AC_SUBST(LDAP_LIBS)
AC_SUBST(SOCKET_LIBS)
AC_OUTPUT
