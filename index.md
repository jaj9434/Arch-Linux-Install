# Arch Linux ARM VM Installation

## 1. Environment & Setup
**Host Machine:** MacBook Pro M2 (Apple Silicon)
**Hypervisor:** UTM
**ISO:** Archboot aarch64 (Arch Linux ARM)
**Virtual Disk:** 64 GB (`/dev/vda`)

**Important Note for this Installation:**
Because this is an M2 Mac, standard x86_64 installation guides do not apply. 
I used the **Archboot aarch64 ISO**. Archboot is a highly minimal live environment designed to be as small as possible. 
Unlike the standard Arch ISO, it does not include many essential installation tools (like `gdisk`, `e2fsprogs`, and `arch-install-scripts`). 
Therefore, a major challenge of this project involved manually installing these missing tools.  

## 2. Boot & Network Verification
Booted the ISO. Pressed `Ctrl+C` to exit the Archboot setup menu and drop to the root shell.
**Verify Network:** `ping -c 3 archlinux.org`
**Verify Disk:** `lsblk` (Confirmed target disk is `/dev/vda`)

## 3. Partitioning the Disks
**Issue:** `gdisk` command not found.
**Fix:** Archboot does not include partitioning tools. Installed it manually using `pacman -Sy gdisk`.
Partitioned the disk using `gdisk /dev/vda`:
`/dev/vda1`: 512M (EFI System, code ef00)
`/dev/vda2`: 2G (Linux Swap, code 8200)
`/dev/vda3`: Remaining space (Linux filesystem, code 8300)

## 4. Formatting & Mounting
**Issue:** `mkfs.fat` and `mkswap` commands not found because it did not installed. 
**Fix:** Installed missing filesystem tools using `pacman -Sy dosfstools e2fsprogs util-linux`.
**Commands Used:**
  `mkfs.fat -F32 /dev/vda1`,
  `mkswap /dev/vda2`,
  `swapon /dev/vda2`,
  `mkfs.ext4 -F /dev/vda3`
Next, I mounted the drives to /mnt and /mnt/boot/efi so the installer had a place to put the files.

## 5. Base Installation
**Issue:** I tried to run `pacstrap`, but got `command not found` again. 
**Fix:** I installed `arch-install-scripts` using pacman. 
I ran `pacstrap /mnt base linux linux-firmware vim sudo nano` to install the base system.
I generated the fstab file with `genfstab -U /mnt >> /mnt/etc/fstab`.

## 6. Chroot & Configuration
**Issue:** I tried `arch-chroot /mnt`, but it threw an `unshare` error. 
**Fix:** After some research, I found out this is a known bug in Archboot because the `unshare` tool doesn't work right on this specific ARM ISO.
I had to manually mount the virtual filesystems, install `coreutils`, and use the standard `chroot` command instead:
  `mount --bind /dev /mnt/dev`,
  `mount --bind /proc /mnt/proc`,
  `mount --bind /sys /mnt/sys`,
  `pacman -Sy coreutils`,
  `chroot /mnt /bin/bash`

**Another Issue:** Once I was inside the chroot, `pacman` couldn't connect to the internet. I realized the `chroot` command didn't copy over my DNS settings.
**Fix:** I fixed this by running `echo "nameserver 8.8.8.8" > /etc/resolv.conf`.

**System Settings:**
I set my timezone to `America/Chicago` (Oklahoma) and generated my locale (`en_US.UTF-8`).
I set the hostname to `arch-custom`.

## 7. Users & Sudo
I set the root password with `passwd`(Jj891091147!).
I created the two required users (`codi` and `Jack`):
  `useradd -m -G wheel -s /bin/bash codi`,
  passwd codi
  `useradd -m -G wheel -s /bin/bash Jack`,
  passwd Jack
I enabled sudo permissions by running `EDITOR=nano visudo` and removing the `#` from the `%wheel ALL=(ALL:ALL) ALL` line.

## 8. Bootloader & Networking (ARM Specific)
**Issue:** When I tried to install GRUB, it failed with `EFI variables are not supported on this system`. 
**Fix:** I learned that UTM doesn't use NVRAM the same way a physical Mac does. To fix this, I used the ARM target
AND added the `--removable` flag, which forces GRUB to install to a default fallback path that UTM can read:
  `pacman -S grub efibootmgr dosfstools mtools`,
  `grub-install --target=arm64-efi --efi-directory=/boot/efi --bootloader-id=ARCH --recheck --removable`,
  `grub-mkconfig -o /boot/grub/grub.cfg`
I installed SSH and NetworkManager, enabled them, and rebooted.

## 9. Post-Installation Modifications (GUI, Shell, Aliases)
I logged in as `Jack`.

### 9a. Shell & Color Coding
I installed Zsh and the color plugins:
  `sudo pacman -S zsh zsh-syntax-highlighting zsh-autosuggestions zsh-completions`,
  `chsh -s /usr/bin/zsh`,
I opened `~/.zshrc` with nano and added the source lines for the highlighting and auto-suggestions.
**Verify:** `echo $SHELL` shows `/usr/bin/zsh`.

### 9b. Aliases 
I added these aliases to the bottom of `~/.zshrc`:
  `alias ll='ls -alF'`,
  `alias update='sudo pacman -Syu'`,
  `alias ..='cd ..'`,
**Verify:** I ran `source ~/.zshrc` and typed `ll`. It worked and showed colors.
**Cautious:** The default shell is bash, ensure it is zsh. If not perform `chsh -s /usr/bin/zsh`.

### 9c. GUI Desktop (Req 1 & 6)
I installed Xorg, LXQt, and SDDM:
  `sudo pacman -S xorg xorg-server lxqt breeze-icons sddm`,
  `sudo systemctl enable sddm`,
**Verify:** I rebooted and the system booted straight into the SDDM graphical login screen. I logged into the LXQt desktop.

## 10. Final Verification Checklist
[Done] Custom Arch Linux built manually (No automated scripts).
[Done] LXQt GUI boots automatically via SDDM.
[Done] Users `codi` and `Jack` created and have sudo access.
[Done] Zsh installed with color coding.
[Done] SSH installed and running.
[Done] Aliases (`ll`, `update`, `..`) are active.
