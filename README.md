# Arch Linux installation guide

> [!CAUTION]
> Keep in mind that this is what works for me. Don't follow this guide
> mindlessly - what you require from an Arch installation will differ from what
> this guide can achieve.

This is how I install Arch on my PCs. Please note that this guide might not be
suitable for others' needs, as I gloss over some details and, among other
things, assume that you have the same functional needs as myself (e.g. using the
`eu` keyboard layout).

## Bootable USB

Start by downloading the Arch ISO on a USB stick. You can either flash it, or
use [Ventoy](https://www.ventoy.net/en/index.html). I use Ventoy.

To get started with Ventoy, download the Ventoy binary from [the Ventoy
website](https://www.ventoy.net/en/download.html). Make sure to choose
`ventoy-x.y.z-linux.tar.gz`.

```
cd Downloads
tar -xzf ventoy-x.y.z-linux.tar.gz
cd ventoy-x-y-z
sudo ./Ventoy2Disk.sh -i /dev/sdX
```

Use `lsblk` to identify the correct device (typically `/dev/sda` or `/dev/sdb`).

Now you can mount the largest partition of `/dev/sdX`, and transfer the Arch ISO
to it.

You now have a bootable Ventoy USB stick.

## Booting into a live environment

Insert the USB stick into the PC you want to install Arch on, and reboot it.\
You might need to change the boot order in BIOS/UEFI. On a ThinkPad, this is
done by spamming the Enter or F1 key upon startup. When you navigate to
"Startup", you can toggle the "Boot device List F12 Option", which allows you to
spam F12 upon startup and then choose to boot from the USB.

When greeted with the Ventoy splash screen, choose the Arch ISO, and "Boot in
Normal mode.

Now you are in a live Arch environment.

## Following the installation guide

Use the guide at [https://wiki.archlinux.org/title/Installation_guide](
https://wiki.archlinux.org/title/Installation_guide). Here are some notes about
the process:

### 1.5 Set the console keyboard and font

This isn't necessary to do yet, the keyboard layout is set in Hyprland - and the
font is fine by default. Keep in mind that these settings only will apply in the
TTY, so it will be overridden by our Hyprland config.

### 1.7 Connect to the internet

If on wireless, use `iwctl`:

```shell
iwctl
[iwd]# station wlan0 connect YOUR_SSID
```

### 1.8 Update the system clock

Ensure the time zone is correct:

```shell
timedatectl set-timezone Europe/Oslo
```

### 1.9 Partition the disks

Use `lsblk` instead of `fdisk -l` to see the devices, it is significantly
easier. Typically, the built-in disk is called `/dev/nvme0n1`, but this can
vary.

In `fdisk`, start by deleting all partitions using the command `d`.

Then, two new partitions. The first should have a size of 1GB, and the second
should take up the rest of the device. To achieve a size of 1GB, choose `+1G`
when specifying the last sector of the partition.

Then, write the changes with `w` (after checking that it looks correct with
`p`).

### 2.1 Select the mirrors

Move Norwegian mirrors to the top, delete all mirrors that are not in Norway,
Worldwide, or the US. Then, uncomment all mirrors that remain.

### 2.2 Install essential packages

Install `base linux linux-firmware man-db man-pages texinfo vim networkmanager`.
The rest of the packages can be installed later.

If this fails, run:

```shell
pacman -Syy
pacman -S archlinux-keyring
```

And then try again.

### 3.3 Time

Make sure to pick the correct timezone:

```shell
ln -sf /usr/share/zoneinfo/Europe/Oslo /etc/localtime
```

### 3.4 Localization

In `locale.gen`, uncomment `en_DK.UTF-8 UTF-8` and `en_US.UTF-8 UTF-8`.
`locale.conf` can be as suggested in the wiki installation guide.

### 3.5 Hostname

My preferred naming scheme is `[OS/distro][number]`, meaning the primary Arch
system should be called `arch1`, the secondary `arch2`, etc.

### 3.6 Initramfs

This step can be skipped

### 3.8 Boot loader

The most common boot loader is GRUB, which is why my systems will use it.

The first step is installing GRUB:

```shell
pacman -S grub efibootmgr
```

We can then use `grub-install`, and also add our OS to the boot menu:

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

If it still doesn't boot, try to boot to the live ISO and chroot (as described
earlier in the wiki install guide) - then run:

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --removable
```

## Post-installation basics

Do all of this while logged-in as `root`.

### Enable NetworkManager

```shell
systemctl enable --now NetworkManager.service
```

We can now connect to the network using `nmcli` or `nmtui`.

### Install zsh

```shell
pacman -S zsh
```

We will install other packages later, but for now we really only need zsh.

### Create a user

```shell
useradd --create-home --groups wheel --shell /usr/bin/zsh USERNAME_HERE 
passwd USERNAME_HERE
```

### Install and set up sudo

```shell
pacman -S sudo
EDITOR=/usr/bin/vim visudo
```

Then, uncomment the line `%wheel ALL=(ALL:ALL) ALL`. This allows users in the
`wheel` group to execute all commands.

## Post-installation as your user

Log in as the user you just created.

### Downloading dotfiles and this guide

```shell
sudo pacman -S git stow
git clone https://github.com/yrjarv/dotfiles
cd .dotfiles
mkdir -p ~/.config/test
stow */
git restore .
rmdir ~/.config/test
cd ~
git clone https://github.com/yrjarv/arch-install-guide
```

The reason for the `~/.config/test` directory is to prevent `stow` from owning
the entire `~/.config` directory - it should create symlinks inside `~/.config`
instead, because several programs use `~/.config` to share cache and similar
things I don't want to have version control on.

### Setting up pacman and installing packages

First, uncomment the `HookDir = /etc/pacman.d/hooks` line in `/etc/pacman.conf`.
Then, we can copy the hook that automatically keeps the `packages.txt` list
updated:

```shell
sudo mkdir /etc/pacman.d/hooks
sudo cp automatic_list.hook /etc/pacman.d/hooks/
```

Make sure to replace `USERNAME` with your username after copying the file.

Finally, install the AUR helper `yay`:

```shell
sudo pacman -S --needed git base-devel
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

We can now install all the packages by running:

```shell
cd ~/arch-install-guide
yay -S --needed - < packages.txt
```

### SSH keys

Now, we can generate SSH keys. First, ensure that `~/.ssh` is a directory and
not a symlink. If it is a symlink, you need to first `rm ~/.ssh`, and then
`mkdir ~/.ssh`.

```shell
sudo pacman -S openssh
cd .ssh
ssh-keygen -t ed25519 -N "" -C "USER@HOST-github" -f github
ssh-keygen -t ed25519 -N "" -C "USER@HOST-git" -f git
ssh-keygen -t ed25519 -N "" -C "USER@HOST-uio" -f uio
```

`ssh` into a UiO server, and copy the contents of `uio.pub` into
`~/.ssh/authorized-keys`

If you removed `.ssh` earlier, now is the time to create all the symlinks from
`.dotfiles` again:

```shell
cd ~/.dotfiles
stow */
```

### Git repos from GitHub

After starting Hyprland, we can start logging in to stuff in the browser - and
we can log in to GitHub and upload the newly generated SSH keys `github` and
`git` as respectively authentication and signing keys.

Once the keys are added, we change the remote in the two repos we currently have
downloaded:

```shell
cd ~/.dotfiles
git remote set-url origin github:yrjarv/.dotfiles
cd ~/arch-install-guide
git remote set-url origin github:yrjarv/arch-install-guide
```

We can now download all the repos we want, but the most necessary one is at
`github:yrjarv/.nvim`:

```shell
cd ~
git clone github:yrjarv/.nvim
cd .nvim
stow */
```

Then, follow the instructions from the `.nvim` repo to make the nvim config work
properly.

### Eduroam (not yet tested)

This is my best guess of how eduroam can be set up, but I have not yet been able
to test the connection on campus.

```shell
cd /tmp
git clone github:geteduroam/linux-app
cd linux-app
make build-cli
./geteduroam-cli
```

### Wireshark

Add your user to the `wireshark` group to get permission to listen to network
traffic:

```shell
sudo usermod USERNAME_HERE -aG wireshark
```

### Bluetooth

Make sure to enable `bluetooth.service`:

```shell
sudo systemctl enable --now bluetooth.service
```

### `/etc/hosts`

Ensure that `dns` is in the `/etc/hosts` list, this will save a lot of time when
pinging to check the internet connection.

```shell
sudo sh -c 'echo "1.1.1.1 dns" >> /etc/hosts'
```

