# Maintainer: nibon7 <nibon7@163.com>

pkgbase=linux-lto
pkgver=6.2.8.lto1
pkgrel=1
pkgdesc='Linux'
url="https://www.kernel.org"
arch=(x86_64)
license=(GPL2)
makedepends=(
  bc libelf pahole cpio perl tar xz clang llvm lld rust-bindgen
  xmlto python-sphinx graphviz imagemagick texlive-latexextra
)
options=('!strip')
_srcname=linux-6.2.1
source=(
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.2.1.tar.xz"
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.2.1-2.xz"
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.2.2-3.xz"
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.2.3-4.xz"
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.2.4-5.xz"
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.2.5-6.xz"
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.2.6-7.xz"
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.2.7-8.xz"
  "https://raw.githubusercontent.com/graysky2/kernel_compiler_patch/master/more-uarches-for-kernel-5.17+.patch"
  config         # the main kernel config file
)
sha256sums=('2fcc07e1c90ea4ce148f50f9beeb0dca0b6e4b379a768de8abc7a4a26f252534'
            'c2ae3a65db0937d661a9ef4e9ff6e86759a0813591aff2888c1af297b1ba2d0b'
            '348ea838d17fc47d1dbfcef462f869a4e5cba30cf167bdd65aac5b9630bce048'
            '5aaa58e180086e942790774e719f92de170dcbaccc0af3f6d870b2418ad65e4a'
            'baa2e56fb9dceb773d24bdd5952021325d30d9605593bb120cd838abfc3abbab'
            'e01f9e9698039f6e12dadb584b62f9c8f561a6ec8758b7e097a253364bbb7020'
            '34889b2031b9eb449d2aa6dce43cf3b724baff234207b76fc9e45a28d2244e08'
            'b1007e3cda6eb9de564498fae9a184bfd891abaf7e0523ca0a02c61564f21349'
            'ba133fdda4dcc62de10792ae1d8149ce4a18d13a6ad808926e8b2d94b72071c3'
            '499b1e8729d4f650b9ae206031aa266349b2a58fd2c50dd43df7969200471b60')

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"
export CC=clang
export LLVM=1

prepare() {
  cd $_srcname

  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

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
  rustc --version
  sed -e 's/--blacklist-type/--blocklist-type/g' \
    -e 's/--whitelist-var/--allowlist-var/g' \
    -e 's/--whitelist-function/--allowlist-function/g' \
    -i rust/Makefile

  echo "Setting config..."
  cp ../config .config
  if [[ "x$NATIVE_INTEL" != "x" ]]; then
      sed -i 's/^# CONFIG_MNATIVE_INTEL is not set/CONFIG_MNATIVE_INTEL=y/' .config
      sed -i 's/^CONFIG_MNATIVE_AMD=y/# CONFIG_MNATIVE_AMD is not set/' .config
  fi
  make olddefconfig
  diff -u ../config .config || :

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  make htmldocs all
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(coreutils kmod)
  optdepends=('wireless-regdb: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices')
  provides=(VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE KSMBD-MODULE)
  replaces=(virtualbox-guest-modules-arch wireguard-arch)

  cd $_srcname
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$pkgdir/boot/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
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

pkgname=("$pkgbase" "$pkgbase-headers" "$pkgbase-docs")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
