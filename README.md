__0. Prepare Installation Media__

__0.1 Download Gentoo ISO__

Obtain the download link from the downloads page

Note

The date in the links below (e.g. 20251123T...) is for reference only. Always select the latest dated file from the mirror.

Download the Minimal ISO (using official Gentoo mirrors):

```
# Use mirrorselect to find the closest mirror
emerge --ask app-portage/mirrorselect
mirrorselect -i -r -D

# Or download directly from the official mirror list:
# https://www.gentoo.org/downloads/mirrors/
wget https://distfiles.gentoo.org/releases/amd64/autobuilds/<latest-version>/install-amd64-minimal-latest-version.iso
wget https://distfiles.gentoo.org/releases/amd64/autobuilds/<latest-version>/install-amd64-minimal-latest-version.iso.asc

```

GRAPHICAL INSTALLATION: Use the LiveGUI USB Image

If you want to use a browser during installation or connect to Wi-Fi more easily, choose the LiveGUI USB Image from the official downloads page OR use ENDEAVOUROS liveGUI which is arch linux base distro


Verify signature (recommended):

```
# Import the Gentoo release signing key.
# Preferred: use the local copy that ships with Live media or sys-apps/openpgp-keys-gentoo-release
gpg --import /usr/share/openpgp-keys/gentoo-release.asc
# Fallback (no local key — needs network access to keys.gentoo.org):
#   gpg --keyserver hkps://keys.gentoo.org --recv-keys D99EAC7379A850BCE47DA5F29E6438C817072058

# Verify the ISO signature
gpg --verify install-amd64-minimal-<latest-version>.iso.asc install-amd64-minimal-latest-version.iso

```


__0.2 Create Bootable USB__

Linux:

```
sudo dd if=install-amd64-minimal-<latest-version>.iso of=/dev/sdX bs=4M status=progress oflag=sync
# Replace sdX with your USB device name (e.g., /dev/sdb)

```

Windows: Use Rufus → Select ISO → Choose DD mode when writing.


__1. Enter Live Environment and Connect to Network__

```
nmtui   # interactive: pick "Activate a connection" or "Edit a connection"

```

Confirm with:

```
ping -c3 gentoo.org

```

If you prefer manual configuration use the sections below.

__1.1 Wired Network (manual)__


```
ip link              # View network interface names (e.g. eno1, eth0)
dhcpcd eno1          # Enable DHCP on the wired interface
ping -c3 gentoo.org  # Test network connectivity

```


__1.2 Wireless Network (manual)__

Interactive helper:


```
net-setup

```


wpa_supplicant (fill in your interface, SSID, password):

```
wpa_passphrase "SSID" "PASSWORD" | tee /etc/wpa_supplicant/wpa_supplicant.conf
wpa_supplicant -B -i wlp0s20f3 -c /etc/wpa_supplicant/wpa_supplicant.conf
dhcpcd wlp0s20f3

```

If WPA3 is unstable, try falling back to WPA2.


TODO:Advanced Settings: Enable SSH for Remote Access (Click to Expand)


__2. Plan Disk Partitioning__


Recommended Partition Scheme (UEFI)

The table below provides a recommended default partition layout for a Gentoo installation.
Device Path	Mount Point	Filesystem	Description

```
/dev/sdX	/efi	vfat	EFI System Partition (ESP)

/dev/sdX	swap	swap	Swap partition

/dev/sdX	/	xfs	Root partition

```

cfdisk Practical Example (Recommended)

cfdisk is a graphical partitioning tool with a simple, intuitive interface.

```
cfdisk /dev/sdX

```

Operation tips:

    Select `GPT` label type.
    Create `ESP`: New partition → size 1G → type EFI System.
    Create `Swap`: New partition → size 4G → type Linux swap.
    Create `Root`: New partition → remaining space → type Linux filesystem (default).
    Select Write to write changes, type yes to confirm.
    Select Quit to exit.

Advanced Settings: `fdisk` Command-Line Partitioning (Click to Expand)


__3. Create Filesystems and Mount__

__3.1 Format__

```
mkfs.fat -F 32 -n "EFI" /dev/sdX  # Format ESP partition as FAT32
mkswap /dev/sdX          # Format Swap partition
mkfs.xfs /dev/sdX        # Format Root partition as XFS

```


For `Btrfs`:

```
mkfs.btrfs -L gentoo /dev/sdX

```

For ext4:

```
mkfs.ext4 /dev/sdX

```

__3.2 Mount (XFS example)__

```
mount /dev/sdX /mnt/gentoo        # Mount root partition
mkdir -p /mnt/gentoo/efi                # Create ESP mount point
mount /dev/sdX /mnt/gentoo/efi    # Mount ESP partition
swapon /dev/sdX                   # Enable Swap partition

```


__Advanced Settings: Btrfs Subvolume Example (Click to Expand)__

1. Format

```
mkfs.fat -F 32 /dev/sdX  # Format ESP
mkswap /dev/sdX          # Format Swap
mkfs.btrfs -L gentoo /dev/sdX # Format Root (Btrfs)

```

2. Create subvolumes

```
mount /dev/sdX /mnt/gentoo
btrfs subvolume create /mnt/gentoo/@
btrfs subvolume create /mnt/gentoo/@home
umount /mnt/gentoo

```

3. Mount subvolumes

```
mount -o compress=zstd,subvol=@ /dev/sdX /mnt/gentoo
mkdir -p /mnt/gentoo/{efi,home}
mount -o subvol=@home /dev/sdX /mnt/gentoo/home
mount /dev/sdX /mnt/gentoo/efi    # Note: ESP must be FAT32
swapon /dev/sdX

```

4. Verify mounts

```
lsblk

```


__Btrfs Snapshot Recommendation__

It is recommended to use Snapper to manage snapshots. A proper subvolume layout (e.g., separating @ and @home) makes system rollback much easier.


Recommendation

After mounting, use lsblk to verify mount points are correct.


```
lsblk

```


```
NAME             MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
 nvme0n1          259:1    0 931.5G  0 disk
├─/dev/sdX      259:7    0     1G  0 part  /efi
├─/dev/sdX      259:8    0     4G  0 part  [SWAP]
└─/dev/sdX      259:9    0 926.5G  0 part  /

```


__4. Download Stage3 and Enter chroot__

__4.1 Choose Stage3__

    OpenRC: stage3-amd64-openrc-*.tar.xz
    systemd: stage3-amd64-systemd-*.tar.xz
    Desktop variants just have some USE flags pre-enabled; the standard version is more flexible.


__4.2 Download and Extract__


```
cd /mnt/gentoo

# Gentoo provides a txt file with the latest Stage3 path — use it to auto-detect:
# For OpenRC:
STAGE3=$(wget -qO- https://distfiles.gentoo.org/releases/amd64/autobuilds/latest-stage3-amd64-openrc.txt | grep -v '^#' | cut -d' ' -f1)
# For systemd:
# STAGE3=$(wget -qO- https://distfiles.gentoo.org/releases/amd64/autobuilds/latest-stage3-amd64-systemd.txt | grep -v '^#' | cut -d' ' -f1)

wget "https://distfiles.gentoo.org/releases/amd64/autobuilds/${STAGE3}"
wget "https://distfiles.gentoo.org/releases/amd64/autobuilds/${STAGE3}.asc"

```


Alternative: Visit https://www.gentoo.org/downloads/, right-click your chosen Stage3 variant → "Copy link address", then paste after wget.

```
# Verify signature (recommended)
gpg --verify stage3-*.tar.xz.asc stage3-*.tar.xz

# Extract Stage3
# x:extract p:preserve permissions v:verbose f:specify file --numeric-owner:use numeric IDs
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner

```

__4.3 Copy DNS and Mount Pseudo-Filesystems__

```
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/ # Copy DNS configuration
mount --types proc /proc /mnt/gentoo/proc          # Mount process information
mount --rbind /sys /mnt/gentoo/sys                 # Bind mount system information
mount --rbind /dev /mnt/gentoo/dev                 # Bind mount device nodes
mount --rbind /run /mnt/gentoo/run                 # Bind mount runtime information
mount --make-rslave /mnt/gentoo/sys                # Set as slave mount (prevents affecting host on unmount)
mount --make-rslave /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/run

```


__4.4 Enter chroot__

```
chroot /mnt/gentoo /bin/bash    # Switch root directory to new system
source /etc/profile             # Load environment variables
export PS1="(chroot) ${PS1}"    # Modify prompt to distinguish environment

```


__5. Initialize Portage and make.conf__

__5.1 Sync Tree__

```
emerge-webrsync   # Get the latest Portage snapshot (faster than rsync)
emerge --sync     # Sync Portage tree (get latest ebuilds)
emerge --ask app-editors/neovim # Install Vim editor (recommended)
eselect editor list          # List available editors
eselect editor set nvim        # Set Vim as default editor

```

Configure mirror (choose one):

```
emerge -av -1 app-portage/mirrorselect
mirrorselect -i -o >> /etc/portage/make.conf
# Or manually set a mirror from the official list:
# https://www.gentoo.org/downloads/mirrors/
# set necessary keyrings
getuto

```

__5.2 make.conf Example__

Edit /etc/portage/make.conf:

```
vim /etc/portage/make.conf

```

Adjust the -j value in MAKEOPTS to match your CPU core count (e.g., use -j4 for a 4-core CPU).


```
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

# limiter j4 -l4
MAKEOPTS="-j4"

ACCEPT_LICENSE="*"

VIDEO_CARDS="intel i965 iris"

INPUT_DEVICES="libinput"
GRUB_PLATFORMS="efi-64"

CPU_FLAGS_X86="aes avx avx2 bmi1 bmi2 f16c fma3 mmx mmxext pclmul popcnt rdrand sse sse2 sse3 sse4_1 sse4_2 ssse3"

USE="pipewire pipewire-pulse ffmpeg extra appindicator"

# NOTE: This stage was built with the bindist USE flag enabled

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C.UTF-8

GENTOO_MIRRORS="https://za.mirrors.cicku.me/gentoo/ \
    https://gentoo.dimensiondata.com/ \
    https://setebos.za.ext.planetunix.net/pub/gentoo/ \
    https://gentoo.uls.co.za/ \
    https://mirror.ufs.ac.za/gentoo/"

# Always use binary packages by default and verify signatures
FEATURES="${FEATURES} getbinpkg binpkg-request-signature"

```

CPU Instruction Set Optimization (CPU_FLAGS_X86) (Click to Expand)

o let Portage know which CPU instruction sets your processor supports (e.g., AES, AVX, SSE4.2), configure CPU_FLAGS_X86.

Install detection tool:

```emerge --ask app-portage/cpuid2cpuflags```

Run detection and write to config:

```
cpuid2cpuflags >> /etc/portage/make.conf

```

__6. Profile, System Settings & Localization__

__6.1 Choose Profile__

```
eselect profile list          # List all available profiles
eselect profile set <number>  # Set the selected profile
emerge -avuDNg @world          # Update system to match new profile

```

Common options:

```
  [1]   default/linux/amd64/23.0 (stable) x
  [2]   default/linux/amd64/23.0/systemd (stable) *
  [3]   default/linux/amd64/23.0/desktop (stablei) x

```

__6.2 Timezone and Locale__

```
# Set timezone (use your actual timezone)
# List available timezones:
ls /usr/share/zoneinfo/
ls -l /usr/share/zoneinfo/Africa/
# Examples: UTC, America/New_York, Europe/London, Asia/Tokyo, Australia/Sydney
ln -sf ../usr/share/zoneinfo/Africa/Lagos /etc/localtime
emerge --config sys-libs/timezone-data
2026-08-23                      # Verify
nvim /etc/locale.gen            # set locale
en_US.UTF-8 UTF-8               # For a normal English setup, uncomment
locale-gen                      # Generate selected locales
eselect locale set en_US.utf8   # Set system default locale
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"

```

__6.3 Hostname and Network Configuration__

Set hostname:

```
echo "gentoo" > /etc/hostname

```

Network manager options:

```
emerge --ask net-misc/networkmanager
# OpenRC:
rc-update add NetworkManager default
# systemd:
systemctl enable NetworkManager

```


__6.4 Configure fstab__

genfstab ships in the sys-fs/genfstab package (originally from Arch)


```
emerge --ask sys-fs/genfstab

```

Standard usage (run outside `chroot`):

```
# 1. Confirm all partitions are correctly mounted
lsblk
mount | grep /mnt/gentoo

# 2. Generate fstab (using UUID)
genfstab -U /mnt/gentoo >> /mnt/gentoo/etc/fstab

# 3. Check the generated file
cat /mnt/gentoo/etc/fstab

```

If you've already `chrooted` into the new system, you can:

Method 1: Run inside `chroot` (simplest)

```
emerge --ask sys-fs/genfstab
genfstab -U / >> /etc/fstab
nvim /etc/fstab  # Check and clean up extra entries (e.g., /proc, /sys, /dev)

```

Method 1: Run inside `chroot` (simplest)

```
emerge --ask sys-fs/genfstab
genfstab -U / >> /etc/fstab
vim /etc/fstab  # Check and clean up extra entries (e.g., /proc, /sys, /dev)

```

Method 2: Open a new terminal window (LiveGUI)

If using a Live environment with a GUI (like the official Gentoo LiveGUI), open a new terminal:

```
genfstab -U /mnt/gentoo >> /mnt/gentoo/etc/fstab

```

Method 3: TTY switch (Minimal ISO)

1. Press Ctrl+Alt+F2 to switch to a new TTY (Live environment)
2. Install and run:

```
emerge --ask sys-fs/genfstab
genfstab -U /mnt/gentoo >> /mnt/gentoo/etc/fstab

```

3. Press Ctrl+Alt+F1 to return to `chroot`

Prerequisite: every partition must already be mounted correctly (including Btrfs subvolumes and unlocked LUKS partitions) before you run `genfstab`


Method B: Manual Edit

1. Get partition UUIDs

```
blkid

```
Output example:

```
/dev/nvme0n1p1: UUID="7E91-5869" TYPE="vfat" PARTLABEL="EFI"
/dev/nvme0n1p2: UUID="7fb33b5d-..." TYPE="swap" PARTLABEL="swap"
/dev/nvme0n1p3: UUID="8c08f447-..." TYPE="xfs" PARTLABEL="root"

```

2. Edit `fstab`

```
nvim /etc/fstab

```

Basic configuration example (ext4/xfs):

```
# <UUID>                                   <Mount>      <Type> <Options>         <dump> <fsck>
UUID=7E91-5869                             /efi         vfat   defaults,noatime  0      2
UUID=7fb33b5d-4cff-47ff-ab12-7b461b5d6e13  none         swap   sw                0      0
UUID=8c08f447-c79c-4fda-8c08-f447c79ce690  /            xfs    defaults,noatime  0      1

```


fstab fields
Field	Meaning
UUID	Unique filesystem identifier (get it from blkid)
Mount	Mount point (none for swap)
Type	`vfat`, `ext4`, `xfs`, `btrfs`, `swap`, etc.
Options	Comma-separated mount options
dump	Backup flag — usually `0`
fsck	Boot-time check order: `1` = root, `2` = others, `0` = skip



Btrfs subvolume configuration

With `genfstab`:

If the `Btrfs` subvolumes are already mounted correctly, `genfstab -U` picks up `subvol=` automatically.

```
# Confirm subvolume mounts
mount | grep btrfs
# Example output:
#   /dev/nvme0n1p3 on /mnt/gentoo type btrfs (rw,noatime,compress=zstd:3,subvol=/@)

# Generate
genfstab -U /mnt/gentoo >> /mnt/gentoo/etc/fstab

```

Manual example:

```
# Root subvolume
UUID=7b44c5eb-caa0-413b-9b7e-a991e1697465  /       btrfs  defaults,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=@       0 0

# Home subvolume (same UUID, different subvolume)
UUID=7b44c5eb-caa0-413b-9b7e-a991e1697465  /home   btrfs  defaults,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=@home   0 0

# Swap (separate partition)
UUID=7fb33b5d-4cff-47ff-ab12-7b461b5d6e13  none    swap   sw                                                                            0 0

# EFI partition
UUID=7E91-5869                             /efi    vfat   defaults,noatime,fmask=0022,dmask=0022                                        0 2

```


Common `Btrfs` mount options

Option	             Meaning
`compress=zstd:3`	     `zstd` compression at level 3 (good performance/ratio balance)
`discard=async`	          Async TRIM (recommended for SSDs)
`space_cache=v2`	      v2 space cache (default; better performance)
`subvol=@`	              The subvolume to mount
`noatime`	              Skip access-time updates (slight performance win)



Notes

   - All subvolumes of the same Btrfs partition share the same UUID.
   - Always use blkid to read your real UUIDs.



__7. Kernel and Firmware__

__7.1 Quick Option: Pre-compiled Kernel__

```
emerge --ask sys-kernel/gentoo-kernel-bin

```

Remember to regenerate the bootloader configuration after kernel upgrades.

```
grub-mkconfig -o /boot/grub/grub.cfg

```

__7.2 Install Firmware and Microcode__

```
mkdir -p /etc/portage/package.license
# Accept the Linux firmware license terms
echo 'sys-kernel/linux-firmware linux-fw-redistributable no-source-code' > /etc/portage/package.license/linux-firmware

# Pick the installkernel USE flags that match your bootloader, so dist-kernel
# upgrades automatically regenerate the bootloader config. Pick ONE row:
#   GRUB users          → 'sys-kernel/installkernel dracut grub'
#   systemd-boot users  → 'sys-kernel/installkernel dracut systemd systemd-boot'
#   Limine / other      → 'sys-kernel/installkernel dracut' (bootloader handled manually)
echo 'sys-kernel/installkernel dracut grub' > /etc/portage/package.use/installkernel

emerge --ask sys-kernel/linux-firmware
emerge --ask sys-firmware/intel-microcode  # Intel CPU users

```


__8. Base Tools__

__8.1 System Service Tools__

systemd includes built-in logging and scheduled task services, no additional installation needed.

Time synchronization

```
systemctl enable --now systemd-timesyncd

```

__8.2 Filesystem Tools__

Install tools for your chosen filesystem (required):

```
emerge --ask sys-fs/e2fsprogs  # ext4
emerge --ask sys-fs/xfsprogs   # XFS
emerge --ask sys-fs/dosfstools # FAT/vfat (required for EFI partition)
emerge --ask sys-fs/btrfs-progs # Btrfs

```

__9. Create Users and Permissions__

```
passwd root # Set root password
useradd -m -G wheel,video,audio,plugdev,network username # Create user and add to common groups
passwd username # Set user password
emerge --ask app-admin/sudo

```

Allow users in the wheel group to execute commands as root by editing the sudoers file:

```
visudo

```

Uncomment the following line (remove the # at the beginning):

```
%wheel ALL=(ALL:ALL) ALL

```

__10. Install Bootloader__

__10.1 Option A: GRUB (Recommended/Standard)__

GRUB is the most feature-complete bootloader with the best compatibility, and supports automatic Windows detection.

1. Install and configure

```
emerge --ask sys-boot/grub:2
# Install to ESP (--bootloader-id=Gentoo automatically creates a separate directory, avoiding conflicts)
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=Gentoo

```

2. Multi-OS configuration (Windows/Linux/other)

If you have Windows or other Linux distributions installed, enable os-prober to automatically detect them:

```
emerge --ask sys-boot/os-prober
# Enable os-prober (disabled by default for security)
echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub

```

3. Generate configuration file

```
grub-mkconfig -o /boot/grub/grub.cfg

```

__11. Final Steps__

__11.1 Final Checklist__

1. emerge --info runs without errors

2. UUIDs in /etc/fstab are correct (verify again with blkid)

3. Root and regular user passwords have been set

4. grub-mkconfig has been run or bootctl/Limine configuration is complete

5. If using LUKS, confirm initramfs includes cryptsetup

__11.2 Exit Chroot and Reboot__

After confirming everything is correct, exit the chroot environment and unmount:

```
exit
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
swapoff -a
reboot

```
