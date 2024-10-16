**Alpine EFI Virtual Machine specific OS Image.**

*Build rootfs using docker:*

    docker build --force-rm -t alpine:vm .

*[Optionally] Debug rootfs using docker:*

	docker run --interactive --network host --rm --tty alpine:vm sh

*Dump rootfs from docker image:*

    docker save alpine:vm | tar --strip-components=1 --wildcards -xf - "*/layer.tar" && mv layer.tar rootfs.tar

*Prepare empty raw disk image file:*

	fallocate -l 10G rootfs.img

*Associate disk image with loop device:*

    sudo losetup loop1 rootfs.img

*Create partitions inside disk image:*

	sudo parted /dev/loop1 mklabel gpt
	sudo parted -a opt /dev/loop1 mkpart EFI fat32 2MiB 514MiB
	sudo parted -a opt /dev/loop1 mkpart root ext4 514MiB 100%

*Format partitions inside disk image:*

	sudo mkfs.vfat -F 32 -n EFI /dev/loop1p1
	sudo mkfs.ext4 -O encrypt -O ^FEATURE_C12 -m 0 -L root /dev/loop1p2

*Mount partitions from disk image to local system:*

	sudo mount /dev/loop1p2 /mnt
	sudo mkdir -p /mnt/efi && sudo mount /dev/loop1p1 /mnt/efi

*Apply roots to mounted disk image:*

	sudo tar -xvp -f rootfs.tar -C /mnt --numeric-owner

*Unmount disk image partitions from local system:*

	sudo umount -l /mnt/efi /mnt

*De-associate disk image from loop device:*

	sudo losetup -d /dev/loop1

*[Optionally] Test OS disk image using qemu:*

	qemu-system-x86_64 \
	  -enable-kvm \
	  -bios /usr/share/edk2-ovmf/x64/OVMF_CODE.fd \
	  -drive format=raw,file=rootfs.img \
	  -net none \
	  -serial stdio \
	  -m 4G \
	  -cpu host \
	  -smp 2
