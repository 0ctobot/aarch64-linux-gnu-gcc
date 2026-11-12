# Maintainer: Anatol Pomozov <anatol.pomozov@gmail.com>
# Maintainer: Eli Schwartz <eschwartz@archlinux.org>
# Contributor: Adam W. Willis <return.of.octobot@gmail.com> (Neutrino GCC)

# Neutrino GCC: GCC trunk plus Sultan Alsawaf's simple_gcc fixes and our own,
# from https://github.com/0ctobot/gcc (branch neutrino/gcc-17). The source is a
# git archive of $_commit; make it with:
#   git -C ~/toolchains/src/gcc archive --format=tar \
#     --prefix=gcc-neutrino-${_commit:0:9}/ $_commit | zstd -T4 -10 \
#     -o gcc-neutrino-${_commit:0:9}.tar.zst
# The host tools are built by the bootstrapped Neutrino GCC host compiler in
# $_hostcc with LTO, linking libstdc++ and libgcc statically so that the
# package doesn't depend on it at run time.

_target=aarch64-linux-gnu
pkgname=$_target-gcc-neutrino
_commit=9b380445d547cf2eb1fdd70e8a8ad9fa9dab95b4
_basever=17.0.0
_datestamp=20261112
pkgver=$_basever.r$_datestamp.g${_commit:0:9}
pkgrel=1
pkgdesc='The GNU Compiler Collection - cross compiler for ARM64 target (Neutrino GCC)'
arch=(x86_64)
url='https://github.com/0ctobot/gcc'
license=(GPL LGPL FDL)
depends=($_target-binutils $_target-glibc libmpc zlib libisl zstd)
makedepends=(gmp mpfr)
provides=("$_target-gcc=$_basever")
conflicts=($_target-gcc)
options=(!emptydirs !strip staticlibs !lto)
source=(gcc-neutrino-${_commit:0:9}.tar.zst)
sha256sums=('73ab34fc833d366ec351be0b50ac12a84549c9a58a20d207922bac14ae8a8d0e')

_basedir=gcc-neutrino-${_commit:0:9}
_hostcc=$HOME/toolchains/gcc-17-host
# Optional PGO profile for this exact commit: ~/toolchains/pkg-pgo.sh builds
# this package with NEUTRINO_PGO=train (instrumented), trains it with a kernel
# build and keeps the profile here, so it matches this build's own flags,
# options and source paths
_pgo=$HOME/toolchains/pgo/${_commit:0:9}-pkg

prepare() {
  cd $_basedir

  [[ $(cat gcc/BASE-VER) == $_basever && $(cat gcc/DATESTAMP) == $_datestamp ]] ||
    { echo "BASE-VER/DATESTAMP don't match the PKGBUILD"; return 1; }
  "$_hostcc"/bin/gcc --version | head -1 | grep -q 'Neutrino GCC' ||
    { echo "$_hostcc isn't a Neutrino GCC host compiler"; return 1; }

  # Do not run fixincludes
  sed -i 's@\./fixinc\.sh@-c true@' gcc/Makefile.in

  rm -rf "$srcdir"/gcc-build
  mkdir "$srcdir"/gcc-build
}

build() {
  cd gcc-build

  # using -pipe causes spurious test-suite failures
  # http://gcc.gnu.org/bugzilla/show_bug.cgi?id=48565
  CFLAGS=${CFLAGS/-pipe/}
  CXXFLAGS=${CXXFLAGS/-pipe/}

  # Credits @allanmcrae
  # https://github.com/allanmcrae/toolchain/blob/f18604d70c5933c31b51a320978711e4e6791cf1/gcc/PKGBUILD
  # TODO: properly deal with the build issues resulting from this
  CFLAGS=${CFLAGS/-Werror=format-security/}
  CXXFLAGS=${CXXFLAGS/-Werror=format-security/}
  unset FFLAGS FCFLAGS

  # Probe and build the target libraries with the binutils trunk that the
  # aarch64-linux-gnu-binutils-trunk package installs
  export PATH="$HOME/toolchains/binutils-trunk/bin:$PATH"
  $_target-as --version | head -1 | grep -q 'Binutils trunk' ||
    { echo "binutils trunk not found"; return 1; }

  # Build the host tools with the Neutrino host compiler and LTO; the
  # generator programs run during the build don't need LTO
  export CC="$_hostcc"/bin/gcc CXX="$_hostcc"/bin/g++
  export AR="$_hostcc"/bin/gcc-ar RANLIB="$_hostcc"/bin/gcc-ranlib NM="$_hostcc"/bin/gcc-nm
  export CFLAGS_FOR_BUILD="$CFLAGS" CXXFLAGS_FOR_BUILD="$CXXFLAGS"
  if [[ ${NEUTRINO_PGO:-} == train ]]; then
    # Instrumented build for collecting the profile.  No LTO: GCC instruments
    # each function before LTO runs, so the profile fits the LTO build too
    rm -rf "$_pgo"; mkdir -p "$_pgo"
    _pgoflags=" -fprofile-generate=$_pgo -fprofile-update=prefer-atomic"
    _pgoflags+=" -fprofile-prefix-path=$srcdir/gcc-build"
    CFLAGS+=$_pgoflags CXXFLAGS+=$_pgoflags
    # Some programs (c++tools) link without CXXFLAGS but need libgcov
    LDFLAGS+=" -fprofile-generate=$_pgo"
    _features=", profiling"
  else
  CFLAGS+=" -flto=jobserver"
  CXXFLAGS+=" -flto=jobserver"
  _features=", +lto"
  # The profile files are named by object path relative to the build
  # directory.  Code the training build didn't run only loses its profile,
  # and -fprofile-correction repairs counters that don't quite add up
  if [[ -n $(find "$_pgo" -name '*.gcda' -print -quit 2>/dev/null) ]]; then
    _pgoflags=" -fprofile-use=$_pgo -fprofile-prefix-path=$srcdir/gcc-build"
    _pgoflags+=" -fprofile-partial-training -fprofile-correction"
    _pgoflags+=" -Wno-missing-profile -Wno-error=coverage-mismatch"
    CFLAGS+=$_pgoflags CXXFLAGS+=$_pgoflags
    _features+=", +pgo"
  fi
  fi

  "$srcdir"/$_basedir/configure \
      --prefix=/usr \
      --program-prefix=$_target- \
      --with-local-prefix=/usr/$_target \
      --with-sysroot=/usr/$_target \
      --with-build-sysroot=/usr/$_target \
      --with-native-system-header-dir=/include \
      --libdir=/usr/lib --libexecdir=/usr/lib \
      --target=$_target --host=$CHOST --build=$CHOST \
      --disable-nls --enable-default-pie \
      --enable-languages=c,c++,fortran \
      --enable-shared --enable-threads=posix \
      --with-system-zlib --with-isl --enable-__cxa_atexit \
      --disable-libunwind-exceptions --enable-clocale=gnu \
      --disable-libstdcxx-pch --disable-libssp \
      --enable-gnu-unique-object --enable-linker-build-id \
      --enable-lto --enable-plugin --enable-install-libiberty \
      --with-linker-hash-style=gnu --enable-gnu-indirect-function \
      --disable-multilib --disable-werror \
      --enable-checking=release \
      --with-static-standard-libraries \
      "--with-pkgversion=Neutrino GCC ${_commit:0:9}$_features"

  # More jobs than this runs this machine out of memory during LTO
  make -j6
}

package() {
  cd gcc-build

  # handle lib vs lib64; for regular gcc this is handled by filesystem package
  mkdir -p "$pkgdir"/usr/$_target/lib
  ln -s ./lib "$pkgdir"/usr/$_target/lib64

  make DESTDIR="$pkgdir" install-gcc install-target-{libgcc,libstdc++-v3,libgomp,libgfortran,libquadmath,libatomic}

  # strip target binaries
  export PATH="$HOME/toolchains/binutils-trunk/bin:$PATH"
  find "$pkgdir"/usr/lib/gcc/$_target/ "$pkgdir"/usr/$_target/lib \
       -type f -and \( -name \*.a -or -name \*.o \) \
       -exec $_target-objcopy -R .comment -R .note -R .debug_info -R .debug_aranges \
           -R .debug_pubnames -R .debug_pubtypes -R .debug_abbrev -R .debug_line \
           -R .debug_str -R .debug_ranges -R .debug_loc '{}' \;

  # strip host binaries
  find "$pkgdir"/usr/bin/ "$pkgdir"/usr/lib/gcc/$_target/ -type f -and \( -executable \) -exec strip '{}' \;

  # The host compiler is only needed to build the package
  if find "$pkgdir"/usr/bin "$pkgdir"/usr/lib/gcc/$_target -type f -executable \
       -exec ldd '{}' \; 2>/dev/null | grep -E 'not found|gcc-17-host'; then
    echo "host binaries depend on $_hostcc"; return 1
  fi

  # Remove files that conflict with host gcc package
  rm -r "$pkgdir"/usr/share/man/man7
  rm -r "$pkgdir"/usr/share/info
  rm -r "$pkgdir"/usr/share/gcc-$_basever
}
