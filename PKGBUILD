#!/bin/hint/bash
# Maintainer: Fredrick Brennan <copypaste@kittens.ph>
# Submitter: Lukas Jirkovsky <l.jirkovsky@gmail.com>
# Co-maintainer : bartus <arch-user-repoᘓbartus.33mail.com>

# Configuration.
_fragment=${FRAGMENT:-#branch=master}
[[ -v CUDA_ARCH ]] && _CUDA_ARCH=(${CUDA_ARCH})
((TRAVIS)) && _cuda_capability+=(sm_50 sm_52 sm_60 sm_61 sm_70 sm_75)

# fix gcc:11 regression: https://bugs.archlinux.org/task/70930
makedepends+=('gcc10')
_CMAKE_FLAGS+=( -DCMAKE_C_COMPILER=gcc-10
                -DCMAKE_CXX_COMPILER=g++-10
)

#some extra, unofficially supported stuff goes here:
_CMAKE_FLAGS+=( -DWITH_CYCLES_NETWORK=OFF )

pkgname=blender-git
pkgver=3.0.r106621.g7b62a542308
pkgrel=1
pkgdesc="A fully integrated 3D graphics creation suite (development)"
arch=('i686' 'x86_64')
url="https://blender.org/"
depends+=('alembic' 'embree' 'libgl' 'python' 'python-numpy' 'openjpeg2' 'libharu' 'potrace' 'openxr'
         'ffmpeg' 'fftw' 'openal' 'freetype2' 'libxi' 'openimageio' 'opencolorio'
         'openvdb' 'opencollada' 'opensubdiv' 'openshadinglanguage' 'libtiff' 'libpng')
optdepends=('cuda: CUDA support in Cycles'
            'optix=7.1.0: OptiX support in Cycles'
            'usd=21.05: USD export Scene'
            'openimagedenoise: Intel Open Image Denoise support in compositing')
makedepends+=('git' 'cmake' 'boost' 'mesa' 'ninja' 'llvm')
provides=('blender')
conflicts=('blender')
license=('GPL')
# NOTE: the source array has to be kept in sync with .gitmodules
# the submodules has to be stored in path ending with git to match
# the path in .gitmodules.
# More info:
#   http://wiki.blender.org/index.php/Dev:Doc/Tools/Git
source=("git://git.blender.org/blender.git${_fragment}"
        'blender-addons.git::git://git.blender.org/blender-addons.git'
        'blender-addons-contrib.git::git://git.blender.org/blender-addons-contrib.git'
        'blender-translations.git::git://git.blender.org/blender-translations.git'
        'blender-dev-tools.git::git://git.blender.org/blender-dev-tools.git'
        usd_python.patch #add missing python headers when building against python enabled usd.
        embree.patch #add missing embree link.
        openexr3.patch #fix build against openexr:3
        )
sha256sums=('SKIP'
            'SKIP'
            'SKIP'
            'SKIP'
            'SKIP'
            '333b6fd864d55da2077bc85c55af1a27d4aee9764a1a839df26873a9f19b8703'
            '6249892f99ffd960e36f43fb893c14e2f8e4dd1d901b9581d25882e865f2603f'
            '5297dc61cc4edcc1d5bad3474ab882264b69d68036cebbd0f2600d9fe21d5a1b')

pkgver() {
  blender_version=$(grep -Po "BLENDER_VERSION \K[0-9]{3}" "$srcdir"/blender/source/blender/blenkernel/BKE_blender_version.h)
  printf "%d.%d.r%s.g%s" \
    $((blender_version/100)) \
    $((blender_version%100)) \
    "$(git -C "$srcdir/blender" rev-list --count HEAD)" \
    "$(git -C "$srcdir/blender" rev-parse --short HEAD)"
}

prepare() {
  cd "$srcdir/blender"
  # update the submodules
  git submodule update --init --recursive --remote
  git apply -v "${srcdir}"/{embree,usd_python,openexr3}.patch
}

build() {
  _pyver=$(python -c "from sys import version_info; print(\"%d.%d\" % (version_info[0],version_info[1]))")

  # determine whether we can precompile CUDA kernels
  _CUDA_PKG=`pacman -Qq cuda 2>/dev/null` || true
  if [ "$_CUDA_PKG" != "" ]; then
    _CMAKE_FLAGS+=( -DWITH_CYCLES_CUDA_BINARIES=ON
                  -DCUDA_TOOLKIT_ROOT_DIR=/opt/cuda )
    if [[ -v _CUDA_ARCH ]]; then
      _CMAKE_FLAGS+=( -DCYCLES_CUDA_BINARIES_ARCH="$(IFS=';'; echo "${_CUDA_ARCH[*]}";)" )
    fi
  fi

  # check for optix
  _OPTIX_PKG=`pacman -Qq optix 2>/dev/null` || true
  if [ "$_OPTIX_PKG" != "" ]; then
      _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_OPTIX=ON
                      -DOPTIX_ROOT_DIR=/opt/optix )
  fi

  # check for open image denoise
  _OIDN_PKG=`pacman -Qq openimagedenoise 2>/dev/null` || true
  if [ "$_OIDN_PKG" != "" ]; then
      _CMAKE_FLAGS+=( -DWITH_OPENIMAGEDENOISE=ON )
  fi

  # check for universal scene descriptor
  _USD_PKG=`pacman -Qq usd=21.02 2>/dev/null` || true
  if [ "$_USD_PKG" != "" ]; then
    _CMAKE_FLAGS+=( -DWITH_USD=ON
                    -DUSD_ROOT=/usr )
  fi

  cmake -G Ninja -S "$srcdir/blender" -B build \
        -C "${srcdir}/blender/build_files/cmake/config/blender_release.cmake" \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DCMAKE_BUILD_TYPE=Release \
        -DWITH_INSTALL_PORTABLE=OFF \
        -DWITH_SYSTEM_GLEW=ON \
        -DWITH_PYTHON_INSTALL=OFF \
        -DXR_OPENXR_SDK_ROOT_DIR=/usr \
        -DPYTHON_VERSION="${_pyver}" \
        "${_CMAKE_FLAGS[@]}"
  ninja -C "$srcdir/build" ${MAKEFLAGS:--j1}
}

package() {
  _suffix=${pkgver%%.r*}
  DESTDIR="$pkgdir" ninja -C "$srcdir/build" install

  if [[ -e "$pkgdir/usr/share/blender/${_suffix}/scripts/addons/cycles/lib/" ]] ; then
    # make sure the cuda kernels are not stripped
    chmod 444 "$pkgdir"/usr/share/blender/${_suffix}/scripts/addons/cycles/lib/*
  fi
}
