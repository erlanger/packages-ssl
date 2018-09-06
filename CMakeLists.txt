cmake_minimum_required(VERSION 2.8.12)
project(swipl-ssl)

include("../cmake/PrologPackage.cmake")
include(CheckTypeSize)
include(CheckStructHasMember)
include(Sockets)

find_package(OpenSSL)
if(OPENSSL_FOUND)
set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES} ${OPENSSL_LIBRARIES})
set(CMAKE_REQUIRED_INCLUDES  ${CMAKE_REQUIRED_INCLUDES}  ${OPENSSL_INCLUDE_DIR})

AC_CHECK_HEADERS(unistd.h sys/types.h sys/time.h sys/select.h fcntl.h
		 Security/Security.h)

check_c_source_compiles(
    "#include <sys/types.h>
     #include <Security/Security.h>

     const void *key = kSecClass;
     int main() { return 0;}"
    HAVE_KSECCLASS)

AC_CHECK_FUNCS(timegm ERR_remove_state ERR_remove_thread_state)
AC_CHECK_FUNCS(X509_check_host)
AC_CHECK_FUNCS(CRYPTO_THREADID_get_callback CRYPTO_THREADID_set_callback)
AC_CHECK_FUNCS(EVP_MD_CTX_free OPENSSL_zalloc)
AC_CHECK_FUNCS(X509_CRL_get0_signature X509_get0_signature)
AC_CHECK_FUNCS(X509_get0_notBefore X509_get0_notAfter)
AC_CHECK_FUNCS(X509_digest X509_CRL_digest)
AC_CHECK_FUNCS(X509_STORE_CTX_get0_chain)
AC_CHECK_FUNCS(i2d_re_X509_tbs)
AC_CHECK_FUNCS(OpenSSL_version)
AC_CHECK_FUNCS(EVP_CIPHER_CTX_reset)
AC_CHECK_FUNCS(EVP_blake2b512 EVP_blake2s256)
AC_CHECK_FUNCS(EVP_sha3_224 EVP_sha3_256 EVP_sha3_384 EVP_sha3_512)
AC_CHECK_FUNCS(HMAC_CTX_new HMAC_CTX_free)
AC_CHECK_FUNCS(SSL_CTX_set_alpn_protos)

AC_CHECK_HEADERS(openssl/kdf.h)

if(NOT DEFINED GET0SIG_CONST_T)
  if(HAVE_X509_GET0_SIGNATURE)
    check_c_source_compiles(
	"#include <openssl/x509.h>

	 int main() {
	   ASN1_BIT_STRING *psig;
	   X509_ALGOR *palg;
	   const X509 *data;

	   X509_get0_signature(&psig, &palg, data);
	   return 0;
	 }"
	 GET_SIGNATURE_PASSED)
    if(GET_SIGNATURE_PASSED)
      set(GET0SIG_CONST_T const)
    else()
      set(GET0SIG_CONST_T)
    endif()
  else(HAVE_X509_GET0_SIGNATURE)
    set(GET0SIG_CONST_T const)
  endif(HAVE_X509_GET0_SIGNATURE)

  set(GET0SIG_CONST_T ${GET0SIG_CONST_T}
      CACHE INTERNAL "Define X509_get0_signature const args")
endif(NOT DEFINED GET0SIG_CONST_T)

check_type_size(CRYPTO_THREADID SIZEOF_CRYPTO_THREADID)
if(NOT SIZEOF_CRYPTO_THREADID STREQUAL "")
  set(HAVE_CRYPTO_THREADID)
endif()

check_struct_has_member(X509_VERIFY_PARAM id openssl/ssl.h
			HAVE_X509_VERIFY_PARAM_ID)

if(NOT DEFINED SYSTEM_CACERT_FILENAME)
  set(CERT_CANDIDATES
      /etc/ssl/certs/ca-certificates.crt
      /etc/pki/tls/certs/ca-bundle.crt
      /etc/ssl/ca-bundle.pem
      /etc/ssl/cert.pem)

  set(SYSTEM_CACERT_FILENAME /etc/ssl/certs/ca-certificates.crt)
  foreach(f ${CERT_CANDIDATES})
    if(EXISTS ${f})
      set(SYSTEM_CACERT_FILENAME ${f})
    endif()
  endforeach()
  set(SYSTEM_CACERT_FILENAME ${SYSTEM_CACERT_FILENAME}
      CACHE INTERNAL
      "Location of the system certificate file")
endif()

configure_file(config.h.cmake config.h)

add_custom_command(
    OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/tests/test_certs
    COMMAND cd ${CMAKE_CURRENT_SOURCE_DIR}/tests &&
	    bash mkcerts.sh > /dev/null 2>&1)
add_custom_target(
    test_certificates ALL
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/tests/test_certs)

swipl_plugin(
    ssl4pl
    C_SOURCES ssl4pl.c ../clib/nonblockio.c ../clib/error.c
    C_LIBS ${OPENSSL_SSL_LIBRARY}
    PL_LIBS ssl.pl)
swipl_plugin(
    crypto4pl
    C_SOURCES crypto4pl.c ../clib/nonblockio.c ../clib/error.c
    C_LIBS ${OPENSSL_CRYPTO_LIBRARY}
    PL_LIBS crypto.pl)

swipl_examples(client.pl server.pl https.pl)

test_libs(
    ssl
    PACKAGES clib sgml http
)

endif(OPENSSL_FOUND)