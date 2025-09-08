
# Alpine Linux (Headless) for **Orange Pi Zero 3 (H618)** — Built from **macOS** using **QEMU** (HVF)

This guide gets you a **bootable microSD image** with Alpine Linux (wired LAN, headless, SSH) for the **Orange Pi Zero 3** — *using only your Mac* by running a tiny **ARM Linux VM** in QEMU with the **Hypervisor Framework (HVF)** accelerator. You’ll build **TF‑A (BL31)** + **U‑Boot**, assemble an Alpine rootfs, and write one finished image to your microSD.

Why a VM? macOS has **no reliable ext4 write support**; doing everything in a Linux VM avoids filesystem and toolchain pain. You only use macOS to launch QEMU and `dd` the final image to your card.

---

## 0) Prereqs on macOS (Apple Silicon)

1) Install **Homebrew** if needed: <https://brew.sh>

2) Install QEMU (brings aarch64 system emulator and UEFI firmware):
```bash
brew install qemu coreutils
```
> `coreutils` gives you `gdd` with `status=progress` on macOS.

3) Locate the **UEFI firmware** that QEMU ships (the file name may vary slightly):
```bash
# Find the AArch64 UEFI code image installed by Homebrew
find /opt/homebrew -name 'edk2-aarch64-code.fd'
# Copy it next to where you'll run QEMU from:
cp /opt/homebrew/Cellar/qemu/*/share/qemu/edk2-aarch64-code.fd ./
# Create a writable NVRAM vars file:
cp /opt/homebrew/Cellar/qemu/*/share/qemu/edk2-arm-vars.fd ./ovmf_vars.fd
```
(If `edk2-arm-vars.fd` isn’t present, just `cp edk2-aarch64-code.fd ovmf_vars.fd` — QEMU will create vars at first boot.)

4) Create a work folder and fetch an **ARM64** Linux installer ISO (Ubuntu is simple; Alpine works too). Place the ISO alongside the firmware files, e.g.:
```
ubuntu-24.04.1-live-server-arm64.iso
```

---

## 1) Create an ARM64 Linux VM with QEMU (HVF)

Create a 24 GB disk image:
```bash
qemu-img create -f qcow2 ubuntu-arm64.qcow2 24G
```

Launch the installer with HVF acceleration and UEFI:
```bash
qemu-system-aarch64 \
  -machine virt,highmem=on \
  -accel hvf -cpu host \
  -smp 4 -m 4096 \
  -drive if=pflash,format=raw,readonly=on,file=edk2-aarch64-code.fd \
  -drive if=pflash,format=raw,file=ovmf_vars.fd \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-gpu-pci -display default \
  -drive if=virtio,file=ubuntu-arm64.qcow2,format=qcow2 \
  -drive if=virtio,media=cdrom,file=ubuntu-24.04.1-live-server-arm64.iso \
  -boot menu=on
```

Install Ubuntu normally (minimal server is fine). Ensure **OpenSSH server** is installed. After install, shut down the VM, **remove the CDROM drive**, and reboot the VM with the same command but **without** the `-drive ...cdrom...` line.

You can now SSH into the VM from macOS:
```bash
ssh -p 2222 <user>@127.0.0.1
```

> If you prefer Alpine instead of Ubuntu inside the VM, the steps below are the same—just use your distro’s package names.

---

## 2) Inside the VM — Install build and image tools

```bash
sudo apt update
sudo apt install -y git build-essential device-tree-compiler flex bison \
    libssl-dev pkg-config make gcc wget curl rsync \
    parted e2fsprogs dosfstools mtools \
    u-boot-tools ca-certificates
```

> We’re building **natively on ARM64** inside the VM; no cross toolchain needed.

---

## 3) Build **TF‑A (BL31)** and **U‑Boot** for **H616/H618**

**3.1) Trusted Firmware‑A (H616 platform, works for H618 family):**
```bash
cd ~
git clone https://git.trustedfirmware.org/TF-A/trusted-firmware-a.git
cd trusted-firmware-a
# Release build (smaller/faster); use DEBUG=1 if you want verbose
make -j$(nproc) PLAT=sun50i_h616 bl31
export BL31=$PWD/build/sun50i_h616/release/bl31.bin
```

**3.2) U‑Boot (use a recent stable tag):**
```bash
cd ~
git clone https://source.denx.de/u-boot/u-boot.git
cd u-boot
# Use a known-good stable. Adjust if newer is available.
git checkout v2024.10
make orangepi_zero3_defconfig
make -j$(nproc) BL31=$BL31
# Result:
ls -lh u-boot-sunxi-with-spl.bin
```

> The single file `u-boot-sunxi-with-spl.bin` is what the Allwinner ROM expects on SD/eMMC. We’ll place it at the standard **8 KiB (sector 16)** offset in the final image.

---

## 4) Create a **raw SD image** with partitions

We’ll assemble everything inside a single file `opi-zero3-alpine.img` and later write that to your real microSD from macOS.

```bash
cd ~
# 4 GiB image (expand if you like)
truncate -s 4G opi-zero3-alpine.img

# Partition: p1 = 256MB (ext4, /boot), p2 = rest (ext4, /)
# sfdisk expects sectors; we start at 1MiB (2048 sectors)
cat <<'EOF' | sudo sfdisk opi-zero3-alpine.img
label: gpt
unit: sectors

# start, size, type
2048, 524288, L   # ~256MB
*,     ,     L
EOF
```

Create loop devices and filesystems:
```bash
# Map partitions from the image
LOOP=$(sudo losetup --show -f -P opi-zero3-alpine.img)
echo "Loop is $LOOP (e.g. /dev/loop0)"

sudo mkfs.ext4 -L boot   ${LOOP}p1
sudo mkfs.ext4 -L rootfs ${LOOP}p2

sudo mkdir -p /mnt/opi/boot /mnt/opi/root
sudo mount ${LOOP}p1 /mnt/opi/boot
sudo mount ${LOOP}p2 /mnt/opi/root
```

---

## 5) Populate **Alpine** on the rootfs

Download **Alpine aarch64 Mini RootFS** (latest-stable) inside the VM:
```bash
cd /tmp
wget -O alpine-minirootfs-aarch64.tar.gz \
  https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/aarch64/alpine-minirootfs-3.22.1-aarch64.tar.gz
sudo tar -xpf alpine-minirootfs-aarch64.tar.gz -C /mnt/opi/root
```

Chroot to finish setup (bind mounts first):
```bash
sudo mount --bind /dev  /mnt/opi/root/dev
sudo mount --bind /sys  /mnt/opi/root/sys
sudo mount -t proc proc /mnt/opi/root/proc
sudo chroot /mnt/opi/root /bin/sh
```

Inside the **chroot**:
```sh
# Repos
echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/main"   >  /etc/apk/repositories
echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/community" >> /etc/apk/repositories

apk update
# Use linux-edge to get the H618 DTB and newer kernel/drivers
apk add alpine-base linux-edge openssh openrc ca-certificates

# Networking: DHCP on eth0
cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF
rc-update add networking default

# SSH server
rc-update add sshd default
mkdir -p /root/.ssh && chmod 700 /root/.ssh
# (Optional) Paste your SSH public key now, then Ctrl-D to finish:
# cat >> /root/.ssh/authorized_keys
# chmod 600 /root/.ssh/authorized_keys

# Hostname & timezone (adjust to taste)
echo opi-zero3 > /etc/hostname
setup-timezone -z Europe/Zurich || true

# Serial console (UART header), useful for debugging
echo 'ttyS0::respawn:/sbin/agetty -L 115200 ttyS0 vt100' >> /etc/inittab

# Set a root password (if you didn’t add a key)
passwd
exit
```

Unmount the chroot bind mounts:
```bash
sudo umount /mnt/opi/root/{dev,sys,proc}
```

Copy **kernel/initramfs/DTBs** from rootfs to the boot partition (for extlinux):
```bash
sudo rsync -a /mnt/opi/root/boot/ /mnt/opi/boot/
```

Create **extlinux** config (U‑Boot reads this):
```bash
sudo mkdir -p /mnt/opi/boot/extlinux
sudo tee /mnt/opi/boot/extlinux/extlinux.conf >/dev/null <<'EOF'
TIMEOUT 3
DEFAULT alpine

LABEL alpine
  MENU LABEL Alpine (edge)
  LINUX /boot/vmlinuz-edge
  INITRD /boot/initramfs-edge
  FDT /boot/dtbs-edge/allwinner/sun50i-h618-orangepi-zero3.dtb
  APPEND root=/dev/mmcblk0p2 rootfstype=ext4 rw console=ttyS0,115200 console=tty1
EOF
```

Unmount partitions:
```bash
sudo umount /mnt/opi/boot /mnt/opi/root
sudo losetup -d "$LOOP"
```

---

## 6) **Embed U‑Boot** into the image

Write the integrated **SPL+U‑Boot** at the Allwinner offset (**8 KiB**):
```bash
cd ~/u-boot
sudo dd if=u-boot-sunxi-with-spl.bin of=~/opi-zero3-alpine.img bs=1024 seek=8 conv=notrunc
sync
```

> The H616/H618 BootROM also checks a secondary higher offset on GPT media; the standard **8 KiB** placement is sufficient for SD boots in practice.

---

## 7) Copy the finished image to macOS and flash your microSD

From macOS, pull the image through the SSH port forward:
```bash
# On macOS host
scp -P 2222 <user>@127.0.0.1:~/opi-zero3-alpine.img .
```

Find your microSD (replace N with the right number):
```bash
diskutil list
diskutil unmountDisk /dev/diskN
```

Flash (fast path using GNU dd from coreutils):
```bash
sudo gdd if=opi-zero3-alpine.img of=/dev/rdiskN bs=4M status=progress oflag=direct conv=fsync
```
or with BSD `dd` (slower, no progress bar):
```bash
sudo dd if=opi-zero3-alpine.img of=/dev/rdiskN bs=4m conv=sync
sync
```

Eject:
```bash
diskutil eject /dev/diskN
```

---

## 8) First boot on the Orange Pi Zero 3

1. Insert the microSD, plug **Ethernet**, power via USB‑C.
2. If you have a USB‑TTL dongle, watch serial @ **115200 N‑8‑1**.
3. Otherwise, find the IP from your router’s DHCP leases or scan:
   ```bash
   nmap -sn 192.168.1.0/24
   ```
4. SSH in:
   ```bash
   ssh root@<opi-ip>
   ```

Verify:
```bash
uname -a
ls /boot/dtbs-edge/allwinner | grep h618
ip a show eth0
rc-status
```

---

## 9) Troubleshooting

- **Black screen / drops to U‑Boot prompt**  
  Check that `extlinux.conf` is at `/boot/extlinux/extlinux.conf` on **p1**, and that the paths to `vmlinuz-edge`, `initramfs-edge`, and the **H618 DTB** are correct.

- **“Bad Linux ARM64 Image magic!”** at boot  
  You pointed U‑Boot at the wrong kernel image or a format it can’t parse. Use `vmlinuz-edge` (as installed) and keep the extlinux stanza above.

- **No Ethernet**  
  Make sure you installed **`linux-edge`** (newer kernels include Zero 3 support) and you’re using the **H618** DTB. DHCP is enabled in `/etc/network/interfaces`.

- **1.5 GB RAM variants**  
  Very old U‑Boots misreport 1.5 GB units as 2 GB; use an up‑to‑date U‑Boot as built above.

---

## 10) Notes & rationale

- **Why a VM?** macOS can *read* ext4 with extra drivers, but reliable **write** support is still problematic; assembling an image inside Linux avoids all of that.
- **Why `extlinux`?** Modern U‑Boot implements the **Generic Distro** boot flow and reads `extlinux.conf` out of the box.
- **Why `linux-edge`?** It carries the **`sun50i-h618-orangepi-zero3.dtb`** and up‑to‑date drivers for H618 boards.

---

## Appendix A — Re-run the VM (without ISO)

```bash
qemu-system-aarch64 \
  -machine virt,highmem=on \
  -accel hvf -cpu host \
  -smp 4 -m 4096 \
  -drive if=pflash,format=raw,readonly=on,file=edk2-aarch64-code.fd \
  -drive if=pflash,format=raw,file=ovmf_vars.fd \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-gpu-pci -display default \
  -drive if=virtio,file=ubuntu-arm64.qcow2,format=qcow2 \
  -boot menu=on
```

## Appendix B — Static IP (optional)

Replace the `eth0` stanza with e.g.:
```sh
auto eth0
iface eth0 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1
```
Then:
```sh
rc-service networking restart
```

---

### Done

You now have a **repeatable macOS → QEMU → microSD** path that produces a clean Alpine Linux headless system for the Orange Pi Zero 3.
