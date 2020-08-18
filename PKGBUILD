# Maintainer: 9l

# CJKTTY patch
# https://github.com/Gentoo-zh/linux-cjktty
# https://github.com/torvalds/linux/compare/v5.6...Gentoo-zh:5.6-utf8
_CJKTTY_LLL_VER=5.6
_CJKTTY_PATCH_FILE=0001-linux-cjktty-${_CJKTTY_LLL_VER}.patch
_CJKTTY_PATCH_URL="https://github.com/torvalds/linux/compare/v${_CJKTTY_LLL_VER}...Gentoo-zh:${_CJKTTY_LLL_VER}-utf8.patch"
_CJKTTY_PATCH="${_CJKTTY_PATCH_FILE}::${_CJKTTY_PATCH_URL}"

pkgbase=linux-cjktty
pkgver=${_CJKTTY_LLL_VER}.utf8
pkgrel=1
pkgdesc='Linux kernel with CJKTTY patch'
url="https://github.com/Gentoo-zh/linux-cjktty"
arch=(x86_64)
license=(GPL2)
makedepends=(
  bc kmod libelf pahole
  xmlto python-sphinx python-sphinx_rtd_theme graphviz imagemagick
  git
)
options=('!strip')
_srcname=linux-${pkgver%.*}
source=(
  https://mirrors.edge.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/linux-${pkgver%.*}.tar.xz
  config         # the main kernel config file
  0000-sphinx-workaround.patch
  ${_CJKTTY_PATCH} # CJKTTY patch file
  0002-gcc-plugins-drop-support-for-GCC-4.7.patch
  0003-gcc-common.h-Update-for-GCC-10.patch
  0004-ZEN-Add-sysctl-and-CONFIG-to-disallow-unprivileged-C.patch
  0005-drivers-tty-vt-vt.c.patch
)
validpgpkeys=(
  'ABAF11C65A2970B130ABE3C479BE3E4300411886'  # Linus Torvalds
  '647F28654894E3BD457199BE38DBBDC86092693E'  # Greg Kroah-Hartman
  '65EEFE022108E2B708CBFCF7F9E712E59AF5F22A'  # Daniel Micay
  'E240B57E2C4630BA768E2F26FC1B547C8D8172C8'  # Levente Polyak
)
sha256sums=('e342b04a2aa63808ea0ef1baab28fc520bd031ef8cf93d9ee4a31d4058fcb622'
            'b912c39b24f92e2d99384cfccd80a58ad8e2f9887563b153dbec77fbcba60f8b'
            '8cb21e0b3411327b627a9dd15b8eb773295a0d2782b1a41b2a8839d1b2f5778c'
            'd871ef6a82f7e7a9782369d16aeb2c2e9f3923726481252bf00437e4b650890a'
            '118531186e7069b006d48fdfb2dbd9f28ee6d01cbfaacb007d8f44e8e76a57e1'
            'd2ee933dd10dee475746a7e9081bca3ebdae8c6fb631f458d8f0d4987c9a0845'
            '967a285fa215780615f4cd5aa1ec05a35816a471b5585c776feccd16fd70e18a'
            'd0f5f01491e31732865ed8e27939ca4270111b8a5e07a7975103ed4a63a22a3b')

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

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
    [[ $src = 0*.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  echo "Setting config..."
  cp ../config .config

  # FS#66613
  # https://bugzilla.kernel.org/show_bug.cgi?id=207173#c6
  sed -i -e 's/CONFIG_KVM_WERROR=y/# CONFIG_KVM_WERROR is not set/' ./.config

  make olddefconfig

  echo "Setting CJKTTY config..."
  sed -i -e 's/CONFIG_FRAMEBUFFER_CONSOLE_ROTATION=y/# CONFIG_FRAMEBUFFER_CONSOLE_ROTATION is not set/' ./.config
  sed -i -e 's/# CONFIG_FONT_8x8 is not set/CONFIG_FONT_8x8=y/' ./.config
  sed -i -e 's/# CONFIG_FONT_16x16_CJK is not set/CONFIG_FONT_16x16_CJK=y/' ./.config
  make oldconfig

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  make all
  make htmldocs
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(coreutils kmod initramfs)
  optdepends=('crda: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices')
  provides=(VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE)
  replaces=(virtualbox-guest-modules-arch wireguard-arch)

  cd $_srcname
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" modules_install

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

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
    case "$(file -bi "$file")" in
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
