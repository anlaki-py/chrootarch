<img src="https://raw.githubusercontent.com/anlaki-py/chrootarch/main/screenshots/arch.png" alt="Alt Text"/>
<img src="https://raw.githubusercontent.com/anlaki-py/chrootarch/main/screenshots/arch2.png" alt="Alt Text"/>

> [!CAUTION]
> READ CAREFULLY! When using Chroot environments to exit completely close the Termux application even from the background apps or if necessary force close it. Otherwise, in case you do some command like "rm -rf chrootFolder" the device will go crazy and you will have to force reboot it.

> [!NOTE]
> My phone is a 6GB RAM [Redmi Note 10S](https://m.gsmarena.com/xiaomi_redmi_note_10s-10769.php) with ~~[LineageOS](https://lineageos.org/) Android 14.~~

---
# CHROOTARCH

## First steps for arch CLI

1. **First you need to have your device <u>rooted</u>.**
2. **You need to flash [Busybox](https://github.com/Magisk-Modules-Alt-Repo/BuiltIn-BusyBox/releases) with Magisk.**
3. **Then you need to use any terminal that has root access. (e.g., Termux)**

---  

<br>

## 💻 Setting Arch chroot <a name=arch-chroot></a>

- **Enter Termux super user terminal with the command `su`**
- **Navigate to the folder where you want to install Arch Chroot and download the rootfs tar ball.**

```bash
cd /data/local/tmp
wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
```

- **Create a folder to uncompress the file:**

```bash
mkdir chrootarch
cd chrootarch

tar xvf /data/local/tmp/ArchLinuxARM-aarch64-latest.tar.gz --numeric-owner
```

- **Create the start script:**

```bash
cd /data/local/tmp
vi arch-terminal.sh
```

```bash
#!/bin/sh

mnt="/data/local/tmp/chrootarch"

# Remount /data with dev and suid options
if ! busybox mount -o remount,dev,suid /data; then
  echo "Error: Failed to remount /data with dev,suid options."
  exit 1
fi

# Ensure the rootfs path exists
if [ ! -d "$mnt" ]; then
  echo "Error: Arch rootfs path does not exist."
  exit 1
fi

# Create necessary directories if they don't exist
[ ! -d "$mnt/dev/shm" ] && mkdir -p $mnt/dev/shm
[ ! -d "$mnt/media/sdcard" ] && mkdir -p $mnt/media/sdcard
[ ! -d "$mnt/var/cache" ] && mkdir -p $mnt/var/cache

# Mount /dev if not already mounted
if ! mountpoint -q "$mnt/dev"; then
  if ! mount -o bind /dev $mnt/dev; then
    echo "Error: Failed to bind mount /dev."
    exit 1
  fi
fi

# Mount /proc if not already mounted
if ! mountpoint -q "$mnt/proc"; then
  if ! busybox mount -t proc proc $mnt/proc; then
    echo "Error: Failed to mount /proc."
    exit 1
  fi
fi

# Mount /sys if not already mounted
if ! mountpoint -q "$mnt/sys"; then
  if ! busybox mount -t sysfs sysfs $mnt/sys; then
    echo "Error: Failed to mount /sys."
    exit 1
  fi
fi

# Mount /dev/pts if not already mounted
if ! mountpoint -q "$mnt/dev/pts"; then
  if ! busybox mount -t devpts devpts $mnt/dev/pts; then
    echo "Error: Failed to mount /dev/pts."
    exit 1
  fi
fi

# Mount /sdcard if not already mounted
if ! mountpoint -q "$mnt/media/sdcard"; then
  if ! busybox mount -o bind /sdcard $mnt/media/sdcard; then
    echo "Error: Failed to bind mount /sdcard."
    exit 1
  fi
fi

# Mount /var/cache if not already mounted
if ! mountpoint -q "$mnt/var/cache"; then
  if ! busybox mount -t tmpfs /cache $mnt/var/cache; then
    echo "Error: Failed to mount /var/cache."
    exit 1
  fi
fi

# Mount /dev/shm if not already mounted
if ! mountpoint -q "$mnt/dev/shm"; then
  if ! busybox mount -t tmpfs -o size=256M tmpfs $mnt/dev/shm; then
    echo "Error: Failed to mount /dev/shm."
    exit 1
  fi
fi

# Create a default resolv.conf if it doesn't exist
if [ ! -f "$mnt/run/systemd/resolve/resolv.conf" ]; then
  mkdir -p "$mnt/run/systemd/resolve"  # Ensure the directory exists
  echo "nameserver 9.9.9.9" > "$mnt/run/systemd/resolve/resolv.conf"
  echo "nameserver 8.8.4.4" >> "$mnt/run/systemd/resolve/resolv.conf"
fi

# Create hosts file if it doesn't exist
if [ ! -f "$mnt/etc/hosts" ]; then
  echo "127.0.0.1 localhost" > "$mnt/etc/hosts"
fi

# Bind mount resolv.conf for DNS resolution
#if ! busybox mount --bind /etc/resolv.conf $mnt/run/systemd/resolve/resolv.conf; then
  #echo "Error: Failed to bind mount /run/systemd/resolve/resolv.conf."
  #exit 1
#fi

# Chroot into Arch
if ! busybox chroot $mnt /bin/su - root; then
  echo "Error: Failed to chroot into Arch."
  exit 1
fi
```

- **Make the script executable and run it. The prompt will change to `root@localhost`**

```bash
chmod +x arch-terminal.sh
sh arch-terminal.sh
```

- **Comment `CheckSpace` pacman config so we can install and update packages**

```bash
nano /etc/pacman.conf
```

- **Initialize pacman keys**

```bash
rm -r /etc/pacman.d/gnupg
pacman-key --init
pacman-key --populate archlinuxarm
pacman-key --refresh-keys
```

Tip: You can edit the mirrorlist and uncomment mirrors close to your location: `nano /etc/pacman.d/mirrorlist`

- **Execute some fixes**

```bash
groupadd -g 3003 aid_inet
groupadd -g 3004 aid_net_raw
groupadd -g 1003 aid_graphics
usermod -G 3003 -a root
```

- **Upgrade the system and install common tools**

```bash
pacman -Syu
pacman -S vim net-tools sudo git
```

- Optional.

```bash
pacman -Syu openssh
```

- Set root password 
```bash
passwd root
```

- **Create a new user, in my case `aki`**

```bash
groupadd storage
groupadd wheel
useradd -m -g users -G wheel,audio,video,storage,aid_inet -s /bin/bash aki
```

- Set password for user

```bash
passwd aki
```

- Add the user to `sudoers` file so you can execute `sudo` commands

```bash
nano /etc/sudoers
```

- Paste this under `root  ALL=(ALL:ALL) ALL` 

```bash 
aki  ALL=(ALL:ALL) ALL
```

- Fix locales to avoid weird characters by uncommenting `en_US.UTF-8 UTF-8`

```bash
nano /etc/locale.gen
```

```bash
locale-gen
```

- Replace `LANG=C` with `LANG=en_US.UTF-8`

```bash
nano /etc/locale.conf
```

- Enter the new user
```bash
su aki
```

> [!NOTE]
> Edit the `arch-terminal.sh` at the bottom of the script in `# Chroot into Arch`, replace `root` with `aki` (or your user name) to directly chroot to that user (recommended)

```bash
# Chroot into Arch
if ! busybox chroot $mnt /bin/su - aki; then
  echo "Error: Failed to chroot into Arch."
  exit 1
fi
```

---
