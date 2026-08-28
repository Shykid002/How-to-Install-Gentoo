One important warning first: the commands below destroy /dev/sda2 completely.
Make sure `/dev/sda1` is actually your EFI partition and `/dev/sda2` is the partition
you intend to encrypt before running the storage commands.

Gentoo Linux — `systemd` + `LUKS2` + `Btrfs`
MacBook Air 2017 — copy-paste-ready installation

The resulting layout will be:

    ```

    /dev/sda
    ├── /dev/sda1       EFI System Partition
    └── /dev/sda2       LUKS2
          │
          └── cryptroot
                │
                └── Btrfs
                     ├── @
                     ├── @home
                     ├── @snapshots
                     └── @swap
                          └── swapfile


    ``` 


_Step 1 — Boot the Live USB_

Boot a Gentoo Minimal Installation CD/USB or another Linux live environment.

-Check the disks first:

```
!#bash

lsblk -f

```

You should identify something resembling:

```
    sda
    ├─sda1   vfat
    └─sda2

```

Do not blindly assume `/dev/sda2` is correct. On some systems the disk may appear as a different device.

Check networking:

```
!#bash

    ping -c 3 gentoo.org

```

-Set the variables:

```
    export DEV_EFI="/dev/sda1"
    export DEV_LUKS="/dev/sda2"
    export MAPPER_NAME="cryptroot"
    export LUKS_PATH="/dev/mapper/${MAPPER_NAME}"

```

-Verify them:

```
!#bash

    echo "EFI:  $DEV_EFI"
    echo "LUKS: $DEV_LUKS"
    echo "Mapper: $LUKS_PATH"

```

_Step 2 — Create the LUKS2 + Btrfs storage layout_

**⚠️ DESTROYS the data on /dev/sda2**


-First, make absolutely sure:

```
lsblk -f "$DEV_LUKS"

```

-Then create the LUKS2 container:

```
cryptsetup luksFormat --type luks2 "$DEV_LUKS"

```

-Open it:

```
cryptsetup open "$DEV_LUKS" "$MAPPER_NAME"

```


-Verify:

```
lsblk -f

```

-You should now see:

```
cryptroot

```

-Create the Btrfs filesystem:

```
mkfs.btrfs -L gentoo_root "$LUKS_PATH"

```

-Mount the Btrfs filesystem temporarily:

```
mkdir -p /mnt/gentoo
mount "$LUKS_PATH" /mnt/gentoo

```


-Create the subvolumes:

```
btrfs subvolume create /mnt/gentoo/@
btrfs subvolume create /mnt/gentoo/@home
btrfs subvolume create /mnt/gentoo/@snapshots
btrfs subvolume create /mnt/gentoo/@swap

```

-Check them:

```
btrfs subvolume list /mnt/gentoo

```

-Unmount:

```
umount /mnt/gentoo

```

_Step 3 — Mount the final Btrfs layout_

-Mount root:

```
mount -o noatime,compress=zstd:3,subvol=@ "$LUKS_PATH" /mnt/gentoo

```


-Create directories:

```
mkdir -p /mnt/gentoo/{home,snapshots,swap,efi}

```

-Mount /home:

```
mount -o noatime,compress=zstd:3,subvol=@home \
    "$LUKS_PATH" /mnt/gentoo/home

```

-Mount snapshots:

```
mount -o noatime,compress=zstd:3,subvol=@snapshots \
    "$LUKS_PATH" /mnt/gentoo/snapshots

```


-Mount swap subvolume:

```
mount -o noatime,subvol=@swap \
    "$LUKS_PATH" /mnt/gentoo/swap

```


-Mount EFI:

```
mount "$DEV_EFI" /mnt/gentoo/efi

```


-Check everything:

```
findmnt /mnt/gentoo

```


-You should have something along the lines of:

```
/mnt/gentoo
/mnt/gentoo/home
/mnt/gentoo/snapshots
/mnt/gentoo/swap
/mnt/gentoo/efi

```


_Step 4 — Create the Btrfs swapfile_


For Btrfs, use the filesystem's swapfile support rather than simply creating a normal file with `dd`.

-First check your Btrfs tools:

```
btrfs --version

```

-Create a 4 GiB swapfile:

```
btrfs filesystem mkswapfile --size 4g /mnt/gentoo/swap/swapfile

```

-Activate it:

```
swapon /mnt/gentoo/swap/swapfile

```


-Verify:

```
swapon --show

```


-You should see:

```
/mnt/gentoo/swap/swapfile

```

Why 4 GiB? Your MacBook has 8 GiB RAM, so 4 GiB is a reasonable basic swap allocation.
If you intend to use hibernation, the swap strategy should be sized/configured differently.


_Step 5 — Download the Gentoo Stage 3_

-Enter the installation root:

```
cd /mnt/gentoo

```

Rather than hard-coding an old timestamped URL, use the current Gentoo Stage 3 download listed by
Gentoo for your installation date.

For example, once you've selected the current amd64 systemd desktop Stage 3, download it with:

```

wget '<CURRENT_STAGE3_URL>'

#Then extract:

tar xpvf stage3-*.tar.xz \
    --xattrs-include='*.*' \
    --numeric-owner

```

-Verify:

```
ls

```

-You should see:

```
#bash

bin
boot
dev
etc
home
lib
mnt
opt
proc
root
run
sbin
sys
tmp
usr
var

```


_Step 6 — Prepare the chroot_

-Bind /dev:

```
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev

```


-Bind /proc:

```
mount --rbind /proc /mnt/gentoo/proc
mount --make-rslave /mnt/gentoo/proc

```

-Bind /sys:

```
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys

```

-Bind /run:

```
mount --rbind /run /mnt/gentoo/run
mount --make-rslave /mnt/gentoo/run

```


-Copy DNS configuration:

```
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/resolv.conf

```


-Enter the Gentoo installation:

```
chroot /mnt/gentoo /bin/bash

```

-Load the Gentoo environment:

```
source /etc/profile

```


-Set the prompt:

```
export PS1="(chroot) $PS1"

```

Check:

```
ping -c 3 gentoo.org

```

_Step 7 — Configure /etc/portage/make.conf_

-Create/edit:

```
nano /etc/portage/make.conf

```

Use:

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




__Why Broadwell?__

The 2017 MacBook Air uses Intel Broadwell-era hardware, and `-march=broadwell` is appropriate
if your particular CPU is actually Broadwell.

-Confirm it:

```
lscpu | grep -E 'Model name|Architecture'

```


-You can also:

```
grep 'model name' /proc/cpuinfo | head -1

```

__**_If it reports a different CPU generation, don't use `-march=broadwell.`_**__

Also, `-O2` is intentional. Don't use `-O3` just because it sounds faster; for Gentoo system packages,
`-O2` is the sensible default.

__**_Step 8 — Sync Portage_**__

```
emerge-webrsync

```

- Update the repository metadata:

```
emerge --sync

```

__**_Step 9 — Select the systemd profile_**__

- List profiles:

```
eselect profile list

```

- Look for something resembling:

```
default/linux/amd64/23.0/systemd

```

__or the currently supported Gentoo systemd profile.__


- Set the appropriate index:

```
eselect profile set <INDEX>

```

- For example:

```
eselect profile set 4

```


Use the number actually shown by your eselect profile list; _don't copy_ `set 4` unless that's what your system shows.

-Verify:

```
eselect profile show

```

__**_Step 10 — Configure timezone_**__


- For your system:

```
echo "Africa/Lagos" > /etc/timezone

```

- Then run:

```
emerge --config sys-libs/timezone-data

```

- Verify:

```
#bash

date

```

__**_Step 11 — Configure locale_**__


- Edit:

```
# bash

nano /etc/locale.gen

```

- Make sure this line is enabled:

```
en_US.UTF-8 UTF-8

```

- Then:

```
#bash

locale-gen

```


- Select it:

```
#bash

eselect locale list

```

- Then:

```
#bash

eselect locale set en_US.utf8

```


- Update environment:

```
#bash

env-update
source /etc/profile

```

- Check:

- locale


__**_Step 12 — Install the required packages_**__

- Install `firmware, encryption, Btrfs, kernel and GRUB2` tools:


```
#bash

emerge \
    sys-kernel/linux-firmware \
    sys-fs/cryptsetup \
    sys-fs/btrfs-progs \
    sys-kernel/gentoo-kernel-bin \
    sys-boot/grub:2\
    firmware/intel-microcode


```



__**_Step 13 — Select the kernel_**__

- List kernels:

```
eselect kernel list

```


- Verify:

```
ls -l /usr/src/linux

```



- Check:

```
#bash

ls -lh /boot

```

__**_Step 14 - Install Filesystem Tools_**__

- Run:

```
emerge --ask sys-fs/e2fsprogs  # ext4
emerge --ask sys-fs/xfsprogs   # XFS
emerge --ask sys-fs/dosfstools # FAT/vfat (required for EFI partition)
emerge --ask sys-fs/btrfs-progs # Btrfs


```


__**_Step 16 — Get the actual UUIDs_**__


- Run:

```
#bash

blkid

```

__You need two different UUIDs:__

- EFI UUID

Something like:

```
/dev/sda1: UUID="XXXX-XXXX" TYPE="vfat"
LUKS UUID

```

- Something like:

```
/dev/sda2: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="crypto_LUKS"

```

- And the Btrfs filesystem has its own UUID:

```
blkid /dev/mapper/cryptroot

```

- You'll see:

```
TYPE="btrfs"
UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

```

Save these values somewhere temporarily.


__**_Step 17 — Configure /etc/crypttab_**__

- Create:

```
nano /etc/crypttab

```

- Put:

```
cryptroot UUID=<YOUR-LUKS-UUID> none luks

```

For example:

```
cryptroot UUID=12345678-abcd-1234-abcd-123456789abc none luks

```

NOTE! Use the UUID from /dev/sda2, not the Btrfs UUID.


- Check:

```
cat /etc/crypttab

```


__**_Step 18 — Configure /etc/fstab_**___

<details>
    <summary> __genfstab ships in the sys-fs/genfstab package (originally from Arch)__</summary>

```
emerge --ask sys-fs/genfstab

```

- Standard usage (run outside `chroot`):

```
# 1. Confirm all partitions are correctly mounted
lsblk
mount | grep /mnt/gentoo

# 2. Generate fstab (using UUID)
genfstab -U /mnt/gentoo >> /mnt/gentoo/etc/fstab

# 3. Check the generated file
cat /mnt/gentoo/etc/fstab

```

- If you've already `chrooted` into the new system, you can:

Run inside `chroot` (simplest)

```
emerge --ask sys-fs/genfstab
genfstab -U / >> /etc/fstab
vim /etc/fstab  # Check and clean up extra entries (e.g., /proc, /sys, /dev)

```
</details>


- Get the Btrfs UUID:

```
blkid /dev/mapper/cryptroot

```


- Then:


```
nano /etc/fstab

```


- Use:

```
# EFI
UUID=XXXX-XXXX  /efi        vfat    defaults,noatime                         0 2

# Btrfs root
UUID=YOUR-BTRFS-UUID  /            btrfs   noatime,compress=zstd:3,subvol=@          0 0

# Home
UUID=YOUR-BTRFS-UUID  /home        btrfs   noatime,compress=zstd:3,subvol=@home      0 0

# Snapshots
UUID=YOUR-BTRFS-UUID  /snapshots   btrfs   noatime,compress=zstd:3,subvol=@snapshots 0 0

# Swap subvolume
UUID=YOUR-BTRFS-UUID  /swap        btrfs   noatime,subvol=@swap                       0 0

# Swapfile
/swap/swapfile        none         swap    defaults                                  0 0
Important

```


- The Btrfs entries use:

`YOUR-BTRFS-UUID` not the `LUKS UUID`.

Systemd will unlock:


`LUKS UUID`
    ↓
`cryptroot`
    ↓
`Btrfs UUID`


__**_Step 19 — Configure the hostname_**__

```
echo "macgentoOser" > /etc/hostname

```

__**_Step 20 — Configure networking_**__

- Install network manager:

``` 
emerge --ask net-misc/networkmanager

```

Then enable it:

```
systemctl enable NetworkManager.service

```

__**_Step 22 — Install GRUB for the MacBook's UEFI_**__

First make sure the EFI directory is mounted:

- mountpoint /efi

Then:

```
grub-install \
    --target=x86_64-efi \
    --efi-directory=/efi \
    --bootloader-id=Gentoo

```


If successful, you'll get an installation-success message.


__**_Step 23 — Configure GRUB_**__

- Edit:

```
nano /etc/default/grub

```

- Use:

```
GRUB_DISTRIBUTOR="Gentoo"
GRUB_TIMEOUT=5
GRUB_ENABLE_CRYPTODISK=y
GRUB_CMDLINE_LINUX="rd.luks.uuid=YOUR-LUKS-UUID root=/dev/mapper/cryptroot rootfstype=btrfs rootflags=subvol=@ rw"

```


__**_Step 24 — Generate GRUB configuration_**__

```
grub-mkconfig -o /boot/grub/grub.cfg

```

- Check:

```
grep -E 'linux|root=' /boot/grub/grub.cfg | head

```

__**_Step 25 — Root password_**__


- Set your root password:

```
passwd

```

__**_Step 26 — Create a normal user_**__


- Don't run your desktop session as root.

For example:

```
useradd -m -G users,wheel,audio,video,input,plugdev -s /bin/bash optimist23

```

- Set password:

```
passwd optimist23

```


- Enable wheel access:

```
EDITOR=nano visudo

```

- Uncomment:

```
%wheel ALL=(ALL:ALL) ALL

```

__**_Step 28 — Check the installation before rebooting_**__

- Check your mounts:

```
findmnt

```

- Check Btrfs:

```
btrfs filesystem show

```

- Check subvolumes:

```
btrfs subvolume list /

```

- Check crypttab:

```
cat /etc/crypttab

```

- Check fstab:

```

mount -a
findmnt --verify
findmnt -t btrfs
cat /etc/fstab

```

- Check GRUB:

```
grep GRUB_CMDLINE_LINUX /etc/default/grub

```

- Check EFI:

```
find /efi -maxdepth 3 -type f

```

- Check kernel:

```
ls -lh /boot

```

- check USE FLAGS

```
emerge --info

```

__**_Step 29 — Leave the chroot_**__

```
exit

```


__**_Step 30 — Unmount everything_**__

- From the live environment:

```
swapoff -a

```

- Then:

```
umount -R /mnt/gentoo

```


- If something refuses to unmount:

```
umount -Rl /mnt/gentoo

```

- Check:

```
mount | grep gentoo

```

There should be nothing mounted under `/mnt/gentoo`.


__**_Step 31 — Close LUKS_**__

cryptsetup close cryptroot

- Verify:

```
ls /dev/mapper

```

cryptroot should no longer be there.


__**_Step 32 — Reboot_**__
reboot


    You should get:

    Mac EFI
       ↓
    Gentoo GRUB
       ↓
    LUKS password
       ↓
    cryptroot
       ↓
    Btrfs @
       ↓
    Gentoo systemd
    

Check that systemd is actually PID 1:

ps -p 1 -o comm=

Expected:

systemd

- Check the encrypted volume:

```
lsblk -f

```

- You should see approximately:

```
    sda
    ├─sda1      vfat
    └─sda2      crypto_LUKS
       └─cryptroot
          └─    btrfs


```          



- Check Btrfs:

```
findmnt -t btrfs

```


You should have:


```
    /
    ├── @
    ├── @home
    ├── @snapshots
    └── @swap

```


Check swap:

```
swapon --show
free -h

```

__One MacBook-specific issue: Broadcom Wi-Fi__



After the first boot, check:

```
lspci -nn | grep -i -E 'network|wireless'

```

and:

```
ip link

```



Final architecture

This corrected version gives you the setup you were aiming for:

                 MacBook Air 2017
                       │
                       ▼
                    UEFI
                       │
                       ▼
                 Gentoo GRUB
                       │
                       ▼
                    LUKS2
                       │
                /dev/mapper/cryptroot
                       │
                       ▼
                    Btrfs
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
       @             @home        @snapshots
        │
        ▼
      Gentoo
        │
        ▼
      systemd
        │
     networkmanager
        
        
       
        @swap
           │
           └── 4 GiB Btrfs swapfile

