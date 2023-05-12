# Maintainer: nibon7 <nibon7@163.com>

pkgbase=linux-lto
pkgver=6.3.2.lto1
pkgrel=1
pkgdesc='Linux'
url="https://www.kernel.org"
arch=(x86_64)
license=(GPL2)
makedepends=(
  bc
  cpio
  gettext
  libelf
  pahole
  perl
  tar
  xz

  # rust
  # rust-bindgen
  clang
  lld
  llvm

  # htmldocs
  graphviz
  imagemagick
  python-sphinx
  texlive-latexextra
  xmlto
)
options=('!strip')
_srcname=linux-6.3.1
source=(
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.3.1.tar.xz"
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.3.1-2.xz"
  "https://raw.githubusercontent.com/graysky2/kernel_compiler_patch/master/more-uarches-for-kernel-5.17+.patch"
  config # the main kernel config file
)
sha256sums=('78620fb4a7d5e0db1d4eb8d5b1c6e207ba5d19564efa63967a59b6daf89b3f2a'
            'a6cdb915b39452a69b2196ca731947b9ea26b837e51056164a62107584116d91'
            'ba133fdda4dcc62de10792ae1d8149ce4a18d13a6ad808926e8b2d94b72071c3'
            'd0f632409ba9557772f55b41d0debafc9f6ae46a7858af9b7c4c551e5406b03a')

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"
export CC=clang
export LLVM=1

_make() {
  test -s version
  make KERNELRELEASE="$(<version)" "$@"
}

prepare() {
  cd $_srcname

  echo "Setting version..."
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname
  make defconfig
  make -s kernelrelease > version
  make mrproper

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    if [[ $src = *.patch ]]; then
        echo "Applying patch $src..."
        patch -Np1 < "../$src"
    elif [[ $src = patch-*.xz ]]; then
        echo "Applying patch $src..."
        xzcat "../$src" | patch -Np1
    else
        continue
    fi
  done

  echo "Setting rust toolchain..."
  cat > rust-toolchain.toml << EOF
[toolchain]
channel = "1.63.0"
components = [ "rust-src" ]
EOF

  echo "Setting config..."
  cp ../config .config
  if [[ "x$NATIVE_INTEL" != "x" ]]; then
      sed -i 's/^# CONFIG_MNATIVE_INTEL is not set/CONFIG_MNATIVE_INTEL=y/' .config
      sed -i 's/^CONFIG_MNATIVE_AMD=y/# CONFIG_MNATIVE_AMD is not set/' .config
  fi
  _make olddefconfig
  diff -u ../config .config || :

  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  _make htmldocs all
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(
    coreutils
    kmod
  )
  optdepends=(
    'wireless-regdb: to set the correct wireless channels of your country'
    'linux-firmware: firmware images needed for some devices'
  )
  provides=(
    KSMBD-MODULE
    VIRTUALBOX-GUEST-MODULES
    WIREGUARD-MODULE
  )
  replaces=(
    virtualbox-guest-modules-arch
    wireguard-arch
  )

  cd $_srcname
  local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(_make -s image_name)" "$pkgdir/boot/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  _make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install  # Suppress depmod

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # required when STACK_VALIDATION is enabled
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # required when DEBUG_INFO_BTF_MODULES is enabled
  #install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Installing Rust development files..."
  cp -r -t "$builddir" rust

  echo "Removing generated files for rust..."
  find "$builddir/rust" -type f -name 'built-in.a' -printf 'Removing %P\n' -delete
  find "$builddir/rust" -type f -name 'modules.order' -printf 'Removing %P\n' -delete
  find "$builddir/rust" -type f -name '*.cmd' -printf 'Removing %P\n' -delete

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

_package-docs() {
  pkgdesc="Documentation for the $pkgdesc kernel"

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing documentation..."
  local src dst
  while read -rd '' src; do
    dst="${src#Documentation/}"
    dst="$builddir/Documentation/${dst#output/}"
    install -Dm644 "$src" "$dst"
  done < <(find Documentation -name '.*' -prune -o ! -type d -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/share/doc"
  ln -sr "$builddir/Documentation" "$pkgdir/usr/share/doc/$pkgbase"
}

pkgname=(
  "$pkgbase"
  "$pkgbase-headers"
  "$pkgbase-docs"
)
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
