---
title: Manually installing Debian 11 (Bullseye) with fully encrypted LUKS (besides /boot) using debootstrap
author: Steffen Scheib
---
## Introduction

In this post, we'll cover the installation and configuration of a Debian 11 (Bullseye) system within a live environment.

Such a live system could be the [Hetzner rescue mode](https://docs.hetzner.com/robot/dedicated-server/troubleshooting/hetzner-rescue-system/), or any other live CD based on
Debian, such as [SystemRescue](https://www.system-rescue.org/) [formerly known as *SystemRescueCd*]).
In this example we are going to use the Hetzner rescue mode.

## Prerequisites

The only real prerequisite is, that you have booted your server and are logged in to your live CD environment as root.

## Preparing the disks of the server: Partitioning the drives

The first step is to partition the drives we are going to use using `gdisk` (in this example), `fdisk` or anything similar.
We are going to configure three partitions:

1. `BIOS boot partition` (**32 MB**, although 1 MB would be theoretically enough, type `EF02`).
   This partition is necessary for GPT partitions in order to load the second stage of GRUB.
1. `/boot partition` (**1024 MB - 4096MB**, type `8300`):
   In this partition the boot files will be stored. In order to be able to boot a machine, this partition needs to be unencrypted.
   If you are not planning on rebooting very frequently (to apply kernel updates), consider using 4096MB for the partition size;
   Otherwise you may need to do manual cleanups in order to install new kernel versions.
1. The remaining space will be put in this partition (type `8300` as well), which is later holding all our **encrypted** data

<!-- markdownlint-disable MD033 -->
<details>
<summary><i>Example output of <b>gdisk</b></i></summary>

```terminal
root@rescue ~ # gdisk /dev/sda 
GPT fdisk (gdisk) version 1.0.3

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries.

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): y

Command (? for help): n
Partition number (1-128, default 1): 
First sector (34-937406430, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-937406430, default = 937406430) or {+-}size{KMGTP}: +32M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): ef02
Changed type of partition to 'BIOS boot partition'

Command (? for help): n
Partition number (2-128, default 2): 
First sector (34-937406430, default = 67584) or {+-}size{KMGTP}: 
Last sector (67584-937406430, default = 937406430) or {+-}size{KMGTP}: +4096M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): n
Partition number (3-128, default 3): 
First sector (34-937406430, default = 8456192) or {+-}size{KMGTP}: 
Last sector (8456192-937406430, default = 937406430) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sda.
The operation has completed successfully.
root@rescue ~ #
```

</details>
<br>
<!-- markdownlint-disable MD033 -->

At the end we will have partitions, which should look similar to the following output (besides the last partition size, which depends - of course - on the overall disk size):

```terminal
root@rescue ~ # gdisk -l /dev/sda
GPT fdisk (gdisk) version 1.0.3
[..]
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048           67583   32.0 MiB    EF02  BIOS boot partition
   2           67584         8456191   4.0 GiB     8300  Linux filesystem
   3         8456192       937406430   443.0 GiB   8300  Linux filesystem
root@rescue ~ #
```

## Optional: Partition another drive to store your (application) data to keep it separate from the system disk

In this case we are just going to partition the whole drive `/dev/sdb` as a single partition using `type 8300`.
<!-- markdownlint-disable MD033 -->
<details>
<summary><i>Sample</i> output of <b>gdisk</b></summary>

```terminal
root@rescue ~ # gdisk /dev/sdb
GPT fdisk (gdisk) version 0.8.10

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries.

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): n
Partition number (1-128, default 1): 
First sector (34-11719933918, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-11719933918, default = 11719933918) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sdb.
The operation has completed successfully.
Just for the sake of completeness the output of gdisk showing the partitioning of /dev/sdb
Partitioned disk output  
root@rescue ~ # gdisk -l /dev/sdb
GPT fdisk (gdisk) version 0.8.10

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.
Disk /dev/sdb: 11719933952 sectors, 5.5 TiB
Logical sector size: 512 bytes
Disk identifier (GUID): EFC18675-65D1-47FB-BD1D-BC2FB3AC2FC4
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 11719933918
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048     11719933918   5.5 TiB     8300  Linux filesystem
root@rescue ~ #
```

</details>
<!-- markdownlint-enable MD033 -->

## Formatting partitions and setting up the encrypted LVM

The goal of this section is to end up with a partition layout using the [Logical Volume Manager](https://en.wikipedia.org/wiki/Logical_volume_management) (LVM).
In order to achieve this we are going to create an encrypted [Linux Unified Key Setup](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) (LUKS) partition on which we will
create our LVM.
The partition table should look ideally similar to the one in the table below. As I am currently setting up a server to use with [Proxmox](https://www.proxmox.com) the sizing
might be different from your choice.

I inserted an additional column where I set an **X** whether the partition is recommended to create on any system or whether the partition is specific for the usage with
Proxmox.

In my case the disk, where my system is going to get installed on (`/dev/sda`) is roughly 420GB big. So it is easily possible at any time to extend the current partition
layout or even extend the size of the different point mounts - thanks to LVM!

| mount point       | filesystem type | size | recommended | volume group |
| :---------------- | :-------------: | :--: | :---------: | :----------: |
| /                 | XFS             | 16GB | X           | vg_system    |
| /home             | XFS             | 4GB  | X           | vg_system    |
| /tmp              | XFS             | 4GB  | X           | vg_system    |
| /var              | XFS             | 32GB | X           | vg_system    |
| /var/tmp          | XFS             | 4GB  | X           | vg_system    |
| /var/log          | XFS             | 8GB  | X           | vg_system    |
| /var/log/audit    | XFS             | 2GB  |             | vg_system    |
| swap              | swap            | 16GB | X           | vg_system    |
| /var/lib/vz       | XFS             | 5TB  | -           | vg_data      |

## Formatting /boot

After creating the partitions on the system disk (`/dev/sda`) earlier, we are going to format the second partition (which we will use as `/boot`) using `XFS` as filesystem.
For that we'll use `mkfs.xfs /dev/sda2`:

```terminal
root@rescue ~ # mkfs.xfs /dev/sda2 
meta-data=/dev/sda2              isize=512    agcount=4, agsize=262144 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=1048576, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
root@rescue ~ # 
```

## Creating a LUKS partition to hold our system partitions

Next, we want to have all remaining partitions (e.g. `/`, `/home`, `/tmp`, etc.) within an encrypted LUKS partition. In this current case we have three partitions on the device `/dev/sda`:

- `/dev/sda1`: BIOS boot partition
- `/dev/sda2`: Unencrypted `/boot` partition (which we formatted in the section earlier)
- `/dev/sda3`: This is the partition we want to encrypt with LUKS

In order to encrypt the partition, following command is executed:

```plaintext
cryptsetup -s 512 -c aes-xts-plain64 luksFormat /dev/sda3
```

:warning: First, you need to confirm, that **all data on this partition will be lost**.
After you confirmed it, you will be prompted to enter a password for the encryption (please use a **complex** and **unique** password!) and repeat the password.

:warning: Make sure to save the password in your password safe (or remember it :open_mouth: ), otherwise you will not be able to access the system ever again - all data will
be lost. **There is no way to recover them**.

```terminal
root@rescue ~ # cryptsetup -s 512 -c aes-xts-plain64 luksFormat /dev/sda3

WARNING!
========
This will overwrite data on /dev/sda3 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/sda3: 
Verify passphrase: 
root@rescue ~ # 
```

Quickly verify, whether the encrypted LUKS partition is setup properly using the following command:

```plaintext
cryptsetup luksOpen /dev/sda3 crypted_system
```

The system will ask you for the password to decrypt the LUKS partition. After you entered the correct password, you will be able to see a new device:
`/dev/mapper/crypted_system`

```terminal
root@rescue ~ # cryptsetup luksOpen /dev/sda3 crypted_system
Enter passphrase for /dev/sda3: 
root@rescue ~ # ls -la /dev/mapper/
total 0
drwxr-xr-x  2 root root      80 Sep  5 19:40 .
drwxr-xr-x 15 root root    7.5K Sep  5 19:40 ..
crw-------  1 root root 10, 236 Sep  5 19:40 control
lrwxrwxrwx  1 root root       7 Sep  5 19:40 crypted_system -> ../dm-0
root@rescue ~ # 
```

### Optional: Creating LUKS partition for the data partition

Basically, the same steps as we used for the LUKS partition that holds our system partitions have to be applied for the data partition.
The differences are only a few simple things:

- The device is now `/dev/sdb`
- We will encrypt the device as a whole and not creating partitions before hand - simply because we don't need to. On the system drive (`/dev/sda`) we needed to create three
  partitions for both the BIOS boot and the unencrypted `/boot` partition. In this case we will put encrypted data on the whole drive.
- With `cryptsetup luksOpen` we need to specify a different name for the device, which holds the decrypted data: `crypted_data`
- **Ideally** you want to use a different password as for the system LUKS partition. We will later replace the password with a key file on the encrypted root file system in
  order to make the unlocking of the system during the boot easier

## Setting up LVM: Creating a physical volume, a volume group and several logical volumes for the encrypted LUKS partition

In order to implement and use LVM we need to follow the following approach:

1. Create a [physical volume](https://tldp.org/HOWTO/LVM-HOWTO/pv.html) (PV) using pvcreate on top of the decrypted LUKS partition
1. Create a [volume group](https://tldp.org/HOWTO/LVM-HOWTO/vg.html) (VG) using vgcreate on top of the physical volume
1. Create several [logical volumes](https://tldp.org/HOWTO/LVM-HOWTO/lv.html) (LV) using lvcreate on top of the volume group

## Creating a physical volume on top of the LUKS partition

The first step is to create a physical volume on top of the LUKS partition. This is very simple and does not need any further explanation.

Following command is used to create the physical volume:

```plaintext
pvcreate /dev/mapper/crypted_system
```

The output will look similar to the following:

```terminal
root@rescue ~ # pvcreate /dev/mapper/crypted_system
  Physical volume "/dev/mapper/crypted_system" successfully created.
root@rescue ~ # 
```

## Creating a volume group on top of the physical volume

The second step is to create a volume group on top of the just created physical volume. In this case we are going to use `vg_system` as the name of the volume group.
Following command is used to create a volume group:

```plaintext
vgcreate vg_system /dev/mapper/crypted_system
```

The output will look similar to the following:

```terminal
root@rescue ~ # vgcreate vg_system /dev/mapper/crypted_system
  Volume group "vg_system" successfully created
root@rescue ~ # 
```

## Creating logical volumes on top of the volume group

The last step is to create logical volumes on top of the just created volume group. Following command is used to create a logical volume:

```plaintext
lvcreate -L <size> -n <name> <volume_group_name>
```

For example:

```plaintext
lvcreate -L 16G -n root vg_system
```

In the above example a logical volume with the size of **16 GB** and the name **root** in the volume group **vg_system** is created.

This command can be used to create all logical volumes accordingly. Best practice - regarding the naming - is to use the name of the mount point (e.g. `/` = **root**,
`/tmp` = **tmp**, etc.).

If the mount point contains slashes, replace them via underscore (e.g. `/var/log` = **var_log**, `/var/tmp` = **var_tmp**, `/var/log/audit` = **var_log_audit**, etc.).
This is the best practice approach, which I have implemented on many servers/infrastructures.

The output will look similar to the following:

```terminal
root@rescue ~ # lvcreate -L 16G -n root vg_system
  Logical volume "root" created.
root@rescue ~ # lvcreate -L 4G -n home vg_system
  Logical volume "home" created.
root@rescue ~ # lvcreate -L 4G -n tmp vg_system
  Logical volume "tmp" created.
root@rescue ~ # lvcreate -L 32G -n var vg_system
  Logical volume "var" created.
root@rescue ~ # lvcreate -L 4G -n var_tmp vg_system
  Logical volume "var_tmp" created.
root@rescue ~ # lvcreate -L 8G -n var_log vg_system
  Logical volume "var_log" created.
root@rescue ~ # lvcreate -L 2G -n var_log_audit vg_system
  Logical volume "var_log_audit" created.
root@rescue ~ # lvcreate -L 16G -n swap vg_system
  Logical volume "swap" created.
root@rescue ~ #
```

And will leave us with following logical volumes:

```terminal
root@rescue ~ # lvs
  LV            VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home          vg_system -wi-a-----  4.00g                                                    
  root          vg_system -wi-a----- 16.00g                                                    
  swap          vg_system -wi-a----- 16.00g                                                    
  tmp           vg_system -wi-a-----  4.00g                                                    
  var           vg_system -wi-a----- 32.00g                                                    
  var_log       vg_system -wi-a-----  8.00g                                                    
  var_log_audit vg_system -wi-a-----  2.00g                                                    
  var_tmp       vg_system -wi-a-----  4.00g                                                    
root@rescue ~ #
```

.. and plenty of space left in the volume group:

```terminal
root@rescue ~ # vgs
  VG        #PV #LV #SN Attr   VSize    VFree   
  vg_system   1   8   0 wz--n- <442.94g <356.94g
root@rescue ~ #
```

### Optional: Create LVM for the data disk

As for the system partition, the same approach needs to be done for the data disk. As I briefly explained the exact approach and implementation above, here just the command
output:

```terminal
root@rescue ~ # pvcreate /dev/mapper/crypted_data 
  Physical volume "/dev/mapper/crypted_data" successfully created
root@rescue ~ # vgcreate vg_data /dev/mapper/crypted_data 
  Volume group "vg_data" successfully created
root@rescue ~ # lvcreate -L 5T -n var_lib_vz vg_data
  Logical volume "var_lib_vz" created
root@rescue ~ #
```

## Create filesystems on logical volumes

In order to use the logical volumes, we need to create file systems on them. To ease this process - and save me some manual work - I wrote a little BASH script.
The purpose of this script is to create XFS filesystems on all logical volumes on all volume groups specified and create a swap "filesystem" on the logical volume, which is
named `<vg>-swap`. Ff you have multiple swaps or a different naming, you can easily modify this script.

I added a semicolon after each command, so one can simply copy/paste the whole script into the terminal.

```bash
#!/bin/bash
# define the filesystem to use here
declare -r filesystem_type="xfs"
# define your volume groups here
for vg in vg_system vg_data; do
  # iterate over all LVs
  for lv in "/dev/mapper/${vg}-"*; do
    if [[ "$(basename "${lv}")" =~ ^${vg}-swap$ ]]; then
      # create a swap
      mkswap "${lv}";
    else
      # create filesystem
      mkfs."${filesystem_type}" -f "${lv}";
    fi;
  done; 
done
```

<!-- markdownlint-disable MD033 -->
<details>
<summary>Following a sample output:</summary>

```terminal
root@rescue ~ # for vg in vg_system; do
>   # iterate over all LVs
>   for lv in "/dev/mapper/${vg}-"*; do
>     if [[ "$(basename "${lv}")" =~ ^${vg}-swap$ ]]; then
>       # create a swap
>       mkswap "${lv}";
>     else
>       # create filesystem with volume label
>       mkfs."${filesystem_type}" -f "${lv}";
>     fi;
>   done; 
> done
meta-data=/dev/mapper/vg_system-home isize=512    agcount=4, agsize=262144 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=1048576, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
meta-data=/dev/mapper/vg_system-root isize=512    agcount=4, agsize=1048576 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=4194304, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
mkswap: /dev/mapper/vg_system-swap: warning: wiping old swap signature.
Setting up swapspace version 1, size = 16 GiB (17179865088 bytes)
no label, UUID=a2f49117-66f0-489b-9092-6548c272766d
meta-data=/dev/mapper/vg_system-tmp isize=512    agcount=4, agsize=262144 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=1048576, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
meta-data=/dev/mapper/vg_system-var isize=512    agcount=4, agsize=2097152 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=8388608, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=4096, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
meta-data=/dev/mapper/vg_system-var_log isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
meta-data=/dev/mapper/vg_system-var_log_audit isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
meta-data=/dev/mapper/vg_system-var_tmp isize=512    agcount=4, agsize=262144 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=1048576, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
root@rescue ~ #
```

</details>
<!-- markdownlint-enable MD033 -->

Note: If you have volume groups defined in the script, which are non-existent, you will retrieve some error messages, like the following (obviously `vg_data` did not exist):

```terminal
Error accessing specified device /dev/mapper/vg_data-*: No such file or directory
Usage: mkfs.xfs
/* blocksize */         [-b size=num]
/* metadata */          [-m crc=0|1,finobt=0|1,uuid=xxx,rmapbt=0|1,reflink=0|1]
/* data subvol */       [-d agcount=n,agsize=n,file,name=xxx,size=num,
                            (sunit=value,swidth=value|su=num,sw=num|noalign),
                            sectsize=num
/* force overwrite */   [-f]
/* inode size */        [-i log=n|perblock=n|size=num,maxpct=n,attr=0|1|2,
                            projid32bit=0|1,sparse=0|1]
/* no discard */        [-K]
/* log subvol */        [-l agnum=n,internal,size=num,logdev=xxx,version=n
                            sunit=value|su=num,sectsize=num,lazy-count=0|1]
/* label */             [-L label (maximum 12 characters)]
/* naming */            [-n size=num,version=2|ci,ftype=0|1]
/* no-op info only */   [-N]
/* prototype file */    [-p fname]
/* quiet */             [-q]
/* realtime subvol */   [-r extsize=num,size=num,rtdev=xxx]
/* sectorsize */        [-s size=num]
/* version */           [-V]
                        devicename
<devicename> is required unless -d name=xxx is given.
<num> is xxx (bytes), xxxs (sectors), xxxb (fs blocks), xxxk (xxx KiB),
      xxxm (xxx MiB), xxxg (xxx GiB), xxxt (xxx TiB) or xxxp (xxx PiB).
<value> is xxx (512 byte blocks).
root@rescue ~ # 
```

## Prepare for the installation

After we created our filesystems, we need to prepare the system for the manual installation.

### Mounting the partitions

In order to install our system within our live system, we need to mount the just created partitions under `/mnt`.
To easy this process again - and save me some manual work again - I created a script for this purpose.
To be able to use this script, we need to "close", both the volume group `vg_system` and - if you created - the volume group `vg_data` and afterwards close the LUKS partition.
This can be done using the following commands:

```plaintext
# "close" the volume group
lvchange -a n vg_system

# close the LUKS partition
cryptsetup luksClose /dev/mapper/crypted_system 
```

The output (or well, no output) will look similar to this (depending on - as already mentioned - whether you have chosen to create `vg_data` or not):

```terminal
root@rescue ~ # lvchange -a n vg_system
root@rescue ~ # cryptsetup luksClose /dev/mapper/crypted_system 
root@rescue ~ #
```

You can verify, whether we “closed” the volume group and the LUKS partition with the following commands:

```terminal
root@rescue ~ # lvs
root@rescue ~ # ls -la /dev/mapper/
total 0
drwxr-xr-x  2 root root      60 Sep  5 20:03 .
drwxr-xr-x 15 root root    7.5K Sep  5 20:03 ..
crw-------  1 root root 10, 236 Sep  5 19:40 control
root@rescue ~ #
```

To explain the general approach a bit, this is what the following script is doing:

- Unlock both the system and data LUKS partition (if defined) - will ask for a password obviously :slightly_smiling_face:
- In order to install a system within a live system, we need to mount the root LV somewhere - in our case it's `/mnt`
- To be able to mount the LVs we need to create the necessary directories beforehand (e.g. `/mnt/var`, `/mnt/var/log`, `/mnt/home`, etc.)
- Finally the LVs get mounted to those created directories

```bash
#!/bin/bash
# name of the volume group for the system
declare -r __VG_SYSTEM="vg_system"
# name of the volume group for the data partition - leave empty if you don't have it
declare -r __VG_DATA="vg_data"
# mount point where the chroot environment will be mounted
declare -r __DESTINATION_PARENT="/mnt"
# name of the logical volume, which contains the root ( / ) partition
declare -r __ROOT_LV_NAME="root"
# device where the /boot partition is stored on
declare -r __BOOT_DEVICE="/dev/sda2"
# device where the system LUKS partition is stored on
declare -r __SYSTEM_CRYPT_DEVICE="/dev/sda3"
# name of the LUKS partition after unlocking it (/dev/mapper/<NAME>)
declare -r __SYSTEM_CRYPT_NAME="crypted_system"
# device where the data LUKS partition is stored on - leave empty if you don't have it
declare -r __DATA_CRYPT_DEVICE="/dev/sdb"
# name of the LUKS partition after unlocking it (/dev/mapper/<NAME>) - leave empty if you don't have it
declare -r __DATA_CRYPT_NAME="crypted_data"

# function to mount all partitions
# if the script was called with a second parameter also all
# necessary system partitions (/dev /dev/pts etc) are mounted as well
function mount_chroot () {
  echo "Trying to decrypt system crypt device '${__SYSTEM_CRYPT_DEVICE}'"
  cryptsetup luksOpen "${__SYSTEM_CRYPT_DEVICE}" "${__SYSTEM_CRYPT_NAME}" || {
    echo "Decrypting '${__SYSTEM_CRYPT_DEVICE}' failed!";
    exit 1;
  };
  echo "Successful!"
 
  ( [[ -z "${__DATA_CRYPT_DEVICE}" ]] &&
    [[ -z "${__DATA_CRYPT_NAME}" ]]
  ) || {
    echo "Trying to decrypt data crypt device '${__DATA_CRYPT_DEVICE}'"
    cryptsetup luksOpen "${__DATA_CRYPT_DEVICE}" "${__DATA_CRYPT_NAME}" || {
      echo "Decrypting '${__DATA_CRYPT_DEVICE}' failed!";
      exit 1;
    };
    echo "Successful!"
  };
 
  # sleep to prevent that the VGs cant be detected yet
  sleep 2
  # detect vgs and switch to them
  vgchange -aay || {
    echo "Searching and activating volume groups failed!";
    exit 1;
  };
 
  # check whether the given root lv exist
  [[ -e "/dev/mapper/${__VG_SYSTEM}-${__ROOT_LV_NAME}" ]] || {
    echo "root LV '/dev/mapper/${__VG_SYSTEM}-${__ROOT_LV_NAME}' does not exist!";
    exit 1;
  };
 
  # mount the root lv and check whether it has been mounted successfully
  mount "/dev/mapper/${__VG_SYSTEM}-${__ROOT_LV_NAME}" "${__DESTINATION_PARENT}"
  mountpoint -q "${__DESTINATION_PARENT}" || {
    echo "Destination root '${__DESTINATION_PARENT}' is not mounted!"
    exit 1;
  };
 
  ( [[ -e ${__DESTINATION_PARENT}/boot ]] && 
    [[ -d ${__DESTINATION_PARENT}/boot ]]
  ) || {
  # create /boot within __DESTINATION_PARENT if it does not exist
    mkdir "${__DESTINATION_PARENT}/boot"
  };
  # try mounting /boot
  mount "${__BOOT_DEVICE}" "${__DESTINATION_PARENT}/boot"
 
  # go through all system LVs and mount them, unless its the root or swap partition
  for lv in "/dev/mapper/${__VG_SYSTEM}-"*; do
    declare part="$(echo "$(basename "${lv}")" | sed -e 's/vg_.*-//' -e 's/_/\//g')"
    case "${part}" in
      root)
        # we mounted it already
        continue
      ;;
      swap)
        # set swap to the current LV
        swapon "${lv}"
      ;;
      *)
        # create the necessary folders for the current LV
        # if they don't exist yet
        ( [[ -e "${__DESTINATION_PARENT}/${part}" ]] &&
          [[ -d "${__DESTINATION_PARENT}/${part}" ]]
        ) || {
          mkdir -p "${__DESTINATION_PARENT}/${part}"
        };
        # try mounting the LV
        mount "${lv}" "${__DESTINATION_PARENT}/${part}"
      ;;
    esac
  done
 
  [[ -z "${__VG_DATA}" ]] || {
    echo "Data partition defined :3"
    # do the same as for the system LVs for the data LVs
    for lv in "/dev/mapper/${__VG_DATA}-"*; do
      declare part="$(echo "$(basename "${lv}")" | sed -e 's/vg_.*-//' -e 's/_/\//g')"
      ( [[ -e "${__DESTINATION_PARENT}/${part}" ]] &&
        [[ -e "${__DESTINATION_PARENT}/${part}" ]]
      ) || {
        mkdir -p "${__DESTINATION_PARENT}/${part}"
      };
      mount "${lv}" "${__DESTINATION_PARENT}/${part}"
    done
  };
 
 
  [[ -n "${2}" ]] || {
    exit 0;
  };
 
  echo "Mounting necessary system partitions to chroot"
  mount -o bind /dev "${__DESTINATION_PARENT}/dev"
  mount -o bind /run "${__DESTINATION_PARENT}/run"
  mount -t devpts devpts "${__DESTINATION_PARENT}/dev/pts"
  mount -t sysfs sys "${__DESTINATION_PARENT}/sys"
  mount -t proc proc "${__DESTINATION_PARENT}/proc"
  exit 0;
} #; function mount_chroot ( )
 
# function to unmount all partitions defined under __DESTINATION_PARENT
# and "close" the VGs and finally close the LUKS partitions
function umount_chroot () {
  umount -lf "${__DESTINATION_PARENT}"
  for lv in "/dev/mapper/${__VG_SYSTEM}-"*; do
    declare part="$(echo "$(basename "${lv}")" | sed -e 's/vg_.*-//' -e 's/_/\//g')"
    case "${part}" in
      swap)
        swapoff "${lv}"
      ;;
      *)
        continue
      ;;
    esac
  done
  lvchange -a n "${__VG_SYSTEM}"
 
  [[ -z "${__VG_DATA}" ]] || {
    lvchange -a n "${__VG_DATA}"
    cryptsetup luksClose "/dev/mapper/${__DATA_CRYPT_NAME}"
  };
 
  cryptsetup luksClose "/dev/mapper/${__SYSTEM_CRYPT_NAME}"
} #; function umount_chroot ( )
 
case "${1}" in
  mount)
    echo "Mounting chroot"
    mount_chroot "${@}"
  ;;
  umount)
    echo "Unmounting chroot"
    umount_chroot "${@}"
  ;;
  *)
    echo "Use either mount or umount as parameters. Additionally a second parameter with any value can be passed to mount system partitions as well (/dev /dev/pts etc)"
  ;;
esac
```

To be able to copy the script right into the command line, you can use the following approach:

```plaintext
cat <<-'#EOF' > mount.sh
#!/bin/bash
# name of the volume group for the system
declare -r __VG_SYSTEM="vg_system"
# name of the volume group for the data partition - leave empty if you don't have it
declare -r __VG_DATA="vg_data"
# mount point where the chroot environment will be mounted
declare -r __DESTINATION_PARENT="/mnt"
# name of the logical volume, which contains the root ( / ) partition
declare -r __ROOT_LV_NAME="root"
# device where the /boot partition is stored on
declare -r __BOOT_DEVICE="/dev/sda2"
# device where the system LUKS partition is stored on
declare -r __SYSTEM_CRYPT_DEVICE="/dev/sda3"
# name of the LUKS partition after unlocking it (/dev/mapper/<NAME>)
declare -r __SYSTEM_CRYPT_NAME="crypted_system"
# device where the data LUKS partition is stored on - leave empty if you don't have it
declare -r __DATA_CRYPT_DEVICE="/dev/sdb"
# name of the LUKS partition after unlocking it (/dev/mapper/<NAME>) - leave empty if you don't have it
declare -r __DATA_CRYPT_NAME="crypted_data"

# function to mount all partitions
# if the script was called with a second parameter also all
# necessary system partitions (/dev /dev/pts etc) are mounted as well
function mount_chroot () {
  echo "Trying to decrypt system crypt device '${__SYSTEM_CRYPT_DEVICE}'"
  cryptsetup luksOpen "${__SYSTEM_CRYPT_DEVICE}" "${__SYSTEM_CRYPT_NAME}" || {
    echo "Decrypting '${__SYSTEM_CRYPT_DEVICE}' failed!";
    exit 1;
  };
  echo "Successful!"
 
  ( [[ -z "${__DATA_CRYPT_DEVICE}" ]] &&
    [[ -z "${__DATA_CRYPT_NAME}" ]]
  ) || {
    echo "Trying to decrypt data crypt device '${__DATA_CRYPT_DEVICE}'"
    cryptsetup luksOpen "${__DATA_CRYPT_DEVICE}" "${__DATA_CRYPT_NAME}" || {
      echo "Decrypting '${__DATA_CRYPT_DEVICE}' failed!";
      exit 1;
    };
    echo "Successful!"
  };
 
  # sleep to prevent that the VGs cant be detected yet
  sleep 2
  # detect vgs and switch to them
  vgchange -aay || {
    echo "Searching and activating volume groups failed!";
    exit 1;
  };
 
  # check whether the given root lv exist
  [[ -e "/dev/mapper/${__VG_SYSTEM}-${__ROOT_LV_NAME}" ]] || {
    echo "root LV '/dev/mapper/${__VG_SYSTEM}-${__ROOT_LV_NAME}' does not exist!";
    exit 1;
  };
 
  # mount the root lv and check whether it has been mounted successfully
  mount "/dev/mapper/${__VG_SYSTEM}-${__ROOT_LV_NAME}" "${__DESTINATION_PARENT}"
  mountpoint -q "${__DESTINATION_PARENT}" || {
    echo "Destination root '${__DESTINATION_PARENT}' is not mounted!"
    exit 1;
  };
 
  ( [[ -e ${__DESTINATION_PARENT}/boot ]] && 
    [[ -d ${__DESTINATION_PARENT}/boot ]]
  ) || {
  # create /boot within __DESTINATION_PARENT if it does not exist
    mkdir "${__DESTINATION_PARENT}/boot"
  };
  # try mounting /boot
  mount "${__BOOT_DEVICE}" "${__DESTINATION_PARENT}/boot"
 
  # go through all system LVs and mount them, unless its the root or swap partition
  for lv in "/dev/mapper/${__VG_SYSTEM}-"*; do
    declare part="$(echo "$(basename "${lv}")" | sed -e 's/vg_.*-//' -e 's/_/\//g')"
    case "${part}" in
      root)
        # we mounted it already
        continue
      ;;
      swap)
        # set swap to the current LV
        swapon "${lv}"
      ;;
      *)
        # create the necessary folders for the current LV
        # if they don't exist yet
        ( [[ -e "${__DESTINATION_PARENT}/${part}" ]] &&
          [[ -d "${__DESTINATION_PARENT}/${part}" ]]
        ) || {
          mkdir -p "${__DESTINATION_PARENT}/${part}"
        };
        # try mounting the LV
        mount "${lv}" "${__DESTINATION_PARENT}/${part}"
      ;;
    esac
  done
 
  [[ -z "${__VG_DATA}" ]] || {
    echo "Data partition defined :3"
    # do the same as for the system LVs for the data LVs
    for lv in "/dev/mapper/${__VG_DATA}-"*; do
      declare part="$(echo "$(basename "${lv}")" | sed -e 's/vg_.*-//' -e 's/_/\//g')"
      ( [[ -e "${__DESTINATION_PARENT}/${part}" ]] &&
        [[ -e "${__DESTINATION_PARENT}/${part}" ]]
      ) || {
        mkdir -p "${__DESTINATION_PARENT}/${part}"
      };
      mount "${lv}" "${__DESTINATION_PARENT}/${part}"
    done
  };
 
 
  [[ -n "${2}" ]] || {
    exit 0;
  };
 
  echo "Mounting necessary system partitions to chroot"
  mount -o bind /dev "${__DESTINATION_PARENT}/dev"
  mount -o bind /run "${__DESTINATION_PARENT}/run"
  mount -t devpts devpts "${__DESTINATION_PARENT}/dev/pts"
  mount -t sysfs sys "${__DESTINATION_PARENT}/sys"
  mount -t proc proc "${__DESTINATION_PARENT}/proc"
  exit 0;
} #; function mount_chroot ( )
 
# function to unmount all partitions defined under __DESTINATION_PARENT
# and "close" the VGs and finally close the LUKS partitions
function umount_chroot () {
  umount -lf "${__DESTINATION_PARENT}"
  for lv in "/dev/mapper/${__VG_SYSTEM}-"*; do
    declare part="$(echo "$(basename "${lv}")" | sed -e 's/vg_.*-//' -e 's/_/\//g')"
    case "${part}" in
      swap)
        swapoff "${lv}"
      ;;
      *)
        continue
      ;;
    esac
  done
  lvchange -a n "${__VG_SYSTEM}"
 
  [[ -z "${__VG_DATA}" ]] || {
    lvchange -a n "${__VG_DATA}"
    cryptsetup luksClose "/dev/mapper/${__DATA_CRYPT_NAME}"
  };
 
  cryptsetup luksClose "/dev/mapper/${__SYSTEM_CRYPT_NAME}"
} #; function umount_chroot ( )
 
case "${1}" in
  mount)
    echo "Mounting chroot"
    mount_chroot "${@}"
  ;;
  umount)
    echo "Unmounting chroot"
    umount_chroot "${@}"
  ;;
  *)
    echo "Use either mount or umount as parameters. Additionally a second parameter with any value can be passed to mount system partitions as well (/dev /dev/pts etc)"
  ;;
esac
#EOF
```

You should now have a file `mount.sh` in your current directory, with the contents of the script above (minus the first and last line).
Now, go ahead and adjust the necessary parts and execute it via either `bash mount.sh mount` or first setting executable permissions on it and then executing it (done
via `chmod +x mount.sh && ./mount.sh mount`).

The output will look similar to the following (depending again, whether you have an additional data drive or not):

```terminal
root@rescue ~ # bash mount.sh mount
Mounting chroot
Trying to decrypt system crypt device '/dev/sda3'
Enter passphrase for /dev/sda3: 
Successful!
  8 logical volume(s) in volume group "vg_system" now active
root@rescue ~ #
```

Finally you should check whether the script has been successfully run and the partitions are mounted to your needs (the last lines are showing the mounted partitions from the
script):

```terminal
root@rescue ~ # mount
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
sys on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
udev on /dev type devtmpfs (rw,relatime,size=32820460k,nr_inodes=8205115,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
213.133.99.101:/nfs on /root/.oldroot/nfs type nfs (ro,noatime,vers=3,rsize=8192,wsize=8192,namlen=255,acregmin=600,acregmax=600,acdirmin=600,acdirmax=600,hard,nocto,nolock,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=213.133.99.101,mountvers=3,mountproto=tcp,local_lock=all,addr=213.133.99.101)
overlay on / type overlay (rw,relatime,lowerdir=/nfsroot,upperdir=/ramfs/root,workdir=/ramfs/work)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,mode=755)
tmpfs on /run/lock type tmpfs (rw,nosuid,nodev,noexec,relatime,size=5120k)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=27,pgrp=1,timeout=0,minproto=5,maxproto=5,direct)
mqueue on /dev/mqueue type mqueue (rw,relatime)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,pagesize=2M)
debugfs on /sys/kernel/debug type debugfs (rw,relatime)
tracefs on /sys/kernel/debug/tracing type tracefs (rw,relatime)
tmpfs on /run/user/0 type tmpfs (rw,nosuid,nodev,relatime,size=6565472k,mode=700)
/dev/mapper/vg_system-root on /mnt type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/sda2 on /mnt/boot type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/mapper/vg_system-home on /mnt/home type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/mapper/vg_system-tmp on /mnt/tmp type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/mapper/vg_system-var on /mnt/var type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/mapper/vg_system-var_log on /mnt/var/log type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/mapper/vg_system-var_log_audit on /mnt/var/log/audit type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/mapper/vg_system-var_tmp on /mnt/var/tmp type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
root@rescue ~ #
```

As the last step, we need to set the proper permissions on the mounted tmp folder (`/mnt/tmp`):

```terminal
root@rescue ~ # chmod 1777 /mnt/tmp
root@rescue ~ #
```

### Starting the installation

Finally we can start the installation of Debian Bullseye within our live system.
For the installation we are going to use a program called [Debootstrap](https://wiki.debian.org/Debootstrap). Debootstrap is basically used to install a Debian system within a
Debian system (our live environment). The latest version can always be found [package repository of debian.org](http://ftp.debian.org/debian/pool/main/d/debootstrap/) - we
need the version, which is packaged using `.deb`.

### Downloading Debootstrap and modify it

First, we need to download the .deb using wget or curl:

```terminal
root@rescue ~ # cd /tmp/
root@rescue /tmp # wget http://ftp.debian.org/debian/pool/main/d/debootstrap/debootstrap_1.0.124_all.deb
--2021-08-22 12:42:35--  http://ftp.debian.org/debian/pool/main/d/debootstrap/debootstrap_1.0.124_all.deb
Resolving ftp.debian.org (ftp.debian.org)... 151.101.14.132, 2a04:4e42:3::644
Connecting to ftp.debian.org (ftp.debian.org)|151.101.14.132|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 76416 (75K) [application/x-debian-package]
Saving to: ‘debootstrap_1.0.124_all.deb’

debootstrap_1.0.124_all.deb                100%[=======================================================================================>]  74.62K  --.-KB/s    in 0.03s   

2021-08-22 12:42:35 (2.30 MB/s) - ‘debootstrap_1.0.124_all.deb’ saved [76416/76416]

root@rescue /tmp #
```

After we downloaded it, we need to unpack it:

```terminal
root@rescue /tmp # ar x debootstrap_1.0.124_all.deb 
root@rescue /tmp # tar xfz control.tar.gz 
root@rescue /tmp # tar xfz data.tar.gz 
root@rescue /tmp #
```

Next, we want to make sure, that the contents are not broken or modified in any way.

The following commands create a MD5 checksums from the local files and compare it to the MD5 checksums shipped with the `.deb` file.
If there is no output at all, the files do not differ - if there is output, the files differ. If the files differ, re-download it (it shouldn't happen at all - **never!**).
Although it could be a bug from the Debian team, while creating the package (very, very unlikely!).
If the local MD5 checksums still differ from the shipped MD5 checksums, consider re-downloading an older version (doesn't really matter) and file a bug report over at Debian.

```terminal
root@rescue /tmp # cat md5sums | cut -d " " -f 3 | xargs md5sum $1 > md5sums.local; diff md5sums md5sums.local
root@rescue /tmp #
```

Next, we want to search in the file `usr/sbin/debootstrap` for the line `DEBOOTSTRAP_DIR=/usr/share/debootstrap` and prefix `/tmp` in front of the path.
This is necessary in order to run the script from `/tmp` (where we are in the moment).

You can either do it by hand or run following command:

```plaintext
sed 's@\/usr\/share\/debootstrap@/tmp/usr/share/debootstrap@' -i usr/sbin/debootstrap
```

.. and check afterwards if the change has been done successfully using the following command:

```plaintext
grep [[:space:]]DEBOOTSTRAP_DIR= usr/sbin/debootstrap
```

The output should be similar to following:

```terminal
root@rescue /tmp # sed 's@\/usr\/share\/debootstrap@/tmp/usr/share/debootstrap@' -i usr/sbin/debootstrap
root@rescue /tmp # grep [[:space:]]DEBOOTSTRAP_DIR= usr/sbin/debootstrap
                DEBOOTSTRAP_DIR=/debootstrap
                DEBOOTSTRAP_DIR=/tmp/usr/share/debootstrap
root@rescue /tmp #
```

## Installation of the system

Finally we can start the installation using debootstrap with the following command (replace the values you want to change):

```plaintext
usr/sbin/debootstrap --arch amd64 bullseye /mnt/ http://ftp2.de.debian.org/debian | tee /mnt/install.log
```

The installation will take a couple of minutes/seconds (depending on the performance of your system) and the output will look similar to this (truncated):

```terminal
root@rescue /tmp # usr/sbin/debootstrap --arch amd64 buster /mnt/ http://ftp2.de.debian.org/debian | tee /mnt/install.log
I: Retrieving InRelease 
I: Retrieving Release 
I: Retrieving Release.gpg 
I: Checking Release signature
I: Valid Release signature (key id 067E3C456BAE240ACEE88F6FEF0F382A1A7B6500)
I: Retrieving Packages 
I: Validating Packages 
I: Resolving dependencies of required packages...
I: Resolving dependencies of base packages...
I: Checking component main on http://ftp2.de.debian.org/debian...
[..]
I: Validating kmod 26-1
I: Retrieving libkmod2 26-1
I: Validating libkmod2 26-1
I: Retrieving less 487-0.1+b1
I: Validating less 487-0.1+b1
I: Retrieving libbsd0 0.9.1-2
I: Validating libbsd0 0.9.1-2
I: Retrieving libcap-ng0 0.7.9-2
I: Validating libcap-ng0 0.7.9-2
I: Retrieving libcap2 1:2.25-2
I: Validating libcap2 1:2.25-2
[..]
I: Chosen extractor for .deb packages: dpkg-deb
I: Extracting libacl1...
I: Extracting adduser...
I: Extracting apt...
I: Extracting libapt-pkg5.0...
I: Extracting libattr1...
[..]
I: Installing core packages...
I: Unpacking required packages...
I: Unpacking libacl1:amd64...
I: Unpacking adduser...
I: Unpacking apt...
I: Unpacking libapt-pkg5.0:amd64...
[..]
I: Configuring required packages...
I: Configuring debian-archive-keyring...
I: Configuring libaudit-common...
I: Configuring libsemanage-common...
I: Configuring ncurses-base...
[..]
I: Configuring systemd...
I: Base system installed successfully.
root@rescue /tmp # 
```

After the installation has been sucessfully finished, we need to mount the necessary system partitions, in order to configure the system, using the following commands:

```terminal
root@rescue /tmp # mount -o bind /dev/ /mnt/dev/
root@rescue /tmp # mount -t devpts devpts /mnt/dev/pts
root@rescue /tmp # mount -t proc proc /mnt/proc/
root@rescue /tmp # mount -t sysfs sys /mnt/sys/
root@rescue /tmp # mount -o bind /run /mnt/run
root@rescue /tmp #
```

.. or “copy-paste-friendlier”:

```plaintext
mount -o bind /dev/ /mnt/dev/; mount -t devpts devpts /mnt/dev/pts; mount -t proc proc /mnt/proc/; mount -t sysfs sys /mnt/sys/; mount -o bind /run /mnt/run
```

Finally we have the system installed and ready to configure - we still have a couple of things to do, in order to make the system work as we want it to.

## Base configuration

First we want to chroot to the environment:

```terminal
root@rescue /tmp # XTERM=xterm-color LANG=C.UTF-8 chroot /mnt /bin/bash
root@rescue:/# pwd
/
root@rescue:/#
```

We need to set the proper permissions for `/tmp`:

```terminal
root@rescue:/# chmod 1777 /tmp/
```

Next, we want to change the root password, as it is currently not set (use a **complex** and **unique** password!):

```terminal
root@rescue:/# passwd
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
root@rescue:/#
```

Then we want to set both the hostname (`/etc/hostname`) and mailname (`/etc/mailname` - this will come in handy later, when we install `postfix`) and as well add ourselves to
`/etc/hosts`:

```terminal
root@rescue:/# echo "pven.scheib.me" > /etc/hostname
root@rescue:/# echo "pven.scheib.me" > /etc/mailname
root@rescue:/# echo "$(ip addr show eth0 | grep inet[[:space:]] | awk '{print $2}' | sed -E 's@\/[[:digit:]]+$@@') $(cat /etc/hostname) $(cat /etc/hostname | sed 's@\.@ @g' | awk '{ print $1 }')" >> /etc/hosts
root@rescue:/#
```

As next step we want to configure our network adapter(s) within `/etc/network/interfaces` - mine looks like the following:

```terminal
root@rescue:/# cat /etc/network/interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

auto            lo
iface           lo inet loopback

auto            eth0
iface           eth0 inet static
address         159.69.68.69
netmask         255.255.255.192
gateway         159.69.68.65
broadcast       159.69.68.127
pre-up          /sbin/ip addr flush dev eth0 || true
root@rescue:/#
```

Please note, I modified the IP addresses and they do not reflect an actual system. Please adjust the file accordingly.

The `pre-up` command `/sbin/ip addr flush dev eth0 || true` has to be used, as we have an IP address already *before* the final system is up and running. This is due to the
`Dropbear instance` that is running which allows as to unlock the LUKS partition(s) - more on that later.

Of course we want to set our nameservers correctly - in this case I am using the nameservers from
[Cloudflare](https://www.cloudflare.com/de-de/learning/dns/what-is-1.1.1.1/) and [Google](https://developers.google.com/speed/public-dns):

```terminal
root@rescue:/# cat /etc/resolv.conf 
nameserver 1.1.1.1
nameserver 1.0.0.1
nameserver 8.8.8.8
root@rescue:/#
```

Also we want to set the sources for aptitude correctly (change it to your needs):

```terminal
cat > /etc/apt/sources.list << "#EOF"
deb http://ftp.de.debian.org/debian/ bullseye main non-free contrib
deb-src http://ftp.de.debian.org/debian/ bullseye main non-free contrib
deb http://security.debian.org/ bullseye-security main non-free contrib
deb-src http://security.debian.org/ bullseye-security main non-free contrib
deb http://ftp.de.debian.org/debian/ bullseye-updates main non-free contrib
deb-src http://ftp.de.debian.org/debian/ bullseye-updates main non-free contrib
#EOF
```

.. which will result in the following file:

```terminal
root@rescue:/# cat /etc/apt/sources.list
deb http://ftp.de.debian.org/debian/ buster main non-free contrib
deb-src http://ftp.de.debian.org/debian/ buster main non-free contrib
deb http://security.debian.org/ buster/updates main non-free contrib
deb-src http://security.debian.org/ buster/updates main non-free contrib
deb http://ftp.de.debian.org/debian/ buster-updates main non-free contrib
deb-src http://ftp.de.debian.org/debian/ buster-updates main non-free contrib
root@rescue:/#
```

Let’s update the cache .. :slightly_smiling_face:

```terminal
root@rescue:/# apt-get update
Fetched 22.9 MB in 3s (8565 kB/s)                              
[..]
Reading package lists... Done
Building dependency tree... Done
3 packages can be upgraded. Run 'apt list --upgradable' to see them.
root@rescue:/#
```

Finally we want to install and configure the `locales` (I use `en_US.UTF-8` as the system language), using the following command:

```plaintext
apt-get install -y locales && dpkg-reconfigure locales
```

<!-- markdownlint-disable MD033 -->
<details>
<summary>Sample output</summary>

```terminal
root@rescue:/# apt-get install -y locales && dpkg-reconfigure locales
Reading package lists... Done
Building dependency tree... Done
The following additional packages will be installed:
  libc-l10n
The following NEW packages will be installed:
  libc-l10n locales
0 upgraded, 2 newly installed, 0 to remove and 3 not upgraded.
Need to get 4907 kB of archives.
After this operation, 20.8 MB of additional disk space will be used.
Get:1 http://ftp.de.debian.org/debian bullseye/main amd64 libc-l10n all 2.28-10 [847 kB]
Get:2 http://ftp.de.debian.org/debian bullseye/main amd64 locales all 2.28-10 [4060 kB]
Fetched 4907 kB in 0s (20.0 MB/s)
Preconfiguring packages ...
Selecting previously unselected package libc-l10n.
(Reading database ... 9230 files and directories currently installed.)
Preparing to unpack .../libc-l10n_2.28-10_all.deb ...
Unpacking libc-l10n (2.28-10) ...
Selecting previously unselected package locales.
Preparing to unpack .../locales_2.28-10_all.deb ...
Unpacking locales (2.28-10) ...
Setting up libc-l10n (2.28-10) ...
Setting up locales (2.28-10) ...
Generating locales (this might take a while)...
Generation complete.
Generating locales (this might take a while)...
  en_US.UTF-8... done
Generation complete.
root@rescue:/#
```

</details>
<!-- markdownlint-disable MD033 -->

Additionally we need to set a few more locale settings in `/etc/environment`, which are not set by default, but are causing warning messages when not set, while installing
packages using `aptitude`:

```terminal
cat > /etc/environment << "EOF"
export LANGUAGE=en_US.UTF-8
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
EOF
```

<!-- markdownlint-disable MD033 -->
<details>
<summary>Sample output:</summary>

```terminal
root@rescue:/# cat > /etc/environment << "EOF"
> export LANGUAGE=en_US.UTF-8
> export LC_ALL=en_US.UTF-8
> export LANG=en_US.UTF-8
> EOF
root@rescue:/# cat /etc/environment 
export LANGUAGE=en_US.UTF-8
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
root@rescue:/# 
```

</details>
<!-- markdownlint-enable MD033 -->

Next, we want to set the correct timezone using `tzdata` and the following command:

```plaintext
dpkg-reconfigure tzdata
```

<!-- markdownlint-disable MD033 -->
<details>
<summary>Sample output:</summary>

```shell
root@rescue:/# dpkg-reconfigure tzdata

Current default time zone: 'Europe/Berlin'
Local time is now:      Sat Sep  5 20:52:58 CEST 2020.
Universal Time is now:  Sat Sep  5 18:52:58 UTC 2020.

root@rescue:/#
```

</details>
<!-- markdownlint-enable MD033 -->

Last but not least, we want to install `cryptsetup` and the linux image itself for AMD64 architectures (`linux-image-amd64`; For i386 (32bit) systems you need to install the
package for i386).
During the installation also the keyboard is configured - I set mine to German.

The following command will be used to install the needed packages:

```plaintext
apt-get install -y linux-image-amd64 cryptsetup
```

<!-- markdownlint-disable MD033 -->
<details>
<summary>Sample output:</summary>

```terminal
root@rescue:/# apt-get install -y linux-image-amd64 cryptsetup
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following additional packages will be installed:
  apparmor busybox bzip2 console-setup console-setup-linux cryptsetup-bin cryptsetup-initramfs cryptsetup-run file firmware-linux-free initramfs-tools initramfs-tools-core kbd
  keyboard-configuration klibc-utils libexpat1 libklibc libmagic-mgc libmagic1 libmpdec2 libpython3-stdlib libpython3.7-minimal libpython3.7-stdlib libreadline7 libsqlite3-0
  linux-base linux-image-4.19.0-10-amd64 mime-support pigz python3 python3-minimal python3.7 python3.7-minimal xkb-data xz-utils
Suggested packages:
  apparmor-profiles-extra apparmor-utils bzip2-doc dosfstools keyutils bash-completion linux-doc-4.19 debian-kernel-handbook grub-pc | grub-efi-amd64 | extlinux python3-doc
  python3-tk python3-venv python3.7-venv python3.7-doc binutils binfmt-support
The following NEW packages will be installed:
  apparmor busybox bzip2 console-setup console-setup-linux cryptsetup cryptsetup-bin cryptsetup-initramfs cryptsetup-run file firmware-linux-free initramfs-tools
  initramfs-tools-core kbd keyboard-configuration klibc-utils libexpat1 libklibc libmagic-mgc libmagic1 libmpdec2 libpython3-stdlib libpython3.7-minimal libpython3.7-stdlib
  libreadline7 libsqlite3-0 linux-base linux-image-4.19.0-10-amd64 linux-image-amd64 mime-support pigz python3 python3-minimal python3.7 python3.7-minimal xkb-data xz-utils
0 upgraded, 37 newly installed, 0 to remove and 3 not upgraded.
Need to get 59.4 MB of archives.
After this operation, 320 MB of additional disk space will be used.
Get:1 http://ftp.de.debian.org/debian bullseye/main amd64 kbd amd64 2.0.4-4 [337 kB]
Get:2 http://ftp.de.debian.org/debian bullseye/main amd64 keyboard-configuration all 1.193~deb10u1 [404 kB]
Get:3 http://ftp.de.debian.org/debian bullseye/main amd64 console-setup-linux all 1.193~deb10u1 [1494 kB]
Get:4 http://ftp.de.debian.org/debian bullseye/main amd64 xkb-data all 2.26-2 [681 kB]
Get:5 http://ftp.de.debian.org/debian bullseye/main amd64 console-setup all 1.193~deb10u1 [98.5 kB]
Get:6 http://ftp.de.debian.org/debian bullseye/main amd64 pigz amd64 2.4-1 [57.8 kB]
Get:7 http://ftp.de.debian.org/debian bullseye/main amd64 libpython3.7-minimal amd64 3.7.3-2+deb10u2 [589 kB]
Get:8 http://ftp.de.debian.org/debian bullseye/main amd64 libexpat1 amd64 2.2.6-2+deb10u1 [106 kB]
Get:9 http://ftp.de.debian.org/debian bullseye/main amd64 python3.7-minimal amd64 3.7.3-2+deb10u2 [1731 kB]
Get:10 http://ftp.de.debian.org/debian bullseye/main amd64 python3-minimal amd64 3.7.3-1 [36.6 kB]
Get:11 http://ftp.de.debian.org/debian bullseye/main amd64 mime-support all 3.62 [37.2 kB]
Get:12 http://ftp.de.debian.org/debian bullseye/main amd64 libmpdec2 amd64 2.4.2-2 [87.2 kB]
Get:13 http://ftp.de.debian.org/debian bullseye/main amd64 libreadline7 amd64 7.0-5 [151 kB]
Get:14 http://ftp.de.debian.org/debian bullseye/main amd64 libsqlite3-0 amd64 3.27.2-3 [641 kB]
Get:15 http://ftp.de.debian.org/debian bullseye/main amd64 libpython3.7-stdlib amd64 3.7.3-2+deb10u2 [1732 kB]
Get:16 http://ftp.de.debian.org/debian bullseye/main amd64 python3.7 amd64 3.7.3-2+deb10u2 [330 kB]
Get:17 http://ftp.de.debian.org/debian bullseye/main amd64 libpython3-stdlib amd64 3.7.3-1 [20.0 kB]
Get:18 http://ftp.de.debian.org/debian bullseye/main amd64 python3 amd64 3.7.3-1 [61.5 kB]
Get:19 http://ftp.de.debian.org/debian bullseye/main amd64 bzip2 amd64 1.0.6-9.2~deb10u1 [48.4 kB]
Get:20 http://ftp.de.debian.org/debian bullseye/main amd64 libmagic-mgc amd64 1:5.35-4+deb10u1 [242 kB]
Get:21 http://ftp.de.debian.org/debian bullseye/main amd64 libmagic1 amd64 1:5.35-4+deb10u1 [117 kB]
Get:22 http://ftp.de.debian.org/debian bullseye/main amd64 file amd64 1:5.35-4+deb10u1 [66.4 kB]
Get:23 http://ftp.de.debian.org/debian bullseye/main amd64 xz-utils amd64 5.2.4-1 [183 kB]
Get:24 http://ftp.de.debian.org/debian bullseye/main amd64 apparmor amd64 2.13.2-10 [537 kB]
Get:25 http://ftp.de.debian.org/debian bullseye/main amd64 busybox amd64 1:1.30.1-4 [432 kB]
Get:26 http://ftp.de.debian.org/debian bullseye/main amd64 cryptsetup-bin amd64 2:2.1.0-5+deb10u2 [303 kB]
Get:27 http://ftp.de.debian.org/debian bullseye/main amd64 cryptsetup-run amd64 2:2.1.0-5+deb10u2 [198 kB]
Get:28 http://ftp.de.debian.org/debian bullseye/main amd64 libklibc amd64 2.0.6-1 [54.2 kB]
Get:29 http://ftp.de.debian.org/debian bullseye/main amd64 klibc-utils amd64 2.0.6-1 [95.0 kB]
Get:30 http://ftp.de.debian.org/debian bullseye/main amd64 initramfs-tools-core all 0.133+deb10u1 [99.2 kB]
Get:31 http://ftp.de.debian.org/debian bullseye/main amd64 linux-base all 4.6 [32.4 kB]
Get:32 http://ftp.de.debian.org/debian bullseye/main amd64 initramfs-tools all 0.133+deb10u1 [69.7 kB]
Get:33 http://ftp.de.debian.org/debian bullseye/main amd64 cryptsetup-initramfs all 2:2.1.0-5+deb10u2 [69.6 kB]
Get:34 http://ftp.de.debian.org/debian bullseye/main amd64 cryptsetup all 2:2.1.0-5+deb10u2 [51.5 kB]
Get:35 http://ftp.de.debian.org/debian bullseye/main amd64 firmware-linux-free all 3.4 [19.2 kB]
Get:36 http://ftp.de.debian.org/debian bullseye/main amd64 linux-image-4.19.0-10-amd64 amd64 4.19.132-1 [48.2 MB]
Get:37 http://ftp.de.debian.org/debian bullseye/main amd64 linux-image-amd64 amd64 4.19+105+deb10u5 [8168 B]
Fetched 59.4 MB in 1s (77.1 MB/s)
Extracting templates from packages: 100%
Preconfiguring packages ...
Selecting previously unselected package kbd.
(Reading database ... 9883 files and directories currently installed.)
Preparing to unpack .../0-kbd_2.0.4-4_amd64.deb ...
Unpacking kbd (2.0.4-4) ...
Selecting previously unselected package keyboard-configuration.
Preparing to unpack .../1-keyboard-configuration_1.193~deb10u1_all.deb ...
Unpacking keyboard-configuration (1.193~deb10u1) ...
Selecting previously unselected package console-setup-linux.
Preparing to unpack .../2-console-setup-linux_1.193~deb10u1_all.deb ...
Unpacking console-setup-linux (1.193~deb10u1) ...
Selecting previously unselected package xkb-data.
Preparing to unpack .../3-xkb-data_2.26-2_all.deb ...
Unpacking xkb-data (2.26-2) ...
Selecting previously unselected package console-setup.
Preparing to unpack .../4-console-setup_1.193~deb10u1_all.deb ...
Unpacking console-setup (1.193~deb10u1) ...
Selecting previously unselected package pigz.
Preparing to unpack .../5-pigz_2.4-1_amd64.deb ...
Unpacking pigz (2.4-1) ...
Selecting previously unselected package libpython3.7-minimal:amd64.
Preparing to unpack .../6-libpython3.7-minimal_3.7.3-2+deb10u2_amd64.deb ...
Unpacking libpython3.7-minimal:amd64 (3.7.3-2+deb10u2) ...
Selecting previously unselected package libexpat1:amd64.
Preparing to unpack .../7-libexpat1_2.2.6-2+deb10u1_amd64.deb ...
Unpacking libexpat1:amd64 (2.2.6-2+deb10u1) ...
Selecting previously unselected package python3.7-minimal.
Preparing to unpack .../8-python3.7-minimal_3.7.3-2+deb10u2_amd64.deb ...
Unpacking python3.7-minimal (3.7.3-2+deb10u2) ...
Setting up libpython3.7-minimal:amd64 (3.7.3-2+deb10u2) ...
Setting up libexpat1:amd64 (2.2.6-2+deb10u1) ...
Setting up python3.7-minimal (3.7.3-2+deb10u2) ...
Selecting previously unselected package python3-minimal.
(Reading database ... 11105 files and directories currently installed.)
Preparing to unpack .../0-python3-minimal_3.7.3-1_amd64.deb ...
Unpacking python3-minimal (3.7.3-1) ...
Selecting previously unselected package mime-support.
Preparing to unpack .../1-mime-support_3.62_all.deb ...
Unpacking mime-support (3.62) ...
Selecting previously unselected package libmpdec2:amd64.
Preparing to unpack .../2-libmpdec2_2.4.2-2_amd64.deb ...
Unpacking libmpdec2:amd64 (2.4.2-2) ...
Selecting previously unselected package libreadline7:amd64.
Preparing to unpack .../3-libreadline7_7.0-5_amd64.deb ...
Unpacking libreadline7:amd64 (7.0-5) ...
Selecting previously unselected package libsqlite3-0:amd64.
Preparing to unpack .../4-libsqlite3-0_3.27.2-3_amd64.deb ...
Unpacking libsqlite3-0:amd64 (3.27.2-3) ...
Selecting previously unselected package libpython3.7-stdlib:amd64.
Preparing to unpack .../5-libpython3.7-stdlib_3.7.3-2+deb10u2_amd64.deb ...
Unpacking libpython3.7-stdlib:amd64 (3.7.3-2+deb10u2) ...
Selecting previously unselected package python3.7.
Preparing to unpack .../6-python3.7_3.7.3-2+deb10u2_amd64.deb ...
Unpacking python3.7 (3.7.3-2+deb10u2) ...
Selecting previously unselected package libpython3-stdlib:amd64.
Preparing to unpack .../7-libpython3-stdlib_3.7.3-1_amd64.deb ...
Unpacking libpython3-stdlib:amd64 (3.7.3-1) ...
Setting up python3-minimal (3.7.3-1) ...
Selecting previously unselected package python3.
(Reading database ... 11562 files and directories currently installed.)
Preparing to unpack .../00-python3_3.7.3-1_amd64.deb ...
Unpacking python3 (3.7.3-1) ...
Selecting previously unselected package bzip2.
Preparing to unpack .../01-bzip2_1.0.6-9.2~deb10u1_amd64.deb ...
Unpacking bzip2 (1.0.6-9.2~deb10u1) ...
Selecting previously unselected package libmagic-mgc.
Preparing to unpack .../02-libmagic-mgc_1%3a5.35-4+deb10u1_amd64.deb ...
Unpacking libmagic-mgc (1:5.35-4+deb10u1) ...
Selecting previously unselected package libmagic1:amd64.
Preparing to unpack .../03-libmagic1_1%3a5.35-4+deb10u1_amd64.deb ...
Unpacking libmagic1:amd64 (1:5.35-4+deb10u1) ...
Selecting previously unselected package file.
Preparing to unpack .../04-file_1%3a5.35-4+deb10u1_amd64.deb ...
Unpacking file (1:5.35-4+deb10u1) ...
Selecting previously unselected package xz-utils.
Preparing to unpack .../05-xz-utils_5.2.4-1_amd64.deb ...
Unpacking xz-utils (5.2.4-1) ...
Selecting previously unselected package apparmor.
Preparing to unpack .../06-apparmor_2.13.2-10_amd64.deb ...
Unpacking apparmor (2.13.2-10) ...
Selecting previously unselected package busybox.
Preparing to unpack .../07-busybox_1%3a1.30.1-4_amd64.deb ...
Unpacking busybox (1:1.30.1-4) ...
Selecting previously unselected package cryptsetup-bin.
Preparing to unpack .../08-cryptsetup-bin_2%3a2.1.0-5+deb10u2_amd64.deb ...
Unpacking cryptsetup-bin (2:2.1.0-5+deb10u2) ...
Selecting previously unselected package cryptsetup-run.
Preparing to unpack .../09-cryptsetup-run_2%3a2.1.0-5+deb10u2_amd64.deb ...
Unpacking cryptsetup-run (2:2.1.0-5+deb10u2) ...
Selecting previously unselected package libklibc:amd64.
Preparing to unpack .../10-libklibc_2.0.6-1_amd64.deb ...
Unpacking libklibc:amd64 (2.0.6-1) ...
Selecting previously unselected package klibc-utils.
Preparing to unpack .../11-klibc-utils_2.0.6-1_amd64.deb ...
Unpacking klibc-utils (2.0.6-1) ...
Selecting previously unselected package initramfs-tools-core.
Preparing to unpack .../12-initramfs-tools-core_0.133+deb10u1_all.deb ...
Unpacking initramfs-tools-core (0.133+deb10u1) ...
Selecting previously unselected package linux-base.
Preparing to unpack .../13-linux-base_4.6_all.deb ...
Unpacking linux-base (4.6) ...
Selecting previously unselected package initramfs-tools.
Preparing to unpack .../14-initramfs-tools_0.133+deb10u1_all.deb ...
Unpacking initramfs-tools (0.133+deb10u1) ...
Selecting previously unselected package cryptsetup-initramfs.
Preparing to unpack .../15-cryptsetup-initramfs_2%3a2.1.0-5+deb10u2_all.deb ...
Unpacking cryptsetup-initramfs (2:2.1.0-5+deb10u2) ...
Selecting previously unselected package cryptsetup.
Preparing to unpack .../16-cryptsetup_2%3a2.1.0-5+deb10u2_all.deb ...
Unpacking cryptsetup (2:2.1.0-5+deb10u2) ...
Selecting previously unselected package firmware-linux-free.
Preparing to unpack .../17-firmware-linux-free_3.4_all.deb ...
Unpacking firmware-linux-free (3.4) ...
Selecting previously unselected package linux-image-4.19.0-10-amd64.
Preparing to unpack .../18-linux-image-4.19.0-10-amd64_4.19.132-1_amd64.deb ...
Unpacking linux-image-4.19.0-10-amd64 (4.19.132-1) ...
Selecting previously unselected package linux-image-amd64.
Preparing to unpack .../19-linux-image-amd64_4.19+105+deb10u5_amd64.deb ...
Unpacking linux-image-amd64 (4.19+105+deb10u5) ...
Setting up cryptsetup-bin (2:2.1.0-5+deb10u2) ...
Setting up linux-base (4.6) ...
Setting up mime-support (3.62) ...
Setting up firmware-linux-free (3.4) ...
Setting up libmagic-mgc (1:5.35-4+deb10u1) ...
Setting up libsqlite3-0:amd64 (3.27.2-3) ...
Setting up libmagic1:amd64 (1:5.35-4+deb10u1) ...
Setting up xkb-data (2.26-2) ...
Setting up file (1:5.35-4+deb10u1) ...
Setting up bzip2 (1.0.6-9.2~deb10u1) ...
Setting up busybox (1:1.30.1-4) ...
Setting up libklibc:amd64 (2.0.6-1) ...
Setting up cryptsetup-run (2:2.1.0-5+deb10u2) ...
Setting up xz-utils (5.2.4-1) ...
update-alternatives: using /usr/bin/xz to provide /usr/bin/lzma (lzma) in auto mode
Setting up pigz (2.4-1) ...
Setting up kbd (2.0.4-4) ...
Setting up libmpdec2:amd64 (2.4.2-2) ...
Setting up keyboard-configuration (1.193~deb10u1) ...
Setting up klibc-utils (2.0.6-1) ...
No diversion 'diversion of /usr/share/initramfs-tools/hooks/klibc to /usr/share/initramfs-tools/hooks/klibc^i-t by klibc-utils', none removed.
Setting up libreadline7:amd64 (7.0-5) ...
Setting up libpython3.7-stdlib:amd64 (3.7.3-2+deb10u2) ...
Setting up console-setup-linux (1.193~deb10u1) ...
Created symlink /etc/systemd/system/sysinit.target.wants/keyboard-setup.service → /lib/systemd/system/keyboard-setup.service.
Created symlink /etc/systemd/system/multi-user.target.wants/console-setup.service → /lib/systemd/system/console-setup.service.
Setting up initramfs-tools-core (0.133+deb10u1) ...
Setting up libpython3-stdlib:amd64 (3.7.3-1) ...
Setting up python3.7 (3.7.3-2+deb10u2) ...
Setting up console-setup (1.193~deb10u1) ...
Setting up initramfs-tools (0.133+deb10u1) ...
update-initramfs: deferring update (trigger activated)
Setting up python3 (3.7.3-1) ...
running python rtupdate hooks for python3.7...
running python post-rtupdate hooks for python3.7...
Setting up apparmor (2.13.2-10) ...
Created symlink /etc/systemd/system/sysinit.target.wants/apparmor.service → /lib/systemd/system/apparmor.service.
Setting up cryptsetup-initramfs (2:2.1.0-5+deb10u2) ...
update-initramfs: deferring update (trigger activated)
Setting up linux-image-4.19.0-10-amd64 (4.19.132-1) ...
I: /vmlinuz.old is now a symlink to boot/vmlinuz-4.19.0-10-amd64
I: /initrd.img.old is now a symlink to boot/initrd.img-4.19.0-10-amd64
I: /vmlinuz is now a symlink to boot/vmlinuz-4.19.0-10-amd64
I: /initrd.img is now a symlink to boot/initrd.img-4.19.0-10-amd64
/etc/kernel/postinst.d/initramfs-tools:
update-initramfs: Generating /boot/initrd.img-4.19.0-10-amd64
cryptsetup: WARNING: target 'crypted_system' not found in /etc/crypttab
cryptsetup: WARNING: The initramfs image may not contain cryptsetup binaries
    nor crypto modules. If that's on purpose, you may want to uninstall the
    'cryptsetup-initramfs' package in order to disable the cryptsetup initramfs
    integration and avoid this warning.
Setting up cryptsetup (2:2.1.0-5+deb10u2) ...
Setting up linux-image-amd64 (4.19+105+deb10u5) ...
Processing triggers for systemd (241-7~deb10u4) ...
Processing triggers for libc-bin (2.28-10) ...
Processing triggers for initramfs-tools (0.133+deb10u1) ...
update-initramfs: Generating /boot/initrd.img-4.19.0-10-amd64
cryptsetup: WARNING: target 'crypted_system' not found in /etc/crypttab
cryptsetup: WARNING: The initramfs image may not contain cryptsetup binaries
    nor crypto modules. If that's on purpose, you may want to uninstall the
    'cryptsetup-initramfs' package in order to disable the cryptsetup initramfs
    integration and avoid this warning.
root@rescue:/#
```

</details>
<!-- markdownlint-enable MD033 -->

Don’t get confused by the warnings, we are going to fix that in the couple of next steps :slightly_smiling_face:

## Configuring /etc/crypttab and /etc/fstab

Next up is the configuration of [/etc/crypttab](https://linux.die.net/man/5/crypttab) and [/etc/fstab](https://wiki.debian.org/fstab).

First, we'll be starting with `/etc/crypttab` - to do so, we first have to read out the UUID of our system LUKS partition by using (adapt to your crypt device if necessary):

```plaintext
cryptsetup luksDump /dev/sda3 | grep UUID
```

The given UUID has to be added to the file `/etc/crypttab` in the following format:

```plaintext
<LUKS_device_name> UUID=<UUID> none luks
```

Substitute `<LUKS_device_name>` with the name of your LUKS device (e.g. `crypted_system`) and `<UUID>` with the UUID you received by the command before.

You can either do it by hand or use the following one liner (you can adjust the name of the luks partition):

```plaintext
echo "crypted_system UUID="$(cryptsetup luksDump /dev/sda3 | grep UUID | awk '/UUID/ { print $2 }')" none luks" > /etc/crypttab
```

The next step is to configure `/etc/fstab`.
To do so, we first need to read out the UUID for both the `/boot` partition (which is stored on `/dev/sda2`) and all partitions that LVM is managing for us. Again depending
whether you have multiple VGs (e.g. another drive holding data) or not. This can be easily achieved using `blkid`:

```terminal
root@rescue:/# blkid /dev/sda2 
/dev/sda2: UUID="67619dac-48fd-4140-9040-d31c1d3bba0f" TYPE="xfs" PARTLABEL="Linux filesystem" PARTUUID="e435544d-b8df-4e6a-bf74-7f094c3007e1"
root@rescue:/# blkid /dev/mapper/vg_system-*
/dev/mapper/vg_system-home: UUID="e560e1fc-897e-4738-832c-09c2663048ba" TYPE="xfs"
/dev/mapper/vg_system-root: UUID="c1934670-c520-4d57-b87c-69a376fa51e4" TYPE="xfs"
/dev/mapper/vg_system-swap: UUID="a2f49117-66f0-489b-9092-6548c272766d" TYPE="swap"
/dev/mapper/vg_system-tmp: UUID="d9f8f8a5-3a5f-4199-817c-a5c454281aeb" TYPE="xfs"
/dev/mapper/vg_system-var: UUID="1408bbe7-88a7-4181-85bc-edcc7b2ef054" TYPE="xfs"
/dev/mapper/vg_system-var_log: UUID="6f354868-dd1e-48d6-8ef7-c0dac5324ac5" TYPE="xfs"
/dev/mapper/vg_system-var_log_audit: UUID="3a5bf013-75e0-4981-a2fb-1666e0e3c293" TYPE="xfs"
/dev/mapper/vg_system-var_tmp: UUID="a2432df7-dd9d-4bc1-9a04-2f118463bf31" TYPE="xfs"
root@rescue:/# 
```

:information_source: Please note, the UUIDs above have been randomized and thus do not reflect an actual system. Your UUIDs will, of course, vary.

With the above information we can build our `/etc/fstab` in the following format:

```plaintext
<file system UUID> <mount point on the system> <type> <mount options> <dump> <pass>
```

In the following table I summarized the flags (mount options) for each mount point. These are documented at e.g. [linux.die.net](https://linux.die.net/man/8/mount).

| mount point       | flags                      | comment                                                                   |
| :---------------- | :------------------------- | :-----------------------------------------------------------------------  |
| `/`               | `defaults`                 | -                                                                         |
| `/boot`           | `defaults`                 | -                                                                         |
| `/tmp`            | `rw,nosuid,nodev,noexec`   | For security reasons, you should consider adding `nosuid,nodev,noexec`    |
| `/var`            | `rw`                       | -                                                                         |
| `/var/log`        | `rw,nosuid,nodev,noexec`   | For security reasons, you should consider adding `nosuid,nodev,noexec`    |
| `/var/log/audit`  | `rw,nosuid,nodev,noexec`   | For security reasons, you should consider adding `nosuid,nodev,noexec`    |
| `/var/tmp`        | `rw,nosuid,nodev`          | For security reasons, you should consider adding `nosuid,nodev`           |
| `/home`           | `rw,nosuid,nodev`          | For security reasons, you should consider adding `nosuid,nodev`           |
| `swap`            | `sw`                       | -                                                                         |

Please note, the hardening mount options are taken from the [CIS Benchmark for Debian Bullseye](https://www.cisecurity.org/benchmark/debian_linux).

The end result will look something like this:

```terminal
root@rescue:/# cat /etc/fstab 
# file system                                   mount point     type    options                 dump    pass
UUID=c1934670-c520-4d57-b87c-69a376fa51e4       /               xfs     defaults                0       1
UUID=67619dac-48fd-4140-9040-d31c1d3bba0f       /boot           xfs     defaults                0       1
UUID=d9f8f8a5-3a5f-4199-817c-a5c454281aeb       /tmp            xfs     rw,nosuid,nodev         0       2
UUID=1408bbe7-88a7-4181-85bc-edcc7b2ef054       /var            xfs     rw                      0       2
UUID=6f354868-dd1e-48d6-8ef7-c0dac5324ac5       /var/log        xfs     rw,nosuid,nodev,noexec  0       2
UUID=3a5bf013-75e0-4981-a2fb-1666e0e3c293       /var/log/audit  xfs     rw,nosuid,nodev,noexec  0       2
UUID=a2432df7-dd9d-4bc1-9a04-2f118463bf31       /var/tmp        xfs     rw,nosuid,nodev         0       2
UUID=e560e1fc-897e-4738-832c-09c2663048ba       /home           xfs     rw,nosuid,nodev         0       2
UUID=a2f49117-66f0-489b-9092-6548c272766d       none            swap    sw                      0       0
root@rescue:/#
```

Some applications still depend on the `/etc/mtab` (although deprecated), rather than `/etc/fstab`; So as a precaution we create a symlink from all current mounts (which are
stored in `/proc/mounts`) to `/etc/mtab`:

```terminal
root@rescue:/# ln -s /proc/mounts /etc/mtab
root@rescue:/#
```

### Optional: Configuring automatical unlock of the data partition

In order to unlock the data LUKS partition after the system LUKS has been unlocked, we can make use of a so-called
[keyfile](https://wiki.archlinux.org/title/dm-crypt/Device_encryption#Keyfiles).

What we can to do is:

- Create a keyfile
- Restrict the permissions of the keyfile, so only the user root has access to it
- Add the keyfile to the data LUKS partition
- Add an additional entry in `/etc/crypttab`, so the data LUKS partition gets automatically unlocked, when the system LUKS partition gets unlocked

First, we need to create a keyfile:

```terminal
root@rescue:/# dd if=/dev/urandom of=/root/keyfile bs=1024 count=4
4+0 records in
4+0 records out
4096 bytes (4.1 kB, 4.0 KiB) copied, 0.000149027 s, 27.5 MB/s
root@rescue:/#
```

Next, we set the appropriate permissions on this file:

```terminal
root@rescue:/# chmod 0400 /root/keyfile 
root@rescue:/#
```

Now, we add the keyfile to the data LUKS partition.
For this process you need to enter the password, which you used to create the LUKS (data) partition:

```terminal
root@rescue:/# cryptsetup luksAddKey /dev/sdb /root/keyfile 
Enter any existing passphrase: 
root@rescue:/#
```

We can verify, that adding of the keyfile worked properly using `cryptsetup luksDump` - here we can see, that two key slots are taken:

```terminal
root@rescue:/# cryptsetup luksDump /dev/sdb
LUKS header information for /dev/sdb
 
Version:        1
Cipher name:    aes
Cipher mode:    xts-plain64
Hash spec:      sha1
Payload offset: 4096
MK bits:        512
MK digest:      3d 0b c6 a2 5c d1 37 98 87 70 ee 22 48 48 ff 90 36 2a 5d 46 
MK salt:        e9 cb 0d bc 5f 1f 1d 77 ff 6b 2f 75 c4 3d 52 4b 
                85 e2 ec 95 47 5c 20 8c dd ad 97 60 08 5c c4 a3 
MK iterations:  158375
UUID:           4132d4a6-929e-4ab0-8bdd-ee43065b4b03
 
Key Slot 0: ENABLED
        Iterations:             633663
        Salt:                   a1 55 d6 82 9e 9b b1 26 55 48 88 01 83 50 fa 8b 
                                d2 7d cc 16 bd 25 75 b9 af 5f 70 ef ee a1 fa 4b 
        Key material offset:    8
        AF stripes:             4000
Key Slot 1: ENABLED
        Iterations:             1414363
        Salt:                   a5 9f f6 6e bc f7 4d 8d 5b 3b a7 03 65 f1 90 34 
                                59 07 51 31 58 de ee be a8 58 49 1f 4b 6f 51 a2 
        Key material offset:    512
        AF stripes:             4000
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
root@rescue:/#
```

Finally we add the device `/dev/sdb` with its UUID (taken from `cryptsetup luksDump`) to `/etc/crypttab` with a reference to the keyfile:

```terminal
root@rescue:/# echo 'crypted_data UUID=4132d4a6-929e-4ab0-8bdd-ee43065b4b03 /root/keyfile luks' >> /etc/crypttab 
root@rescue:/# cat /etc/crypttab 
crypted_system UUID=e6864adb-dd67-42a1-867c-6a241b4119b5 none luks
crypted_data UUID=4132d4a6-929e-4ab0-8bdd-ee43065b4b03 /root/keyfile luks
root@rescue:/# 
```

## Installing additional software

Within this section, we are going to install additional packages. Not all packages I am going to install are needed by the operating system to function properly.
First, install the **required packages**:

```plaintext
apt-get install -y makedev lvm2 ssh dropbear busybox initramfs-tools bash-completion kbd console-setup pciutils psmisc grub-pc git plymouth sudo man xfsprogs ntp python
```

**Optionally**, install more packages, such as:

```plaintext
apt-get install vim iftop iotop htop screen postfix mailutils fail2ban
```

:warning: If you are using a software RAID, you additionally need to install the package `mdadm`!

:warning: It can happen that the disks are detected in a different order during boot than during installation time. To avoid issues while booting, please select all
**physical devices** during GRUB installation, when the installer asks you where to install GRUB to.

:warning: Physical devices are for instance `/dev/sda`, `/dev/sdb`; **NOT** the partitions of those devices (such as `/dev/sda1`, etc.).

## Post-installation tasks

Generally the system is fully installed and can be rebooted.
**However**, the remote unlock via SSH is not configured yet and as well some optional post-installation setup tasks are not done (yet).
This includes adding an additional user, configuring vim, installing fail2ban as a precaution and "hardening" the system a bit.

## Configuring vim

With later versions of vim it is (by default) not possible anymore to copy/paste from vim using your mouse. In order to be able to copy/paste from vim, one needs to create a
`.vimrc` in every users home directory with the following content:

```vim
set clipboard=unnamed
```

It is a pain if you have to do this (and even remembering to do so) every time you add a new user. This is why we make use of `/etc/skel` and simply add a file in there
called `.vimrc` with the line from above.

```plaintext
echo 'set clipboard=unnamed' > /etc/skel/.vimrc
```

By now every new user will have this file in its home directory automatically. As the user root is already created, we need to copy this file to its home directory:

```terminal
root@rescue:/# cp /etc/skel/.vimrc /root/
root@rescue:/# ls -la /root/
total 20
drwx------  2 root root   62 Aug 26 19:08 .
drwxr-xr-x 21 root root 4096 Aug 26 16:59 ..
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   22 Aug 26 19:08 .vimrc
-r--------  1 root root 4096 Aug 26 18:30 keyfile
root@rescue:/# 
```

## Adding an additional user

Adding an additional user is pretty easy and does not need much explanation.

```terminal
root@rescue:/# adduser steffen --disabled-password --gecos "steffen"
Adding user `steffen' ...
Adding new group `steffen' (1000) ...
Adding new user `steffen' (1000) with group `steffen' ...
Creating home directory `/home/steffen' ...
Copying files from `/etc/skel' ...
root@rescue:/# 
```

Adding the user to sudoers:

```terminal
root@rescue:/# usermod -aG sudo steffen
root@rescue:/# groups steffen
steffen : steffen sudo
root@rescue:/#
```

Add (an) SSH public key(s) to the users `authorized_keys` file:

```terminal
root@rescue:/# mkdir /home/steffen/.ssh
root@rescue:/# cat >  /home/steffen/.ssh/authorized_keys << "EOF"
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINmwi1h1337Q/hIomn3VWjw5Ky2vd2ujnYyaTLuil5sh Key1
ecdsa-sha2-nistp521 AAAAE2VjZHNhLXNoYTItbmlzdHA313373bmlzdHA1MjEAAACFBADjklC2kev78lChCQobczoF0Zl1lcL9SWiVcfj/KoloxC2/Nz7gN1UQdqJHKJamwWyFdIUy6JucRK730LV4QQhVNwCY2fyf9gGaOP56FvzzeEq9n2pR3WNyD74Bn8u1abt+vALHDEVO6dejZKunJmzsBzXIe79kAH34NwM+gCd4vQbjoQ== Key2
ecdsa-sha2-nistp521 AAAAE2VjZHNhLXNoYTItbmlzdHA1MjEAAAAIbmlzdHA1MjEAAACFBAF11m0mRdIjcwwrfz04t4O+YqngFbP0gP313373+vUADyOYkdhiimbJYN9ge1nVy3nMkMGMf5HNzD9AbO1ACjRyFkta0b2pNyu3kdKu6EQlJWgl25PzJQLjxhllinS+xf4rT5lcdOu3aSKDrG6lV5xXh/Cw/i84VR5NWFftHfRA== Key3
EOF
root@rescue:/# chown -R steffen:steffen /home/steffen/
root@rescue:/# chmod 600 /home/steffen/.ssh/authorized_keys 
root@rescue:/# 
```

Of course, the SSH keys above have been modified :slightly_smiling_face:

Set the default editor to vim:

```terminal
root@rescue:/# update-alternatives --set editor /usr/bin/vim.basic
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/editor (editor) in manual mode
```

Adding an Ansible user to be able to automate the system later on:

```terminal
root@rescue:/# adduser remote-ansible --disabled-password --gecos "Ansible Remote User"
Adding user `remote-ansible' ...
Adding new group `remote-ansible' (1001) ...
Adding new user `remote-ansible' (1001) with group `remote-ansible' ...
Creating home directory `/home/remote-ansible' ...
Copying files from `/etc/skel' ...
root@rescue:/#
```

Adding the Ansible user to sudoers:

```temrinal
root@rescue:/# usermod -aG sudo remote-ansible
root@rescue:/# groups remote-ansible
remote-ansible : remote-ansible sudo
root@rescue:/#
```

Add a public key to the Ansible user's `authorized_keys` file:

```terminal
root@rescue:/# mkdir /home/remote-ansible/.ssh
root@rescue:/# cat >  /home/remote-ansible/.ssh/authorized_keys << "EOF"
ecdsa-sha2-nistp521 AAAAE2VjZHNhLXNoYTItbmlzdHA1MjEAAAAIbmlzdHA1Mj313373/ok5i5CSwUuK8y8Zn2URC/ex1cQBfVBANQlfhAe7P4eFK43IdSsnp3uEigsLOr9Uju9QvuniTNuIudkfonmeL91znWyP0KyCciOxZO2O7Mtf6V9GLaA== root@ansible.servers.local
EOF
root@rescue:/# chown -R remote-ansible:remote-ansible /home/remote-ansible/
root@rescue:/# chmod 600 /home/remote-ansible/.ssh/authorized_keys 
root@rescue:/# 
```

In order to allow `sudo` commands without entering a password the suoders file needs to be adjusted - specifically the `%sudo rule` (use `visudo`):

```plaintext
# Allow members of group sudo to execute any command
%sudo   ALL=NOPASSWD: ALL
```

## "Hardening" the system

The system should be hardened - at least a bit. In the next few lines I'll show you how to do a **minimal** hardening.
First, we want to harden the SSHd.
My configuration file (`/etc/ssh/sshd.conf`) looks like follows:

```terminal
root@rescue:/# cat /etc/ssh/sshd_config 
#       $OpenBSD: sshd_config,v 1.100 2016/08/15 12:32:04 naddy Exp $
 
# This is the sshd server system-wide configuration file.  See
# sshd_config(5) for more information.
 
# This sshd was compiled with PATH=/usr/bin:/bin:/usr/sbin:/sbin
 
# The strategy used for options in the default sshd_config shipped with
# OpenSSH is to specify options with their default value where
# possible, but leave them commented.  Uncommented options override the
# default value.
 
Port 1905
#AddressFamily ipv4
ListenAddress 176.9.18.15
 
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
 
# Ciphers and keying
RekeyLimit 64M
 
# Logging
SyslogFacility AUTH
LogLevel INFO
 
# Authentication:
 
LoginGraceTime 1m
PermitRootLogin no
StrictModes yes
MaxAuthTries 3
MaxSessions 5
 
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys
 
# To disable tunneled clear text passwords, change to no here!
PasswordAuthentication no
PermitEmptyPasswords no
 
# Change to yes to enable challenge-response passwords (beware issues with
# some PAM modules and threads)
ChallengeResponseAuthentication no
 
# Set this to 'yes' to enable PAM authentication, account processing,
# and session processing. If this is enabled, PAM authentication will
# be allowed through the ChallengeResponseAuthentication and
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication via ChallengeResponseAuthentication may bypass
# the setting of "PermitRootLogin without-password".
# If you just want the PAM account and session checks to run without
# PAM authentication, then enable this but set PasswordAuthentication
# and ChallengeResponseAuthentication to 'no'.
UsePAM yes
 
AllowAgentForwarding no
AllowTcpForwarding no
GatewayPorts no
X11Forwarding no
#X11DisplayOffset 10
#X11UseLocalhost yes
PermitTTY yes
PrintMotd no
PrintLastLog yes
TCPKeepAlive yes
UseLogin no
UsePrivilegeSeparation sandbox
PermitUserEnvironment no
Compression delayed
ClientAliveInterval 0
ClientAliveCountMax 3
UseDNS no
PidFile /var/run/sshd.pid
MaxStartups 10:30:100
PermitTunnel no
ChrootDirectory none
VersionAddendum none
Banner /etc/issue.net
 
# override default of no subsystems
Subsystem       sftp    /usr/lib/openssh/sftp-server
root@rescue:/#
```

Installing `Fail2Ban` is best-practice with systems facing the internet directly:

```terminal
root@rescue:/# apt-get install -y fail2ban
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  python3-pyinotify python3-systemd whois
Suggested packages:
  monit python-pyinotify-doc
The following NEW packages will be installed:
  fail2ban python3-pyinotify python3-systemd whois
0 upgraded, 4 newly installed, 0 to remove and 0 not upgraded.
Need to get 424 kB of archives.
After this operation, 1,967 kB of additional disk space will be used.
Get:1 http://ftp.de.debian.org/debian bullseye/main amd64 fail2ban all 0.9.6-2 [288 kB]
Get:2 http://ftp.de.debian.org/debian bullseye/main amd64 python3-pyinotify all 0.9.6-1 [26.9 kB]
Get:3 http://ftp.de.debian.org/debian bullseye/main amd64 python3-systemd amd64 233-1 [33.3 kB]
Get:4 http://ftp.de.debian.org/debian bullseye/main amd64 whois amd64 5.2.17~deb9u1 [76.8 kB]
Fetched 424 kB in 0s (3,477 kB/s)
Selecting previously unselected package fail2ban.
(Reading database ... 38168 files and directories currently installed.)
Preparing to unpack .../fail2ban_0.9.6-2_all.deb ...
Unpacking fail2ban (0.9.6-2) ...
Selecting previously unselected package python3-pyinotify.
Preparing to unpack .../python3-pyinotify_0.9.6-1_all.deb ...
Unpacking python3-pyinotify (0.9.6-1) ...
Selecting previously unselected package python3-systemd.
Preparing to unpack .../python3-systemd_233-1_amd64.deb ...
Unpacking python3-systemd (233-1) ...
Selecting previously unselected package whois.
Preparing to unpack .../whois_5.2.17~deb9u1_amd64.deb ...
Unpacking whois (5.2.17~deb9u1) ...
Setting up fail2ban (0.9.6-2) ...
Created symlink /etc/systemd/system/multi-user.target.wants/fail2ban.service â†’ /lib/systemd/system/fail2ban.service.
Setting up whois (5.2.17~deb9u1) ...
Setting up python3-systemd (233-1) ...
Processing triggers for systemd (232-25+deb9u4) ...
Processing triggers for man-db (2.7.6.1-2) ...
Setting up python3-pyinotify (0.9.6-1) ...
```

In my case I only enabled the `sshd jail`.
For that simply create the file `/etc/fail2ban/jail.local` with the following content:

```terminal
root@rescue:/# cat /etc/fail2ban/jail.local
[sshd]
 
port    = 1905
logpath = %(sshd_log)s
backend = %(sshd_backend)s
root@rescue:/#
```

Please adjust the `port` directive according to your setup.

## Preparing the remote unlock via SSH of the LUKS partition

In order to unlock the system's LUKS partition(s) via SSH using Dropbear, we need to do a few things. To be able to remotely unlock via SSH, we need to properly configure both
Dropbear and GRUB for that.

### Configure Dropbear

First, we copy the public SSH key, we just added to the created user, to the `authorized_keys` file for Dropbear.

```terminal
root@rescue:/# cp /home/steffen/.ssh/authorized_keys /etc/dropbear-initramfs/
root@rescue:/#
```

Next we configure Dropbear. We will be using the following settings, which can be read about at [linux.die.net](https://linux.die.net/man/8/dropbear):

| option     | explanation                                                        |
| :--------- | : ---------------------------------------------------------------- |
| -p 605     | Dropbear will listen on port 605                                   |
| -s         | Disable password logins                                            |
| -j         | Disable local port forwarding                                      |
| -k         | Disable remote port forwarding                                     |
| -I 60      | Disconnect the session if no traffic is transmitted for 60 seconds |

You can set those settings easily using the below command:

```plaintext
sed 's/#DROPBEAR_OPTIONS=/DROPBEAR_OPTIONS="-p 605 -s -j -k -I 60"/' -i /etc/dropbear-initramfs/config
```

Verify, whether the settings where set correctly:

```plaintext
grep DROPBEAR_OPTIONS /etc/dropbear-initramfs/config 
```

<!-- markdownlint-disable MD033 -->
<details>
<summary>Sample output:</summary>
```terminal
root@rescue:/# sed 's/#DROPBEAR_OPTIONS=/DROPBEAR_OPTIONS="-p 605 -s -j -k -I 60"/' -i /etc/dropbear-initramfs/config
root@rescue:/# grep DROPBEAR_OPTIONS /etc/dropbear-initramfs/config
DROPBEAR_OPTIONS="-p 605 -s -j -k -I 60"
root@rescue:/#
```
</details>
<!-- markdownlint-enable MD033 -->

### Configure GRUB

Configuring GRUB for remote unlocking is as easy as configuring Dropbear.

However, we need to use the "old style-naming" of the ethernet devices (e.g. `eth0`), so we have to use both `net.ifnames=0` and `biosdevname=0` in the
`GRUB_CMDLINE_LINUX_DEFAULT`, which can be found in `/etc/default/grub`.
Additionally we need to enter the IP address, the gateway, the netmask and the ethernet device to use for the remote connection.

Following is the format of the IP parameter (a detailed explanation can be looked up at [kernel.org](https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt)):

```plaintext
ip=<ip>::<gateway>:<netmask>::<ethernet device>:none
```

Substitute `<ip>`, `<gateway>`, `<netmask>` and `<ethernet device>` with your values.

Following an example configuration:

```plaintext
GRUB_CMDLINE_LINUX_DEFAULT="net.ifnames=0 biosdevname=0 ip=159.69.68.69::159.69.68.65:255.255.255.192::eth0:none"
```

### Adding drivers to initramfs

To be able to unlock the system LUKS partition via SSH, an IP address on any ethernet device (in our case `eth0`) is required, obviously. For that reason, it is necessary to
provide the correct drivers to the `initramfs image`.
You can find out which driver your ethernet device uses with the following command:

```terminal
root@rescue:/# grep DRIVER /sys/class/net/eth0/device/uevent 
DRIVER=e1000e
root@rescue:/#
```

Now we need to tell `initramfs-tools` to include the above driver in our `initramfs image`:

```terminal
root@rescue:/# echo "e1000e" >> /etc/initramfs-tools/modules
root@rescue:/# cat /etc/initramfs-tools/modules 
# List of modules that you want to include in your initramfs.
# They will be loaded at boot time in the order below.
#
# Syntax:  module_name [args ...]
#
# You must run update-initramfs(8) to effect this change.
#
# Examples:
#
# raid1
# sd_mod
e1000e
root@rescue:/# 
```

### Generating new initramfs and updating GRUB

Finally we are able to generate a new initramfs and update GRUB. Updating both initramfs and GRUB is necessary as we changed the configuration and/or added new drivers to
the `initramfs image`:

```terminal
root@rescue:/boot# update-initramfs -u -k all
update-initramfs: Generating /boot/initrd.img-4.19.0-10-amd64
root@rescue:/boot# update-grub
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.19.0-10-amd64
Found initrd image: /boot/initrd.img-4.19.0-10-amd64
done
root@rescue:/boot#
```

Now simply logout of the chroot and reboot the system.
You should be able to reach the system on the port you have configured in the Dropbear section, via SSH. Since Dropbear is not aware of any other users than root, you need to
use the user `root` to login.

If you followed my configuration, it's port 605 via SSH. There you'll have to run `cryptroot-unlock` and enter the password for the system LUKS partition.

Congratulations, you managed to manually install a Debian Bullseye :sunglasses:

## Troubleshooting the installation

If something goes wrong during the installation and the system does not come up as expected, you can troubleshoot the installation pretty easily using a live CD (like the
Hetzner rescue system).

First login to the live system as root.
In order to re-mount the system as done during the installation, you can use the script from the section [Mounting the partitions](#Mounting the partitions) or do it manually.
In this case I will do it manually, just to show the general approach behind this procedure.
First, we need to unlock the system's LUKS partition and - if you have it - the data LUKS partition:

```terminal
root@rescue ~ # cryptsetup luksOpen /dev/sda3 crypted_system
Enter passphrase for /dev/sda3: 
root@rescue ~ # cryptsetup luksOpen /dev/sdb crypted_data
Enter passphrase for /dev/sdb: 
root@rescue ~ # ls -la /dev/mapper/
total 0
drwxr-xr-x  2 root root     100 Aug 27 16:29 .
drwxr-xr-x 16 root root    6.4K Aug 27 16:29 ..
crw-------  1 root root 10, 236 Aug 27 16:28 control
lrwxrwxrwx  1 root root       7 Aug 27 16:29 crypted_data -> ../dm-1
lrwxrwxrwx  1 root root       7 Aug 27 16:29 crypted_system -> ../dm-0
root@rescue ~ #
```

Next, we need to make the system aware of our volume groups. This can easily be done using `vgchange -aay`, which will automatically detect all volume groups and active them:

```terminal
root@rescue ~ # vgchange -aay
  1 logical volume(s) in volume group "vg_data" now active
  7 logical volume(s) in volume group "vg_system" now active
root@rescue ~ #
```

Now, we can mount all volumes again to `/mnt`:

```terminal
root@rescue ~ # ls -la /dev/mapper/*
crw------- 1 root root 10, 236 Aug 27 16:28 /dev/mapper/control
lrwxrwxrwx 1 root root       7 Aug 27 16:29 /dev/mapper/crypted_data -> ../dm-1
lrwxrwxrwx 1 root root       7 Aug 27 16:29 /dev/mapper/crypted_system -> ../dm-0
lrwxrwxrwx 1 root root       7 Aug 27 16:33 /dev/mapper/vg_data-var_lib_vz -> ../dm-2
lrwxrwxrwx 1 root root       7 Aug 27 16:33 /dev/mapper/vg_system-home -> ../dm-4
lrwxrwxrwx 1 root root       7 Aug 27 16:33 /dev/mapper/vg_system-root -> ../dm-3
lrwxrwxrwx 1 root root       7 Aug 27 16:33 /dev/mapper/vg_system-swap -> ../dm-8
lrwxrwxrwx 1 root root       7 Aug 27 16:33 /dev/mapper/vg_system-tmp -> ../dm-5
lrwxrwxrwx 1 root root       7 Aug 27 16:33 /dev/mapper/vg_system-var -> ../dm-9
lrwxrwxrwx 1 root root       7 Aug 27 16:33 /dev/mapper/vg_system-var_log -> ../dm-7
lrwxrwxrwx 1 root root       7 Aug 27 16:33 /dev/mapper/vg_system-var_tmp -> ../dm-6
root@rescue ~ # mount /dev/mapper/vg_system-root /mnt/
root@rescue ~ # mount /dev/mapper/vg_system-home /mnt/home/
root@rescue ~ # mount /dev/mapper/vg_system-tmp /mnt/tmp
root@rescue ~ # mount /dev/mapper/vg_system-var /mnt/var/
root@rescue ~ # mount /dev/mapper/vg_system-var_log /mnt/var/log/
root@rescue ~ # mount /dev/mapper/vg_system-var_tmp /mnt/var/tmp/
root@rescue ~ # mount /dev/mapper/vg_data-var_lib_vz /mnt/var/lib/vz/
root@rescue ~ # mount
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
sys on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
udev on /dev type devtmpfs (rw,relatime,size=132021144k,nr_inodes=33005286,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
213.133.99.101:/nfs on /root/.oldroot/nfs type nfs (ro,noatime,vers=3,rsize=8192,wsize=8192,namlen=255,acregmin=600,acregmax=600,acdirmin=600,acdirmax=600,hard,nocto,nolock,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=213.133.99.101,mountvers=3,mountproto=tcp,local_lock=all,addr=213.133.99.101)
overlay on / type overlay (rw,relatime,lowerdir=/nfsroot,upperdir=/ramfs/root,workdir=/ramfs/work)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,mode=755)
tmpfs on /run/lock type tmpfs (rw,nosuid,nodev,noexec,relatime,size=5120k)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=21,pgrp=1,timeout=300,minproto=5,maxproto=5,direct)
mqueue on /dev/mqueue type mqueue (rw,relatime)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime)
fusectl on /sys/fs/fuse/connections type fusectl (rw,relatime)
/dev/mapper/vg_system-root on /mnt type xfs (rw,relatime,attr2,inode64,noquota)
/dev/mapper/vg_system-home on /mnt/home type xfs (rw,relatime,attr2,inode64,noquota)
/dev/mapper/vg_system-tmp on /mnt/tmp type xfs (rw,relatime,attr2,inode64,noquota)
/dev/mapper/vg_system-var on /mnt/var type xfs (rw,relatime,attr2,inode64,noquota)
/dev/mapper/vg_system-var_log on /mnt/var/log type xfs (rw,relatime,attr2,inode64,noquota)
/dev/mapper/vg_system-var_tmp on /mnt/var/tmp type xfs (rw,relatime,attr2,inode64,noquota)
/dev/mapper/vg_data-var_lib_vz on /mnt/var/lib/vz type xfs (rw,relatime,attr2,inode64,noquota)
root@rescue ~ #
```

Before we can chroot into the environment, we need to mount the necessary system partitions:

```terminal
root@rescue ~ # mount -o bind /dev/ /mnt/dev/
root@rescue ~ # mount -t devpts devpts /mnt/dev/pts/
root@rescue ~ # mount -t proc proc /mnt/proc/
root@rescue ~ # mount -t sysfs sys /mnt/sys/
root@rescue ~ # XTERM=xterm-color LANG=C.UTF-8 chroot /mnt /bin/bash
root@rescue:/# pwd
/
root@rescue:/#
```

Now you can change, whatever you need to, unmount everything and reboot the system.

Don't forget to close everything properly:

```terminal
root@rescue ~ # lvchange -a n vg_system
root@rescue ~ # lvchange -a n vg_data
root@rescue ~ # cryptsetup luksClose crypted_system
root@rescue ~ # cryptsetup luksClose crypted_data
root@rescue ~ # 
```
