# syntax = docker/dockerfile:1.4

FROM alpine:latest AS downloader

RUN apk add --no-cache \
        curl \
        libarchive-tools \
        tar \
        xz

RUN mkdir -p /rootfs && \
    curl -L http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz | bsdtar -xpf - -C /rootfs

FROM scratch AS base

COPY --from=downloader /rootfs/ /

RUN ldconfig

FROM base AS upgrade

RUN echo 'LANG="en_US.UTF-8"' > /etc/locale.conf && \
    echo 'en_US.UTF-8 UTF-8' > /etc/locale.gen && \
    locale-gen

RUN mkdir -p /etc/systemd/resolved.conf.d && \
    echo -e "[Resolve]\nDNS=1.1.1.1 8.8.8.8" > /etc/systemd/resolved.conf.d/dns_servers.conf

# https://github.com/moby/buildkit/issues/1267#issuecomment-768903038
RUN sed -i -e '/^HoldPkg/c\HoldPkg     = pacman glibc filesystem' /etc/pacman.conf && \
    sed -i -e 's~#IgnorePkg.*~IgnorePkg = filesystem~g' /etc/pacman.conf

RUN echo "Server = http://mirror.archlinuxarm.org/\$arch/\$repo" > /etc/pacman.d/mirrorlist && \
    pacman-key --init && \
    pacman-key --populate archlinuxarm

RUN pacman -Syy --noconfirm && \
    pacman -Rns --noconfirm \
        dhcpcd \
        netctl \
        openresolv \
        vi \
    && \
    pacman -Rndd --noconfirm \
        iptables \
    && \
    pacman -S --noconfirm \
        iptables-nft \
    && \
    pacman -Syu --noconfirm \
        iptables-nft \
    && \
    pacman -Scc --noconfirm

RUN userdel -r alarm

RUN userdel -r dhcpcd || true

RUN echo "vm" > /etc/hostname

RUN ln -sf /dev/null /etc/systemd/system/systemd-homed.service && \
    ln -sf /dev/null /etc/systemd/system/systemd-userdbd.service && \
    ln -sf /dev/null /etc/systemd/system/systemd-userdbd.socket && \
    ln -sf /dev/null /etc/systemd/system/display-manager.service && \
    ln -sf /dev/null /etc/systemd/system/plymouth-quit-wait.service && \
    ln -sf /dev/null /etc/systemd/system/plymouth-start.service && \
    ln -sf /dev/null /etc/systemd/system/syslog.service && \
    ln -sf /dev/null /etc/systemd/system/syslog.socket

RUN rm -vf /etc/locale.gen.pacnew

FROM upgrade AS yay_builder

RUN pacman -Syu --noconfirm \
        base-devel \
        git \
    && \
    useradd -m makepkg_user

USER makepkg_user

RUN cd /home/makepkg_user && \
    git clone https://aur.archlinux.org/yay-bin.git && \
    cd yay-bin && \
    makepkg -s --noconfirm

FROM upgrade AS rootfs

COPY --from=yay_builder /home/makepkg_user/yay-bin/yay-bin-*.pkg.* /tmp/

RUN pacman -U --noconfirm /tmp/yay-bin-*.pkg.* && \
    rm /tmp/yay-bin-*.pkg.*

RUN echo "root=LABEL=root rw quiet bgrt_disable nowatchdog" > /etc/kernel/cmdline

COPY <<EOF /etc/mkinitcpio.d/linux-aarch64.preset
ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/Image"
PRESETS=("default")
default_config="/etc/mkinitcpio.conf"
default_uki="/efi/EFI/Linux/archlinux-linux.efi"
default_options="-S autodetect"
EOF

COPY <<EOF /etc/systemd/system/chattr-mkinitcpio.service
[Unit]
Description=Set immutable flag on mkinitcpio preset file
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/bin/chattr +i /etc/mkinitcpio.d/linux-aarch64.preset
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

RUN mkdir -p /etc/systemd/system/multi-user.target.wants && \
    ln -s /etc/systemd/system/chattr-mkinitcpio.service /etc/systemd/system/multi-user.target.wants/chattr-mkinitcpio.service

COPY <<EOF /etc/fstab
LABEL=ESP  /efi           vfat     defaults,noatime 0 2
LABEL=root /              ext4     defaults,noatime 0 1
rosetta    /media/rosetta virtiofs ro,nofail        0 0
EOF

RUN mkdir -p /media/rosetta && \
    echo ":rosetta:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00:\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/media/rosetta/rosetta:CF" > /etc/binfmt.d/rosetta.conf

RUN rm -vf /boot/Image.gz /boot/initramfs-linux-fallback.img && \
    mkdir -p /efi/EFI/Linux/ && \
    mkinitcpio -p linux-aarch64

# RUN env SYSTEMD_RELAX_ESP_CHECKS=1 SYSTEMD_RELAX_XBOOTLDR_CHECKS=1 bootctl install --no-variables --esp-path=/efi/ --boot-path=/boot/ --root=/
RUN echo "fs0:\EFI\Linux\archlinux-linux.efi" > /efi/startup.nsh

# https://github.com/moby/buildkit/issues/1267#issuecomment-768903038
RUN pacman -Scc --noconfirm && \
    sed -i -e '/^HoldPkg/c\HoldPkg     = pacman glibc' /etc/pacman.conf && \
    sed -i -e 's~^IgnorePkg~#IgnorePkg~g' /etc/pacman.conf

RUN test -f /etc/pacman.conf.pacnew && \
    mv -f /etc/pacman.conf.pacnew /etc/pacman.conf

FROM scratch
COPY --from=rootfs / /
