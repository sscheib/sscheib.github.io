---
title: Upgrading Raspberry Pi OS 11 (Bullseye) to 12 (Bookworm)
author: Steffen Scheib
---
## Preface

This is going to be a short summary of the steps to do - mainly for my documentation :slightly_smiling_face:.

:warning: The [official blog](https://www.raspberrypi.com/news/bookworm-the-new-version-of-raspberry-pi-os) announcing the release of `Raspberry Pi OS` based on
`Debian Bookworm` **strongly** suggests to reinstall your `Raspberry Pi` instead of updating it in-place. **This procedure might break your `Raspberry Pi OS` installation.**

In case you are adventurous, below you'll find a way of in-place upgrading `Raspberry Pi OS` from `Bullseye` to `Bookworm` :sunglasses:.

## Prepare the upgrade

First, we'll need to update our system to the latest available `Bullseye` release:

```plaintext
sudo apt update && sudo apt upgrade && sudo apt dist-upgrade
```

:information_source: If you've updated any packages in the previous step, I'd recommend rebooting your system as precaution.

Next, we'll update all source lists in `/etc/apt/sources.list` and `/etc/apt/sources.list.d/` to match `bookworm`.

Most important are the repositories that provide the system packages (which are located in `/etc/apt/sources.list`) and those for the `Raspberry Pi` related packages, such as a
special kernel and specific firmware packages. These are located in `/etc/apt/sources.list.d/raspi.list`.

Let's first start with the system package repository located at `/etc/apt/sources.list`:

```plaintext
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

If you'd like to have source packages available (via `apt-get source`), add the following as well to `/etc/apt/sources.list`:

```plaintext
deb-src http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb-src http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

Now, on to the `Raspberry Pi` specific repository, defined in `/etc/apt/sources.list.d/raspi.list`:

```plaintext
deb http://archive.raspberrypi.org/debian/ bookworm main
```

Again, if you'd like to have source packages available, add the following additionally to `/etc/apt/sources.list.d/raspi.list`:

```plaintext
deb-src http://archive.raspberrypi.org/debian/ bookworm main
```

To ensure a smooth upgrade, ensure that the remainder of the repositories are updated to reflect the upgrade to `Bookworm` in `/etc/apt/sources.list.d/`. Depending on your
system configuration, there might be no other repositories. In my case I had there as well the repository for `Zabbix` (`/etc/apt/sources.list.d/zabbix.list`).

Lastly, update the cache:

```plaintext
sudo apt update
```

Ensure no error or warning messages are observed. If you encounter any errors or warnings, I'd encourage you to resolve them prior to continuing - I haven't encountered any.

## Migrating `/boot` to `/boot/firmware`

Starting from `Raspberry Pi OS 12`, we need to mount `/boot` to `/boot/firmware` [^1].

This can easily be done following the steps below:

1. Unmount `/boot` using `sudo umount /boot`
1. Create the directory `/boot/firmware` using `sudo mkdir /boot/firmware`
1. Adjust `/etc/fstab` so that the originally mount point `/boot` now points to `/boot/firmware`, e.g.:

    ```plaintext
    PARTUUID=ffffffff-ff    /boot/firmware  vfat    defaults,flush                  0       2
    ```

    :information_source: I've modified the `PARTUUID` above. Yours will look different, of course.

1. Ensure `systemd` is notified about this change using `sudo systemctl daemon-reload`
1. Finally, ensure that `/boot/firmware` is correctly mounted using `sudo mount -a`

## Install new kernel and remove the old kernel

We need to install the new kernel and the new firmware packages. After that we'll remove the old one. Yes, it sounds insane at first glance, but that's the only way I am aware of to
make it work :rofl:.

The required kernel is different for each `Raspberry Pi` model used and whether it's a 32 bit or 64 bit `Raspberry Pi`.

You can find out which chip your `Raspberry Pi` is based on using `lspci`. Below you'll find the output of one of my `Raspberry Pis`:

```terminal
$ lspci
00:00.0 PCI bridge: Broadcom Inc. and subsidiaries BCM2711 PCIe Bridge (rev 10)
01:00.0 USB controller: VIA Technologies, Inc. VL805/806 xHCI USB 3.0 Controller (rev 01)
```

Following table summarizes the `Raspberry Pi` models along with the old kernel path and the new kernel package name to the best of my knowledge:

| Model               | Chip      |  Bit  | Old kernel name and path        | New kernel package              |
| :-------------------| :-------- | :---- | :------------------------------ | :------------------------------ |
| `Raspberry Pi Zero` | `BCM2835` | 32    | `/boot/kernel.img`              | `linux-image-rpi-v6`            |
| `Raspberry Pi 1`    | `BCM2835` | 32    | `/boot/kernel.img`              | `linux-image-rpi-v6`            |
| `Raspberry Pi 2`    | `BCM2836` | 32    | `/boot/kernel7.img`             | `linux-image-rpi-v7`            |
| `Raspberry Pi 3`    | `BCM2837` | 32    | `/boot/kernel7.img`             | `linux-image-rpi-v7`            |
| `Raspberry Pi 4`    | `BCM2711` | 32    | `/boot/kernel7l.img`            | `linux-image-rpi-v7l`           |
| `Raspberry Pi 3`    | `BCM2837` | 64    | `/boot/kernel8.img`             | `linux-image-rpi-v8`            |
| `Raspberry Pi 4`    | `BCM2711` | 64    | `/boot/kernel8.img`             | `linux-image-rpi-v8`            |
| `Raspberry Pi 5`    | `BCM2712` | 64    | `/boot/kernel8.img`             | `linux-image-rpi-2712`          |

You should be able to easily figure out the correct new kernel package name using the column `Chip` together with `Old kernel name and path` from above table.

Now, up to the actual task.

Installing the new kernel and firmware is done using:

```shell
sudo apt install raspi-firmware linux-image-rpi-v8
```

:warning: Please substitute the package name according to above table.

Next, remove the old kernel - which is for all `Raspberry Pi` models the same:

```shell
sudo apt remove raspberrypi-kernel raspberrypi-bootloader
```

## Upgrading the operating system

The upgrade itself is straight-forward from this point:

1. Update once more the package lists using `sudo apt update` - just for good measure :sunglasses:
1. Run the actual upgrade using `sudo apt full-upgrade`

After the update has concluded, ensure no further packages need installing:

1. Refresh once more the package lists using `sudo apt update`
1. Upgrade any leftover packages using `sudo apt upgrade`

Lastly, cleanup obsolete data:

1. Remove obsolete packages using `sudo apt autoremove`
1. Clean the `apt` cache using `sudo apt clean`

Finally, reboot your system using `sudo systemctl reboot`.

Once the system is back up and running, you should find yourself in a `Raspberry Pi OS 12`, which can be verified by looking at `/etc/debian_version`:

```terminal
$ cat /etc/debian_version
12.5
```

:information_source: The version displayed in `/etc/debian_version` should be `12.x` - at the time of this writing `12.5`.

## Legacy `GPG` `trusted.gpg` `keyring`

When updating the package lists (using `sudo apt update`) after running the upgrade, you might encounter the following issue complaining about `GPG` keys stored in the
legacy `trusted.gpg` `keyring`:

```plaintext
W: http://security.debian.org/debian-security/dists/bookworm-security/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
W: http://deb.debian.org/debian/dists/bookworm/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
W: http://deb.debian.org/debian/dists/bookworm-updates/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
W: http://archive.raspberrypi.org/debian/dists/bookworm/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
W: https://deb.nodesource.com/node_20.x/dists/bookworm/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
```

This is fairly easy to fix.

There is the 'lazy' way, which basically copies the old legacy `keyring` to the new location:

```shell
sudo cp /etc/apt/trusted.gpg /etc/apt/trusted.gpg.d/
```

Alternatively, you can use following `BASH` script to export each key stored in the legacy `keyring` to a file in `/etc/apt/trusted.gpg.d/`:

```bash
#!/bin/bash
# exit immediately if a command fails
set -o errexit

# make the whole pipe fail if any commands within the pipe fail
set -o pipefail

# treats unset variables as an error while performing parameter expansion
# special parameters $* and $@ are not affected from this
set -o nounset

# set initial flag state
declare next=1
while read -r line; do
  # skip on lines not matching ^pub or next is not equal to 0
  (
    [[ "${line}" =~ ^pub ]] ||
    [[ "${next}" -eq 0 ]]
  ) || {
    continue;
  };

  # line matches ^pub
  [[ ! "${line}" =~ ^pub ]] || {
    # we'll need the next line
    declare next=0;
    continue;
  };

  # reset flag
  declare next=1

  [[ "$(sudo apt-key list "${line}" 2> /dev/null | grep -E '^uid')" =~ ^uid.+?\[.+?\][[:space:]]+(.+?)(\<.+?\>)?$ ]] || {
    echo "ERROR: Unable to extract uid of key with ID '${line}";
    exit 1;
  };

  # transform e.g. Mike Thompson (Raspberry Pi Debian armhf ARMv6+VFP) <mpthompson@gmail.com>
  # into e.g. mike_thompson_raspberry_pi_debian_armhf_armv6_vfp.gpg
  declare keyName="$(echo "${BASH_REMATCH[1]}" | sed -E 's/[[:space:]]+<.+>//' | sed -E 's/[^[:alnum:]]+/_/g' | tr '[[:upper:]]' '[[:lower:]]' | sed -E 's/_$//').gpg"

  sudo apt-key export "${line}" 2> /dev/null | sudo gpg --dearmour -o "/etc/apt/trusted.gpg.d/${keyName}" || {
    echo "ERROR: Failed exporting GPG key with ID '${line}' to '/etc/apt/trusted.gpg.d/${keyName}'";
    exit 1;
  };

done < <(sudo apt-key list --list-keys 2> /dev/null | grep -Pzo '(?m)/etc/apt/trusted.gpg[\S\s]+/etc/apt/trusted.gpg.d/')
# ^ match everything starting from /etc/apt/trusted.gpg (legacy keystore) to the next "modern" keystore (stored in /etc/apt/trusted.gpg.d/)
```

The result will be filenames like the following in `/etc/apt/trusted.gpg.d/`:

```terminal
root@example.com:~# ls -la /etc/apt/trusted.gpg.d/
total 80
drwxr-xr-x 2 root root 4096 Apr 15 11:07 .
drwxr-xr-x 9 root root 4096 Apr 14 22:14 ..
-rw-r--r-- 1 root root 5834 Apr 15 11:07 debian_archive_automatic_signing_key_10_buster.gpg
-rw-r--r-- 1 root root 5836 Apr 15 11:07 debian_archive_automatic_signing_key_11_bullseye.gpg
-rw-r--r-- 1 root root 5836 Apr 15 11:07 debian_archive_automatic_signing_key_12_bookworm.gpg
-rw-r--r-- 1 root root 4648 Apr 15 11:07 debian_archive_automatic_signing_key_9_stretch.gpg
-rw-r--r-- 1 root root 5845 Apr 15 11:07 debian_security_archive_automatic_signing_key_11_bullseye.gpg
-rw-r--r-- 1 root root 5845 Apr 15 11:07 debian_security_archive_automatic_signing_key_12_bookworm.gpg
-rw-r--r-- 1 root root  280 Apr 15 11:07 debian_stable_release_key_12_bookworm.gpg
-rw-r--r-- 1 root root 2760 Apr 15 11:07 docker_release_ce_deb.gpg
-rw-r--r-- 1 root root 1225 Apr 15 11:07 mike_thompson_raspberry_pi_debian_armhf_armv6_vfp.gpg
-rw-r--r-- 1 root root 2206 Apr 15 11:07 nodesource.gpg
-rw-r--r-- 1 root root 1183 Apr 15 11:07 raspberry_pi_archive_signing_key.gpg
-rwxr-xr-x 1 root root 1183 Sep 14  2022 zabbix-official-repo.gpg
```

The script leaves the legacy `keyring` (`/etc/apt/trusted.gpg`) untouched. It merely exports the keys *additionally* to `/etc/apt/trusted.gpg.d/`.

With that, the warning with regards to using a legacy `keyring` should be gone :slightly_smiling_face:.

## Conclusion

I hope you learned something today and did not break your `Raspberry Pi OS` installation :slightly_smiling_face:.

I upgraded four `Raspberry Pi 4 (64 bit)` and one `Raspberry Pi 3 (32bit)` using the procedure described in this blog.

Until next Time,

Steffen

## Change log

### 2024-04-15

- Initial release

## Footnotes

[^1]: [Pull request `Raspberry Pi` documentation](https://github.com/raspberrypi/documentation/issues/3089)
