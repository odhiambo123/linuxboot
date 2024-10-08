# Makefile for Ampere Jade.
# This is only partially done. It is derived from other ARM
# systems. 
# The goal is to create a kernel we can start from the UEFI shell. At present,
# when we do this, we get this:
# FS3:\EFI\opensuse\> Image
# EFI stub: Booting Linux Kernel...
# EFI stub: Generating empty DTB
# EFI stub: Exiting boot services and installing virtual address map...

default: flashkernel

flash: tiny.bin

flashtest: testflashkernel

usefultargets:
	echo fetch, uroot, kernel, or image.bin

flashinitramfs.cpio.lzma: flashinitramfs.cpio
	lzma -f -k $<

flashinitramfs.cpio: Makefile
	GO111MODULE=off GOARCH=arm64 u-root -uroot-source ${GOPATH}/src/github.com/u-root/u-root -uinitcmd=systemboot -o $@ core \
		github.com/u-root/u-root/cmds/boot/systemboot \
		github.com/u-root/u-root/cmds/boot/localboot \
		github.com/u-root/u-root/cmds/boot/fbnetboot \
		github.com/u-root/u-root/cmds/exp/acpicat \
		github.com/u-root/u-root/cmds/exp/acpigrep \
		github.com/u-root/u-root/cmds/exp/disk_unlock \
		github.com/u-root/u-root/cmds/exp/dmidecode \
		github.com/u-root/u-root/cmds/exp/ipmidump \
		github.com/u-root/cpu/cmds/cpud

# this target builds an initramfs with only one program, the cpu server.
# It depends on the kernel setting the IP address.
# You need ip=dhcp OR ip=fixed.ip.address.here in the command line
cpu.cpio.lzma: Makefile
	GO111MODULE=off GOARCH=arm64 go run github.com/u-root/u-root -o cpu.cpio -build=bb -initcmd=cpud -files ~/.ssh/cpu_rsa.pub:key.pub \
		-defaultsh="" \
		github.com/u-root/cpu/cmds/cpud
	lzma -f -k cpu.cpio

# this target builds an initramfs with all of u-root, AND all of /bin, /usr/bin, /sbin, /usr/sbin, /etc
# the intent is  to demonstrate the capability of http loading.
# It depends on the kernel setting the IP address.
# You need ip=dhcp OR ip=fixed.ip.address.here in the command line
sshd.cpio.lzma:  Makefile
	GO111MODULE=off GOARCH=arm64 go run github.com/u-root/u-root -o sshd.cpio -build=bb \
		-uinitcmd=/bbin/sshd \
		-files class_key.pub:authorized_keys \
		-files classhostkey:id_rsa \
		-files /usr/bin/vi \
		-files /usr/share/vim \
		-files /usr/share/terminfo \
		-files /bin/bash \
		-files /usr/sbin/flashrom \
		-files /usr/bin/xz \
		all
	lzma -f -k sshd.cpio
	ls -l sshd.cpio.*

flashkernel: flash.config flashinitramfs.cpio.lzma Makefile
	cp $< linux/.config
	(cd linux && git apply ../0001-efistub-Workaround-Linuxboot-for-Ubuntu-20.04.patch)
	(cd linux && ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make olddefconfig && ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j32)
	cp linux/arch/arm64/boot/Image $@

testflashkernel: flashkernel
	qemu-system-aarch64 -machine virt -cpu cortex-a57 -kernel flashkernel -nographic -initrd flashinitramfs.cpio.lzma -append "console=ttyAMA0,115200 keep_bootcon"

# These stanzas fetch code.
fetch: getkernel geturoot 

getkernel:
	rm -rf linux
	git clone --depth=1 -b v5.15 --single-branch https://github.com/torvalds/linux
	(cd linux && make mrproper)
	(cd linux && ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make tinyconfig)

geturoot:
	GO111MODULE=off go get -u github.com/u-root/u-root
	GO111MODULE=off go get -u github.com/u-root/cpu/...

# Serve the combined sshd-kernel and sshd-initramfs image. This includes flashrom
sshd-pxeserver:
	echo Consider Disabling any system services ...
	echo e.g. sudo systemctl  stop tftpd-hpa.service
	echo e.g. sudo systemctl stop isc-dhcp-server
	sudo `go env GOPATH`/bin/pxeserver  -tftp-dir . -bootfilename sshdkernel  -http-dir . -interface $(NETWORK)

# Serve the combined cpu-kernel and cpu-initramfs image.
cpu-pxeserver:
	echo Consider Disabling any system services ...
	echo e.g. sudo systemctl  stop tftpd-hpa.service
	echo e.g. sudo systemctl stop isc-dhcp-server
	sudo `go env GOPATH`/bin/pxeserver  -tftp-dir . -bootfilename cpukernel  -http-dir . -interface $(NETWORK)

ssh:
	ssh -i class_key -p 2022 root@up
