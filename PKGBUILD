# Maintainer: Yurii Kolesykov <root@yurikoles.com>
# based on core/linux: Jan Alexander Steffens (heftig) <jan.steffens@gmail.com>

pkgbase=linux-amd-staging-drm-next-git
pkgdesc='Linux kernel with AMDGPU DC patches'

_branch=amd-staging-drm-next
_kernelname=${pkgbase#linux}
pkgver=5.10.949911.bc32b65255ee
pkgrel=4
arch=(x86_64)
url='https://gitlab.freedesktop.org/drm/amd'
license=(GPL2)
makedepends=(
  bc kmod libelf pahole
  xmlto python-sphinx python-sphinx_rtd_theme graphviz imagemagick
  git
)
options=('!strip')
_srcname=linux-mcoffin
source=(
  "${_srcname}::git+git://github.com/mcoffin/linux.git#branch=${_branch}"
  config         # the main kernel config file
  sphinx-workaround.patch
  0001-drm-amdgpu-pm-Account-for-extra-separator-characters.patch
  0002-amdgpu-debug.patch
  0001-futex-Implement-mechanism-to-wait-on-any-of-several-.patch
  0002-selftests-futex-Add-FUTEX_WAIT_MULTIPLE-timeout-test.patch
  0003-selftests-futex-Add-FUTEX_WAIT_MULTIPLE-wouldblock-t.patch
  0004-selftests-futex-Add-FUTEX_WAIT_MULTIPLE-wake-up-test.patch
  0005-futex-Add-Proton-compatibility-code.patch
  0006-futex-Fix-FUTEX_WAIT_MULTIPLE-for-5.10.patch
  0001-kvm-Squelch-uninitialized-variable-warning.patch
)
sha256sums=('SKIP'
            '623601ed9d7879dd9dba1cd50fc8051f9db508b49b4fc0c47c5a9eb9165fc04e'
            '8cb21e0b3411327b627a9dd15b8eb773295a0d2782b1a41b2a8839d1b2f5778c'
            'f6cf3b90d0c66f7e53eb668a8eab409a8ad5943afbf777a604a3d15c2616216d'
            '40a52fd8a7576fbb99606d2a5f8c9826674ece8b5a0ac128244ad8c1fd0ed316'
            '26ddbd0d075a09f5f5e192f3ba54af33b38c672d366157089565fc78efcfc956'
            '9d40f04e3c254d7f3fbfd05f31d9e8cea5ef0f04989ca89faabfb31cb2fd033c'
            'da3061be1786c39597ac274bd2eb89be62364fe91990346c2aded5279e563047'
            '0c60418808fc75bca9e6f9fa454339f873ae8590bce42f1870e1b1291893323d'
            'd60bc6516c9e5d08d15c160467b4b0476ff26e067ba35609e51b3187e07ae235'
            'bd7b9b1338e863ba7fb8752a7462f36ccd9abcddf2f4b6184ac636cca4f96561'
            'b44f134944ab32d489135197bf96715ac2ca2f1c7890bb0b6133f259933c7eb3')

pkgver() {
  cd "${_srcname}"
  local version="$(grep \^VERSION Makefile|cut -d"=" -f2|cut -d" " -f2)"
  local patch="$(grep \^PATCHLEVEL Makefile|cut -d"=" -f2|cut -d" " -f2)"
  patch=$(( $patch + 1 ))

  echo $version.$patch.$(git rev-list --count HEAD).$(git rev-parse --short HEAD)
}

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
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  echo "Setting config..."
  cp ../config .config
  make olddefconfig

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  make all
  make htmldocs
}

_package-git() {
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
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers-git() {
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

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

_package-docs-git() {
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

_product=linux-amd-staging-drm-next
pkgname=("$pkgbase" "$_product-headers-git" "$_product-docs-git")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$_product}")
    _package${_p#$_product}
  }"
done
