


* Image download



* image unpack

% xz -d ubuntu-22.04.4-preinstalled-server-arm64+raspi.img.xz

* image copy

% df
Filesystem                           512-blocks       Used    Available Capacity    iused       ifree %iused  Mounted on
/dev/disk1s4s1                        489620264   23209728    150500256    14%     356049   752501280    0%   /
devfs                                       381        381            0   100%        660           0  100%   /dev
/dev/disk1s2                          489620264    6814736    150500256     5%       1402   752501280    0%   /System/Volumes/Preboot
   :
/dev/disk2s1                             516190     247303       268887    48%          0           0  100%   /Volumes/system-boot


% sudo diskutil unmount /dev/disk2s1
Volume system-boot on disk2s1 unmounted

% sudo dd bs=1m if=ubuntu-22.04.4-preinstalled-server-arm64+raspi.img of=/dev/rdisk2
4162+0 records in
4162+0 records out
4364173312 bytes transferred in 444.656423 secs (9814709 bytes/sec)



* set IP address


$ cd /etc/netplan
$ cp 50-cloud-init.yaml 60-cloud-init.yaml

---
network:
    version: 2
    wifis:
        wlan0:
            optional: false
            dhcp4: false
        dhcp6: false
        addresses: [192.168.11.210/24]
        routes:
            - to: default
              via: 192.168.1.1 
        nameservers:
            addresses: [192.168.11.1, 8.8.8.8]
        access-points:
            "KYOTARO-4F5G2":
                password: "xxxx"
---

% sudo chmod 600 60-.....

% sudo netplan apply


* sshd setting

$ diff /etc/ssh/sshd_config /etc/ssh/sshd_config.org 
34d33
< PermitRootLogin no
59d57
< PasswordAuthentication yes


* create Btrfs drive on USB memory

現時点でUSBドライブは sda として認識か？

'''
$ lsblk -f
NAME        FSTYPE FSVER LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0                                                                           0   100% /snap/core20/2186
loop1                                                                           0   100% /snap/lxd/27038
loop2                                                                           0   100% /snap/snapd/20674
sda                                                                                      
└─sda1      vfat   FAT32             91AB-019A                                           
mmcblk0                                                                                  
├─mmcblk0p1 vfat   FAT32 system-boot E44D-E979                             184.2M    27% /boot/firmware
└─mmcblk0p2 ext4   1.0   writable    805dc0b8-aabf-48de-bb8e-72905041e507   25.2G     9% /
'''

ファイルシステム構築

'''
$ sudo mkfs.btrfs -L NASDISK001 -f /dev/sda1
btrfs-progs v5.16.2
See http://btrfs.wiki.kernel.org for more information.

NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              NASDISK001
UUID:               319ca66d-36e2-4541-af20-37a51053a0b3
Node size:          16384
Sector size:        4096
Filesystem size:    28.91GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP             256.00MiB
  System:           DUP               8.00MiB
SSD detected:       no
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1    28.91GiB  /dev/sda1
'''

もう一度状態確認

'''
$ sudo lsblk -f
NAME        FSTYPE FSVER LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0                                                                           0   100% /snap/core20/2186
loop1                                                                           0   100% /snap/lxd/27038
loop2                                                                           0   100% /snap/snapd/20674
sda                                                                                      
└─sda1                                                                                   
mmcblk0                                                                                  
├─mmcblk0p1 vfat   FAT32 system-boot E44D-E979                             184.2M    27% /boot/firmware
└─mmcblk0p2 ext4   1.0   writable    805dc0b8-aabf-48de-bb8e-72905041e507   25.2G     9% /

$ sudo btrfs filesystem show
Label: 'NASDISK001'  uuid: 319ca66d-36e2-4541-af20-37a51053a0b3
	Total devices 1 FS bytes used 144.00KiB
	devid    1 size 28.91GiB used 536.00MiB path /dev/sda1
'''

* mount btrfs disk

まずはbtrfsの親ボリュームをマウントする場所を作る。

'''
$ sudo mkdir /volume
$ sudo mkdir /volume/disk1
'''

fstabに書いておく

'''
  :
LABEL=NASDISK001	/volume/disk1	btrfs	noatime,compress-force=lzo,space_cache=v2	0	0
LABEL=NASDISK001	/mnt/nasdisk1_data	btrfs	subvol=/@data,noatime,compress-force=lzo,space_cache=v2	0	0
'''

マウント

'''
% sudo mount /volume/disk1
'''

'''
$ sudo btrfs subvolume create /volume/disk1/@data
Create subvolume '/volume/disk1/@data'
$ sudo mkdir -p /mnt/nasdisk1_data
$ sudo mount /mnt/nasdisk1_data
'''

## -----

groupとuserの追加

'''
% sudo groupadd -g 2000 akagi
$ sudo useradd -c 'Eiji Akagi' -d /home/eiji -g 2000 -m -u 2001 eiji
$ sudo useradd -c 'edata' -d /home/redforest13 -g 2000 -m -s /usr/bin/bash -u 2100 redforest13
'''

* packages

% sudo apt update


-----

パーティションを切る

まずは内容をクリア

$ sudo dd if=/dev/zero of=/dev/sdb bs=512 count=64
64+0 records in
64+0 records out
32768 bytes (33 kB, 32 KiB) copied, 215.324 s, 0.2 kB/s

fdiskでパーティションを作成する -> だめだった

$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x461dae24.

Command (m for help): d
No partition is defined yet!

Command (m for help): p

Disk /dev/sdb: 115.23 GiB, 123731968000 bytes, 241664000 sectors
Disk model: BUM-3D          
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x461dae24

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-241663999, default 2048): 2048
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-241663999, default 241663999): 

Created a new partition 1 of type 'Linux' and of size 115.2 GiB.

Command (m for help): p
Disk /dev/sdb: 115.23 GiB, 123731968000 bytes, 241664000 sectors
Disk model: BUM-3D          
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x461dae24

Device     Boot Start       End   Sectors   Size Id Type
/dev/sdb1        2048 241663999 241661952 115.2G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
/dev/sdb: fsync device failed: Input/output error

ubuntu@ubuntu:~$ sudo lsblk -f
NAME        FSTYPE FSVER LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0                                                                           0   100% /snap/core20/2186
loop1                                                                           0   100% /snap/lxd/27038
loop2                                                                           0   100% /snap/snapd/20674
sda                                                                                      
└─sda1      btrfs        NASDISK001  319ca66d-36e2-4541-af20-37a51053a0b3   28.4G     0% /mnt/nasdisk1_data
                                                                                         /volume/disk1
sdb                                                                                      
mmcblk0                                                                                  
├─mmcblk0p1 vfat   FAT32 system-boot E44D-E979                             115.9M    54% /boot/firmware
└─mmcblk0p2 ext4   1.0   writable    805dc0b8-aabf-48de-bb8e-72905041e507   24.5G    11% /

* 2つ目のUSBメモリをBtrfsでマウント

$ sudo lsblk -f
NAME        FSTYPE  FSVER         LABEL                 UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0                                                                                              0   100% /snap/core20/2186
loop1                                                                                              0   100% /snap/lxd/27038
loop2                                                                                              0   100% /snap/snapd/20674
loop3                                                                                              0   100% /snap/lxd/27437
sda                                                                                                         
└─sda1      btrfs                 NASDISK001            319ca66d-36e2-4541-af20-37a51053a0b3   28.4G     0% /mnt/nasdisk1_data
                                                                                                            /volume/disk1
sdb         iso9660 Joliet Extens Ubuntu 22.04 ja amd64 2022-05-02-03-06-06-00                              
├─sdb1      iso9660 Joliet Extens Ubuntu 22.04 ja amd64 2022-05-02-03-06-06-00                              
├─sdb2      vfat    FAT12         ESP                   8D6C-A9F8                                           
├─sdb3                                                                                                      
└─sdb4      ext4    1.0           writable              76485409-7374-4b6c-b616-e1c2e881e48e                
mmcblk0                                                                                                     
├─mmcblk0p1 vfat    FAT32         system-boot           E44D-E979                             115.9M    54% /boot/firmware
└─mmcblk0p2 ext4    1.0           writable              805dc0b8-aabf-48de-bb8e-72905041e507   24.4G    11% /
ubuntu@ubuntu:~$ !dd
dd if=/dev/zero of=/dev/sdb bs=512 count=64
dd: failed to open '/dev/sdb': Permission denied
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/dev/sdb bs=512 count=64
64+0 records in
64+0 records out
32768 bytes (33 kB, 32 KiB) copied, 0.0252059 s, 1.3 MB/s

fdiskで物理パーティションを作成(/dev/sdb1)

$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

The device contains 'iso9660' signature and it will be removed by a write command. See fdisk(8) man page and --wipe option for more details.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x133fc680.

Command (m for help): p
Disk /dev/sdb: 28.91 GiB, 31037849600 bytes, 60620800 sectors
Disk model: USB Flash Disk  
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x133fc680

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-60620799, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-60620799, default 60620799): 

Created a new partition 1 of type 'Linux' and of size 28.9 GiB.

Command (m for help): p
Disk /dev/sdb: 28.91 GiB, 31037849600 bytes, 60620800 sectors
Disk model: USB Flash Disk  
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x133fc680

Device     Boot Start      End  Sectors  Size Id Type
/dev/sdb1        2048 60620799 60618752 28.9G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.



Btrfsでフォーマット

$ sudo mkfs.btrfs -L NASDISK002 -f /dev/sdb1
btrfs-progs v5.16.2
See http://btrfs.wiki.kernel.org for more information.

NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              NASDISK002
UUID:               82262bb6-3e8d-4187-b96b-7b1f8f834b80
Node size:          16384
Sector size:        4096
Filesystem size:    28.91GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP             256.00MiB
  System:           DUP               8.00MiB
SSD detected:       no
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1    28.91GiB  /dev/sdb1


$ sudo btrfs filesystem show
Label: 'NASDISK001'  uuid: 319ca66d-36e2-4541-af20-37a51053a0b3
	Total devices 1 FS bytes used 160.00KiB
	devid    1 size 28.91GiB used 536.00MiB path /dev/sda1

Label: 'NASDISK002'  uuid: 82262bb6-3e8d-4187-b96b-7b1f8f834b80
	Total devices 1 FS bytes used 144.00KiB
	devid    1 size 28.91GiB used 536.00MiB path /dev/sdb1

ボリューム接続先の作成

$ ls /volume/
disk1
ubuntu@ubuntu:~$ sudo mkdir /volume/disk2
ubuntu@ubuntu:~$ ls /volume/
disk1  disk2




$ sudo btrfs subvolume create /volume/disk2/@data
Create subvolume '/volume/disk2/@data'
ubuntu@ubuntu:~$ sudo mkdir -p /mnt/nasdisk2_data
ubuntu@ubuntu:~$ sudo mount /mnt/nasdisk2_data
ubuntu@ubuntu:~$ sudo lsblk -f
NAME        FSTYPE FSVER LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0                                                                           0   100% /snap/core20/2186
loop1                                                                           0   100% /snap/lxd/27038
loop2                                                                           0   100% /snap/snapd/20674
loop3                                                                           0   100% /snap/lxd/27437
sda                                                                                      
└─sda1      btrfs        NASDISK001  319ca66d-36e2-4541-af20-37a51053a0b3   28.4G     0% /mnt/nasdisk1_data
                                                                                         /volume/disk1
sdb                                                                                      
└─sdb1                                                                      28.4G     0% /mnt/nasdisk2_data
                                                                                         /volume/disk2
mmcblk0                                                                                  
├─mmcblk0p1 vfat   FAT32 system-boot E44D-E979                             115.9M    54% /boot/firmware
└─mmcblk0p2 ext4   1.0   writable    805dc0b8-aabf-48de-bb8e-72905041e507   24.4G    11% /


* SAMBA



$ sudo apt update
$ sudo apt install samba
$ samba --version
Version 4.15.13-Ubuntu

設定ファイルの記述

'''
[global]
   :
;   usershare allow guests = yes
   usershare allow guests = no

  dos charset = CP932
  unix charset = UTF8
  display charset = UTF8

  :

[Public]
  path = /mnt/nasdisk1_data/public
  writeable = yes
  browseable = yes

  case sensitive = yes
  force user = eiji  
  guest ok = no
  hosts allow = 192.168.11.
  passdb backend = smbpasswd
'''





パスワード設定

$ sudo smbpasswd -a eiji
Unknown parameter encountered: "display charset"
Ignoring unknown parameter "display charset"
New SMB password:
Retype new SMB password:
Added user eiji.


---
