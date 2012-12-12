# Linux 3.7 for Marvell Kirkwood devices
# tested on Seagate GoFlex Net
# Based https://github.com/archlinuxarm/PKGBUILDs/tree/master/core/linux-kirkwood
# Look there for original authors/maintainers etc

buildarch=2

pkgbase=linux-kirkwood
pkgname=('linux-kirkwood' 'linux-headers-kirkwood')
#pkgname=linux-test       # Build kernel with a different name
_kernelname=${pkgname#linux}
_basekernel=3.7
pkgver=${_basekernel}
pkgrel=5
cryptover=1.5
arch=('arm')
url="http://www.kernel.org/"
license=('GPL2')
makedepends=('xmlto' 'docbook-xsl' 'uboot-mkimage')
options=('!strip')
source=("ftp://ftp.kernel.org/pub/linux/kernel/v3.x/linux-${_basekernel}.tar.bz2"
        'ebbes-kirkwood.patch'
        'support.patch'
        'config'
        'mach-types::http://www.arm.linux.org.uk/developer/machines/download.php'
        'change-default-console-loglevel.patch'
        'usb-add-reset-resume-quirk-for-several-webcams.patch'
        "http://download.gna.org/cryptodev-linux/cryptodev-linux-${cryptover}.tar.gz")
md5sums=('5323f3faadd051e83af605a63be5ea2e'
         'c2d81cd45dc78e92af4f13fb8e5065b6'
         'f5d3635da03cb45904bedd69b47133de'
         '9701ac075585ff9dbae947a121e9b2b7'
         '9506a43fff451fda36d5d7b1f5eaed04'
         '9d3c56a4b999c8bfbd4018089a62f662'
         'd00814b57448895e65fbbc800e8a58ba'
         '3a4b8d23c1708283e29477931d63ffb8')

build() {
  cd "${srcdir}/linux-${_basekernel}"

  # Add the USB_QUIRK_RESET_RESUME for several webcams
  # FS#26528
  patch -Np1 -i "${srcdir}/usb-add-reset-resume-quirk-for-several-webcams.patch"

  # Add Arch Linux ARM patch for ARMv5te plug computers, requested additional support, mach-types
  patch -Np1 -i "${srcdir}/ebbes-kirkwood.patch"
  patch -Np1 -i "${srcdir}/support.patch"
  cp "${srcdir}/mach-types" arch/arm/tools

  # add latest fixes from stable queue, if needed
  # http://git.kernel.org/?p=linux/kernel/git/stable/stable-queue.git

  # set DEFAULT_CONSOLE_LOGLEVEL to 4 (same value as the 'quiet' kernel param)
  # remove this when a Kconfig knob is made available by upstream
  # (relevant patch sent upstream: https://lkml.org/lkml/2011/7/26/227)
  patch -Np1 -i "${srcdir}/change-default-console-loglevel.patch"

  cat "${srcdir}/config" > ./.config

  # set extraversion to pkgrel
  sed -ri "s|^(EXTRAVERSION =).*|\1 -${pkgrel}|" Makefile

  # don't run depmod on 'make install'. We'll do this ourselves in packaging
  sed -i '2iexit 0' scripts/depmod.sh

  # get kernel version
  make prepare

  # load configuration
  # Configure the kernel. Replace the line below with one of your choice.
  #make menuconfig # CLI menu for configuration
  #make nconfig # new CLI menu for configuration
  #make xconfig # X-based configuration
  #make oldconfig # using old config from previous kernel version
  # ... or manually edit .config

  # Copy back our configuration (use with new kernel version)
  #cp ./.config ../${_basekernel}.config

  ####################
  # stop here
  # this is useful to configure the kernel
  #msg "Stopping build"
  #return 1
  ####################

  #yes "" | make config

  # build!
  make ${MAKEFLAGS} uImage modules

  # build cryptodev module
  cd "${srcdir}/cryptodev-linux-${cryptover}"
  make KERNEL_DIR="${srcdir}/linux-${_basekernel}"
}

package_linux-kirkwood() {
  pkgdesc="The Linux Kernel and modules - Marvell Kirkwood"
  groups=('base')
  depends=('coreutils' 'linux-firmware' 'module-init-tools>=3.16' 'mkinitcpio>=0.7')
  optdepends=('crda: to set the correct wireless channels of your country')
  provides=('kernel26' 'cryptodev_friendly' 'linux=${pkgver}')
  conflicts=('linux')
  install=${pkgname}.install

  cd "${srcdir}/linux-${_basekernel}"

  KARCH=arm

  # get kernel version
  _kernver="$(make kernelrelease)"

  mkdir -p "${pkgdir}"/{lib/modules,lib/firmware,boot}
  make INSTALL_MOD_PATH="${pkgdir}" modules_install
  cp arch/$KARCH/boot/uImage "${pkgdir}/boot/uImage"

  # set correct depmod command for install
  sed \
    -e  "s/KERNEL_NAME=.*/KERNEL_NAME=${_kernelname}/g" \
    -e  "s/KERNEL_VERSION=.*/KERNEL_VERSION=${_kernver}/g" \
    -i "${startdir}/${pkgname}.install"

  # remove build and source links
  rm -f "${pkgdir}"/lib/modules/${_kernver}/{source,build}
  # remove the firmware
  rm -rf "${pkgdir}/lib/firmware"
  # gzip -9 all modules to save 100MB of space
  find "${pkgdir}" -name '*.ko' -exec gzip -9 {} \;
  # make room for external modules
  ln -s "../extramodules-${_basekernel}-${_kernelname:-EBBES}" "${pkgdir}/lib/modules/${_kernver}/extramodules"
  # add real version for building modules and running depmod from post_install/upgrade
  mkdir -p "${pkgdir}/lib/modules/extramodules-${_basekernel}-${_kernelname:-EBBES}"
  echo "${_kernver}" > "${pkgdir}/lib/modules/extramodules-${_basekernel}-${_kernelname:-EBBES}/version"

  # install cryptodev module
  cd "${srcdir}/cryptodev-linux-${cryptover}"
  make -C "${srcdir}/linux-${_basekernel}" INSTALL_MOD_PATH="${pkgdir}" SUBDIRS=`pwd` modules_install

  cd "${srcdir}/linux-${_basekernel}"

  # Now we call depmod...
  depmod -b "$pkgdir" -F System.map "$_kernver"

  # move module tree /lib -> /usr/lib
  mkdir -p "${pkgdir}/usr"
  mv "$pkgdir/lib" "$pkgdir/usr"
}

package_linux-headers-kirkwood() {
  pkgdesc="Header files and scripts for building modules for linux kernel - Marvell Kirkwood"
  provides=('kernel26-headers' 'linux-headers=${pkgver}')
  conflicts=('kernel26-headers' 'linux-headers')

  install -dm755 "${pkgdir}/usr/lib/modules/${_kernver}"

  cd "${pkgdir}/usr/lib/modules/${_kernver}"
  ln -sf ../../../src/linux-${_kernver} build

  cd "${srcdir}/linux-${_basekernel}"
  install -D -m644 Makefile \
    "${pkgdir}/usr/src/linux-${_kernver}/Makefile"
  install -D -m644 kernel/Makefile \
    "${pkgdir}/usr/src/linux-${_kernver}/kernel/Makefile"
  install -D -m644 .config \
    "${pkgdir}/usr/src/linux-${_kernver}/.config"

  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/include"
  make headers_install INSTALL_HDR_PATH="${pkgdir}/usr/src/linux-${_kernver}"

  # Clean up unneeded files
  find "${pkgdir}" -name "..install.cmd" -delete

  # copy arch includes for external modules
  mkdir -p ${pkgdir}/usr/src/linux-${_kernver}/arch/arm
  cp -a arch/arm/include ${pkgdir}/usr/src/linux-${_kernver}/arch/arm/
  mkdir -p ${pkgdir}/usr/src/linux-${_kernver}/arch/arm/mach-kirkwood   
  cp -a arch/arm/mach-kirkwood/include ${pkgdir}/usr/src/linux-${_kernver}/arch/arm/mach-kirkwood/
  mkdir -p ${pkgdir}/usr/src/linux-${_kernver}/arch/arm/plat-orion
  cp -a arch/arm/plat-orion/include ${pkgdir}/usr/src/linux-${_kernver}/arch/arm/plat-orion/

  # copy files necessary for later builds, like nvidia and vmware
  cp Module.symvers "${pkgdir}/usr/src/linux-${_kernver}"
  cp -a scripts "${pkgdir}/usr/src/linux-${_kernver}"

  # fix permissions on scripts dir
  chmod og-w -R "${pkgdir}/usr/src/linux-${_kernver}/scripts"
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/.tmp_versions"

  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/arch/arm/kernel"

  cp arch/arm/Makefile "${pkgdir}/usr/src/linux-${_kernver}/arch/arm/"

  cp arch/arm/kernel/asm-offsets.s "${pkgdir}/usr/src/linux-${_kernver}/arch/arm/kernel/"

  # add docbook makefile
  install -D -m644 Documentation/DocBook/Makefile \
    "${pkgdir}/usr/src/linux-${_kernver}/Documentation/DocBook/Makefile"

  # add xfs and shmem for aufs building
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/fs/xfs"
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/mm"
  cp fs/xfs/xfs_sb.h "${pkgdir}/usr/src/linux-${_kernver}/fs/xfs/xfs_sb.h"

  # copy in Kconfig files
  for i in `find . -name "Kconfig*"`; do
    mkdir -p "${pkgdir}"/usr/src/linux-${_kernver}/`echo ${i} | sed 's|/Kconfig.*||'`
    cp ${i} "${pkgdir}/usr/src/linux-${_kernver}/${i}"
  done

  chown -R root.root "${pkgdir}/usr/src/linux-${_kernver}"
  find "${pkgdir}/usr/src/linux-${_kernver}" -type d -exec chmod 755 {} \;

  # strip scripts directory
  find "${pkgdir}/usr/src/linux-${_kernver}/scripts" -type f -perm -u+w 2>/dev/null | while read binary ; do
    case "$(file -bi "${binary}")" in
      *application/x-sharedlib*) # Libraries (.so)
        /usr/bin/strip ${STRIP_SHARED} "${binary}";;
      *application/x-archive*) # Libraries (.a)
        /usr/bin/strip ${STRIP_STATIC} "${binary}";;
      *application/x-executable*) # Binaries
        /usr/bin/strip ${STRIP_BINARIES} "${binary}";;
    esac
  done

  # remove unneeded architectures
  rm -rf "${pkgdir}"/usr/src/linux-${_kernver}/arch/{alpha,arm26,avr32,blackfin,cris,frv,h8300,ia64,m32r,m68k,m68knommu,mips,microblaze,mn10300,parisc,powerpc,ppc,s390,sh,sh64,sparc,sparc64,um,v850,x86,xtensa}

  # install cryptodev header
  cd "${srcdir}/cryptodev-linux-${cryptover}"
  install -D crypto/cryptodev.h "${pkgdir}/usr/src/linux-${_kernver}/crypto/cryptodev.h"
}
