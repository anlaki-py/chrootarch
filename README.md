<img src="https://raw.githubusercontent.com/anlaki-py/chrootarch/main/Screenshots/arch2.png" alt="Alt Text"/>


> [!CAUTION]
> READ CAREFULLY! When using Chroot environments to exit completely close the Termux application even from the background apps or if necessary force close it. Otherwise, in case you do some command like "rm -rf chrootFolder" the device will go crazy and you will have to force reboot it.

> [!NOTE]
> My phone is a 6GB RAM [Redmi Note 10S](https://m.gsmarena.com/xiaomi_redmi_note_10s-10769.php) with ~~[LineageOS](https://lineageos.org/) Android 14.~~

# 📚 Index

## CHROOT

* 🏁 [First steps](#first-steps-chroot)
* 💻 [Setting Arch chroot](#arch-chroot)

<br>

---  
---  

</br>

## 🏁 First steps <a name=first-steps-chroot></a>

1. **First you need to have your device <u>rooted</u>.**
2. **You need to flash [Busybox](https://github.com/Magisk-Modules-Alt-Repo/BuiltIn-BusyBox/releases) with Magisk.**
3. **Then you need to install the following packages in Termux:** 

```bash
pkg update
pkg install x11-repo
pkg install root-repo
pkg install termux-x11-nightly
pkg update
pkg install tsu
pkg install pulseaudio
```

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

- **Create needed folders:**

```bash
mkdir media
mkdir media/sdcard
mkdir dev/shm
```

- **Create `resolv.conf` file, paste the following content and replace the original file**

```bash
vi resolv.conf
```

- > Type `i` to enter insert mode and past the following:

```bash
# Content
nameserver 9.9.9.9 # or use 8.8.8.8 (google DNS)
nameserver 8.8.4.4
```

- > Click on `ESC` to exit insert mode and then type `:wq` to save and exit.

```bash
mv -vf resolv.conf etc/run/systemd/resolve/resolv.conf
```

- **Create `hosts` file, paste the following content and replace the original file**

```bash
vi hosts
```

```bash
# Content
127.0.0.1 localhost
```

```bash
mv -vf hosts etc/hosts
```

- **Create the start script:**

```bash
cd /data/local/tmp
vi arch.sh
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

# Bind mount resolv.conf for DNS resolution
# if ! busybox mount --bind /etc/resolv.conf $mnt/run/systemd/resolve/resolv.conf; then
  # echo "Error: Failed to bind mount /run/systemd/resolve/resolv.conf."
  # exit 1
# fi

# Chroot into Arch
if ! busybox chroot $mnt /bin/su - root; then
  echo "Error: Failed to chroot into Arch."
  exit 1
fi
```

- **Make the script executable and run it. The prompt will change to `root@localhost`**

```bash
chmod +x arch.sh
sh arch.sh
```

- **Comment `CheckSpace` pacman config so we can install and update packages**

```bash
nano /etc/pacman.conf
```

Now comment the line where it says `CheckSpace` by placing the character `#` at the beginning

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
sudo pacman -Syu openssh
```

- **Create a new user, in this case `aki`**

```bash
groupadd storage
groupadd wheel
useradd -m -g users -G wheel,audio,video,storage,aid_inet -s /bin/bash aki
passwd aki
```

- Add the user to `sudoers` file so you can execute `sudo` commands

```bash
nano /etc/sudoers
```

```bash
# Paste this 
aki  ALL=(ALL:ALL) ALL
```

- **Fix locales to avoid weird characters:**

```bash
sudo nano /etc/locale.gen

# Uncomment en_US UTF-8 UTF-8
```

```bash
sudo locale-gen
```

```bash
sudo nano /etc/locale.conf
# Replace LANG=C with LANG=en_US.UTF-8
```


## Install XFCE4 Desktop

```bash
sudo pacman -S xfce4 xfce4-terminal
```

- **Exit chroot and make sure you are on the correct directory**

```bash
cd /data/local/tmp/
```

- **Create a new script to chroot into arch with xfce4**

```BASH
vi arch-xfce4.sh
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

# Bind mount resolv.conf for DNS resolution
# if ! busybox mount --bind /etc/resolv.conf $mnt/run/systemd/resolve/resolv.conf; then
  # echo "Error: Failed to bind mount /run/systemd/resolve/resolv.conf."
  # exit 1
# fi

# Chroot into Arch
if ! busybox chroot $mnt /bin/su - aki -c "export DISPLAY=:0 PULSE_SERVER=tcp:127.0.0.1:4713 && dbus-launch --exit-with-session startxfce4"; then
  echo "Error: Failed to chroot into Arch."
  exit 1
fi
```

- **Now fully exit Termux, reopen it again and create this two scripts `arch.sh` and `arch-xfce4.sh` on the the Home directory of Termux**

    - `arch.sh` is for chrooting into arch teminal (No GUI).

    - `arch-xfce4.sh` is for chrooting into arch with xfce4 DE.

```bash
touch arch.sh && touch arch-xfce4.sh
echo "su -c 'sh /data/local/tmp/arch-terminal.sh'" >> arch.sh
```

```bash
vi arch-xfce4.sh
```

```bash
#!/bin/bash
pkill -f com.termux.x11
killall -9 termux-x11 Xwayland pulseaudio virgl_test_server_android termux-wake-lock

am start --user 0 -n com.termux.x11/com.termux.x11.MainActivity

sudo busybox mount --bind $PREFIX/tmp /data/local/tmp/chrootarch/tmp

XDG_RUNTIME_DIR=${TMPDIR} termux-x11 :0 -ac &

sleep 3

pulseaudio --start --load="module-native-protocol-tcp auth-ip-acl=127.0.0.1 auth-anonymous=1" --exit-idle-time=-1
pacmd load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1 auth-anonymous=1

virgl_test_server_android &

sudo chmod -R 1777 /data/data/com.termux/files/usr/tmp # i put this as virgl kept throwing errors and this seemed to work

su -c "sh /data/local/tmp/arch-xfce4.sh"
```

## Caution Regarding Chroot Environments in Termux

When utilizing Chroot environments within Termux, it's crucial to follow proper exit procedures to prevent potential system instability. Failure to do so may lead to unexpected behavior and necessitate a forced reboot of your device.

**Recommended Exit Procedure:**

1. Upon completion of your work within the Chroot environment, ensure you exit the environment using the appropriate command (e.g., exit).


2. Completely close the Termux application, including removing it from the list of background applications.


3. If necessary, force close the Termux application through your device's application settings.



**Explanation:**

Chroot environments create an isolated filesystem within Termux. If the Termux application remains active in the background after exiting the Chroot environment, certain commands executed outside the Chroot (e.g., deleting the Chroot folder using `rm -rf`) can have unintended consequences on the main filesystem, potentially leading to system instability.

By completely closing Termux, you ensure that all processes associated with the Chroot environment are terminated, preventing any conflicts with subsequent commands.

> [!NOTE]
> Always exercise caution when working with Chroot environments and commands that modify the filesystem. It's recommended to back up important data before making any significant changes.

### Accessing Arch Linux Terminal (No GUI)

- You can chroot into an Arch Linux terminal environment (without a graphical user interface) using any terminal application with root access.
- This includes terminals like the one built into MT Manager.
- To do this, execute the script `arch.sh` located at `/data/local/tmp/`.

### Accessing Arch Linux with a Desktop Environment (XFCE4 or others)

- To use Arch Linux with a desktop environment like XFCE4, you need to execute the script `arch-xfce4.sh` located within your Termux home directory.

- Following these guidelines will help ensure a safe and stable experience when using Chroot environments in Android.

credit: [@LinuxDroidMaster](https://github.com/LinuxDroidMaster)
