#!/bin/bash
# Maintainer: Fredrick Brennan <copypaste@kittens.ph>
# Submitter: Lukas Jirkovsky <l.jirkovsky@gmail.com>
# Co-maintainer: bartus <arch-user-repoᘓbartus.33mail.com>
# Contributor: Adrian Sausenthaler <aur@sausenthaler.de>

#Configuration:
#Use: makepkg VAR1=0 VAR2=1 to enable(1) disable(0) a feature
#Use: {yay,paru} --mflags=VAR1=0,VAR2=1
#Use: aurutils --margs=VAR1=0,VAR2=1
#Use: VAR1=0 VAR2=1 pamac

# Use FRAGMENT=#{commit,tag,brach}=xxx for bisect build
_fragment="${FRAGMENT:-#branch=main}"

# Use CUDA_ARCH to build for specific GPU architecture
# Supports: single arch (sm_52) and list of archs (sm_52;sm_60)
[[ -v CUDA_ARCH ]] && _CMAKE_FLAGS+=(-DCYCLES_CUDA_BINARIES_ARCH="${CUDA_ARCH}")
[[ -v HIP_ARCH  ]] && _CMAKE_FLAGS+=(-DCYCLES_HIP_BINARIES_ARCH="${HIP_ARCH}")

pkgname=blender-git
pkgver=5.0.r154311.gacde9be6fd2
pkgrel=1
pkgdesc="A fully integrated 3D graphics creation suite (development)"
arch=('i686' 'x86_64')
url="https://blender.org/"
depends+=('alembic' 'embree' 'libgl' 'python' 'python-numpy' 'openjpeg2' 'libharu' 'potrace' 'openxr'
          'ffmpeg' 'fftw' 'openal' 'freetype2' 'libxi' 'manifold' 'openimageio' 'opencolorio'
          'openvdb' 'opensubdiv' 'openshadinglanguage' 'libtiff' 'libpng'
          'python' 'python-zstandard' 'ccache')
depends+=('libdecor' 'libepoxy')
optdepends=('cuda: CUDA support in Cycles'
            'optix8: OptiX support in Cycles >=8.0.0 <9.0.0'
            'usd: USD export Scene'
            'openpgl: Intel Path Guiding library in Cycles'
            'openimagedenoise: Intel Open Image Denoise support in compositing'
            'materialx: MaterialX materials'
            'level-zero-headers: Intel OpenCL FPGA kernels (all four needed)'
            'intel-compute-runtime: Intel OpenCL FPGA kernels (all four needed)'
            'intel-graphics-compiler: Intel OpenCL FPGA kernels (all four needed)'
            'intel-oneapi-basekit: Intel OpenCL FPGA kernels (all four needed)'
            'makepkg-cg: Control resources during compilation')
makedepends+=('git' 'cmake' 'boost' 'mesa' 'llvm' 'clang' 'subversion')
makedepends+=('wayland-protocols')
makedepends+=('cython')
makedepends+=('vulkan-headers')
makedepends+=('makepkg-git-lfs-proto') # provides /usr/share/makepkg/source/git-lfs.sh (download_git-lfs, extract_git-lfs)
provides=('blender')
conflicts=('blender' 'blender-4.1-bin')
license=('GPL')
source=("blender::git-lfs+https://projects.blender.org/blender/blender${_fragment}"
  )
sha256sums=('SKIP'
  )

pkgver() {
  blender_version=$(grep -Po "BLENDER_VERSION \K[0-9]{3}" "$srcdir"/blender/source/blender/blenkernel/BKE_blender_version.h)
  printf "%d.%d.r%s.g%s" \
    $((blender_version/100)) \
    $((blender_version%100)) \
    "$(git -C "$srcdir/blender" rev-list --count HEAD)" \
    "$(git -C "$srcdir/blender" rev-parse --short HEAD)"
}

prepare() {
  mapfile -t patches < <(grep -Po '^.*?(patch|diff)(?=::|$)' < <(printf "${srcdir}/%s\n" ${source[@]}))
  for patch in "${patches[@]}"; do
    msg2  "apply ${patch##*/}..."
    git apply -v "${patches[@]}"
  done
}

build() {
  export PATH="/opt/lib:/opt/bin:$PATH"
  _pyver=$(python -c "from sys import version_info; print(\"%d.%d\" % (version_info[0],version_info[1]))")
  msg "python version detected: ${_pyver}"

  declare -a -g _CMAKE_FLAGS
  # determine whether we can install python modules
  if [[ -n "$_pyver" ]]; then
    export PYTHON_LIBRARY=/usr/lib/libpython${_pyver}.so
    export PYTHON_VERSION=${_pyver}
    _CMAKE_FLAGS+=( -DPYTHON_VERSION=$_pyver \
                    -DPYTHON_LIBRARY=/usr/lib/libpython${_pyver}.so \
                    -DWITH_PYTHON_INSTALL=ON \
                    -DWITH_PYTHON_SAFETY=OFF )
  fi

  export CUDAHOSTCXX="$NVCC_CCBIN"

  _CMAKE_FLAGS+=( -DWITH_CLANG=ON \
                  -DWITH_CYCLES=ON )

  # check for oneapi
  export _ONEAPI_CLANG=/opt/intel/oneapi/compiler/latest/linux/bin-llvm/clang
  export _ONEAPI_CLANGXX=/opt/intel/oneapi/compiler/latest/linux/bin-llvm/clang++
  if [ -f "$_ONEAPI_CLANG" ]; then
    _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_ONEAPI=ON \
                    -DWITH_CYCLES_ONEAPI_BINARIES=ON \
                    -DWITH_CLANG=ON )
  else
    # Because some defaults are ON.
    _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_ONEAPI=OFF \
                    -DWITH_CYCLES_ONEAPI_BINARIES=OFF )
  fi
  [[ -f /opt/bin/clang ]] && _CMAKE_FLAGS+=( -DLLVM_ROOT_DIR=/opt/lib )

  # determine whether we can precompile CUDA kernels
  _CUDA_PKG=$(pacman -Qq cuda 2>/dev/null) || true
  if [ "$_CUDA_PKG" != "" ]; then
    # https://wiki.blender.org/wiki/Building_Blender/GPU_Binaries
    _CMAKE_FLAGS+=( -DWITH_CYCLES_CUDA_BINARIES=ON \
                    -DWITH_COMPILER_ASAN=OFF \
                    -DCMAKE_CUDA_HOST_COMPILER=${NVCC_CCBIN} ) # defined in /etc/profile.d/cuda.sh
  fi

  # check for materialx
  _MX_PKG=$(pacman -Qq materialx 2>/dev/null) || true
  if [ "$_MX_PKG" != "" ]; then
    _CMAKE_FLAGS+=( -DWITH_MATERIALX=ON )
    PATH="/usr/materialx:$PATH"  
  fi

  _USD_PKG=$(pacman -Qq usd 2>/dev/null) || true
  if [ "$_USD_PKG" != "" ]; then
    _CMAKE_FLAGS+=( -DWITH_USD=ON )
    PATH="/usr/share/usd:$PATH"  
  fi

  # check for optix
  _OPTIX_PKG=$(pacman -Qq optix8 2>/dev/null) || true
  if [ "$_OPTIX_PKG" != "" ]; then
      _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_OPTIX=ON \
                      -DOPTIX_ROOT_DIR=/opt/optix )
  fi

  # check for open image denoise
  _OIDN_PKG=$(pacman -Qq openimagedenoise 2>/dev/null) || true
  if [ "$_OIDN_PKG" != "" ]; then
      _CMAKE_FLAGS+=( -DWITH_OPENIMAGEDENOISE=ON )
  fi

  if [ -d /opt/rocm/bin ]; then
      _CMAKE_FLAGS+=( -DWITH_CYCLES_HIP_BINARIES=ON
                      -DWITH_CYCLES_DEVICE_HIP=ON
                      -DWITH_CYCLES_DEVICE_HIPRT=ON
                      -DWITH_CYCLES_HYDRA_RENDER_DELEGATE:BOOL=FALSE
                    )
  else
      # Because some defaults are ON.
      _CMAKE_FLAGS+=( -DWITH_CYCLES_HIP_BINARIES=OFF
                      -DWITH_CYCLES_DEVICE_HIP=OFF
                      -DWITH_CYCLES_DEVICE_HIPRT=OFF
                    )
  fi

  if [[ -f "$srcdir/blender/CMakeCache.txt" && -z "$KEEP_CMAKE_CACHE" ]]; then
    rm "$srcdir/blender/CMakeCache.txt"
  fi

  NUMPY_PY_INCLUDE=/usr/lib/python${_pyver}/site-packages/numpy/_core/include/
  [[ -d "$NUMPY_PY_INCLUDE" ]] && (
    _CMAKE_FLAGS+=( -DNUMPY_INCLUDE_DIR="$NUMPY_PY_INCLUDE" );
    __CFLAGS="$CFLAGS -I$NUMPY_PY_INCLUDE"
    __CXXFLAGS="$CXXFLAGS -I$NUMPY_PY_INCLUDE"
    export CFLAGS="$__CFLAGS"
    export CXXFLAGS="$__CXXFLAGS"
  )

  export CFLAGS="$CFLAGS -fno-lto"
  export CXXFLAGS="$CXXFLAGS -fno-lto"
  # Who even knows why this is needed
  export CFLAGS="$CFLAGS -lSPIRV -lSPIRV-Tools -lSPIRV-Tools-opt -lSPIRV-Tools-link -lSPIRV-Tools-reduce -lSPIRV-Tools-shared -lglslang"
  export CXXFLAGS="$CXXFLAGS -lSPIRV -lSPIRV-Tools -lSPIRV-Tools-opt -lSPIRV-Tools-link -lSPIRV-Tools-reduce -lSPIRV-Tools-shared -lglslang"
  _CMAKE_FLAGS+=( -DCMAKE_C_FLAGS="$CFLAGS" );
  _CMAKE_FLAGS+=( -DCMAKE_CXX_FLAGS="$CXXFLAGS" );

  CMAKE_CMD=(CUDAHOSTCXX="$CUDAHOSTCXX" cmake -B "$srcdir/build" --fresh
                -C "${srcdir}/blender/build_files/cmake/config/blender_release.cmake"
                -GUnix\ Makefiles
                -DCMAKE_INSTALL_PREFIX=/usr
                -DCMAKE_INSTALL_PREFIX_WITH_CONFIG="${pkgdir}/usr"
                -DCMAKE_SKIP_INSTALL_RPATH=ON
                -DCMAKE_SKIP_BUILD_RPATH=ON
                -DCMAKE_BUILD_TYPE=Release
                -DWITH_INSTALL_PORTABLE=OFF
                -DWITH_LIBS_PRECOMPILED=OFF
                -DWITH_STATIC_LIBS=OFF
                -DXR_OPENXR_SDK_ROOT_DIR=/usr
                -DPYTHON_VERSION="${_pyver}"
                "${_CMAKE_FLAGS[@]}"
  ) #> "$srcdir/../cmake_out"
                #--trace-expand \

  MAKE_CMD="make ${MAKEFLAGS:--j1} blender"

  USING_MAKEPKG_CG="$(systemctl --user -t slice | grep -o makepkg-cg-`id -u`-'[[:digit:]]\+'.slice'[[:space:]]\+'loaded'[[:space:]]\+'active)" || true
  MAKEPKG_CG_WARNING=$(
    cat << 'EOF'
If you use systemd, consider trying `makepkg-cg`.
This build is otherwise very likely to use more RAM than
the system has, especially with a high `-j`!
EOF
  )
  [[ -z "$USING_MAKEPKG_CG" ]] && warning "$MAKEPKG_CG_WARNING"
  
  cd blender
  env "${CMAKE_CMD[@]}"
  cd ../build
  env CFLAGS="$CFLAGS" CXXFLAGS="$CXXFLAGS" $MAKE_CMD
}

package() {
  _suffix=${pkgver%%.r*}
  cd "$srcdir/build"
  sed -ie 's/\(file(INSTALL\)\(.*blender\.1"\))/#\1\2)/' source/creator/cmake_install.cmake
  sed -ie 's|/usr/lib/python/|/usr/lib64/python3.13/|g' source/creator/cmake_install.cmake
  BLENDER_SYSTEM_RESOURCES="${pkgdir}/usr/share/blender/${_suffix}" make DESTDIR="$pkgdir" install
  #find . -name 'cmake_install.cmake' -exec sed -i -e 's|/usr/lib64/|'"$pkgdir"'/usr/lib/|g' {} \;
  #cmake --install . --prefix "$pkgdir/usr"

  if [[ -e "$pkgdir/usr/share/blender/${_suffix}/scripts/addons/cycles/lib/" ]] ; then
    # make sure the cuda kernels are not stripped
    chmod 444 "$pkgdir"/usr/share/blender/${_suffix}/scripts/addons/cycles/lib/*
  fi
}

# vim: syntax=bash:et:ts=2:sw=2
