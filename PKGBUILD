# AArch64 multi-platform
# Maintainer: Samuel (kwankiu)
# Contributor: Jat-faan Wong, Guoxin "7Ji" Pu

pkgbase=linux-aarch64-rockchip-bsp6.1-armbian-git
pkgname=("${pkgbase}"{,-headers})
pkgver=6.1.99.r1277473.72269e53
pkgrel=1
arch=('aarch64')
license=('GPL2')
url="https://github.com/armbian"
_desc="Armbian Rockchip BSP6.1 Kernel" 
makedepends=('cpio' 'xmlto' 'docbook-xsl' 'kmod' 'inetutils' 'bc' 'git' 'dtc')
options=('!strip')
_srcname='linux-rockchip'
_config='linux-rk35xx-vendor'
source=(
  "git+${url}/${_srcname}.git#branch=rk-6.1-rkr5"
  "https://raw.githubusercontent.com/armbian/build/main/config/kernel/${_config}.config"
  'local.config'
  "001-intel_be200.patch::https://patch-diff.githubusercontent.com/raw/Joshua-Riek/linux-rockchip/pull/34.patch"
  "002-gpu_pll_tune.patch::https://github.com/hbiyik/linux/commit/e4fd428dd34fe13cbd5fa6ed79e2f787bc7655b0.patch"
)

sha512sums=('SKIP'
            '6f5cafe346bf4eed533a6647929d27cd9306095f37f4701d3476cd2d69b85c1b3835de80788dc659920bd534eab22f6a5e818c5ca59fd71716cb893cbce35e57'
            '286f7e585eff92da3562d90b0c8a568df9aa67074697cd05c425f7e5bc267e09f28d0a139037cb4c34c27d3ddf8388e6d7b9311f7129a788f4685db7daf7f687'
            '3670998a0fe640113fa04bc4e24682812343ed997457885711ba583ba7c534b02d2ea0634b7dc282ba525c9b1722df7b3cd29464a749227120b678e5dcb67276'
            'SKIP')

pkgver() {
  cd "${_srcname}"
  printf "%s.%s%s%s.r%s.%s" \
    "$(grep '^VERSION = ' Makefile|awk -F' = ' '{print $2}')" \
    "$(grep '^PATCHLEVEL = ' Makefile|awk -F' = ' '{print $2}')" \
    "$(grep '^SUBLEVEL = ' Makefile|awk -F' = ' '{print $2}'|grep -vE '^0$'|sed 's/.*/.\0/')" \
    "$(grep '^EXTRAVERSION = ' Makefile|awk -F' = ' '{print $2}'|tr -d -|sed -E 's/rockchip[0-9]+//')" \
    "$(git rev-list --count HEAD)" \
    "$(git rev-parse --short=8 HEAD)"
}

prepare() {
  cd "${_srcname}"
  
  rm -rf localversion*
  echo "Setting version..."
  echo "-rockchip" > localversion.10-pkgname
  #echo "-r$(git rev-list --count HEAD)" > localversion.20-revision

  for p in $srcdir/*.patch; do
    echo "Patching with ${p}"
    patch -p1 -N -i $p
  done

  echo "Preparing config..."
  scripts/kconfig/merge_config.sh -m ../${_config}.config ../local.config
}

build() {
  cd "${_srcname}"

  make olddefconfig prepare
  make -s kernelrelease > version

  unset LDFLAGS
  make ${MAKEFLAGS} Image modules
  make ${MAKEFLAGS} DTC_FLAGS="-@" dtbs
}

_package() {
  pkgdesc="The ${_srcname} kernel, ${_desc}"
  depends=('coreutils' 'kmod' 'initramfs')
  optdepends=('wireless-regdb: to set the correct wireless channels of your country')

  cd "${_srcname}"
  
  # install dtbs
  make INSTALL_DTBS_PATH="${pkgdir}/boot/dtbs/${pkgbase}" dtbs_install

  # install modules
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # copy kernel
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # remove reference to build host
  rm -f "${_dir_module}/"{build,source}

  # used by mkinitcpio to name the kernel
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the ${_srcname} kernel, ${_desc}"
  depends=("python")

  cd "${_srcname}"
  local builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map version
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "$builddir" -a scripts

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/arm64" -a arch/arm64/include
  install -Dt "$builddir/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s
  mkdir -p "$builddir/arch/arm"
  cp -t "$builddir/arch/arm" -a arch/arm/include

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

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */arm64/ || $arch == */arm/ ]] && continue
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
  done < <(find "$builddir" -type f -perm -u+x ! -name -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#${pkgbase}}
  }"
done
