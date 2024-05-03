


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

---

* macでUSBメモリにISOを書く

状態確認

```
% sudo diskutil list                                                   
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *251.0 GB   disk0
   1:                        EFI EFI                     314.6 MB   disk0s1
   2:                 Apple_APFS Container disk1         250.7 GB   disk0s2

/dev/disk1 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +250.7 GB   disk1
                                 Physical Store disk0s2
   1:                APFS Volume Machintosh HD - Data    132.2 GB   disk1s1
   2:                APFS Volume Preboot                 3.5 GB     disk1s2
   3:                APFS Volume Recovery                1.2 GB     disk1s3
   4:                APFS Volume Machintosh HD           11.9 GB    disk1s4
   5:              APFS Snapshot com.apple.os.update-... 11.9 GB    disk1s4s1
   6:                APFS Volume VM                      2.1 GB     disk1s5

/dev/disk3 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *8.1 GB     disk3
   1:               Windows_NTFS PNY USB 2.0             8.1 GB     disk3s1
```

* フォーマットする


```
% sudo diskutil list                                              
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *251.0 GB   disk0
   1:                        EFI EFI                     314.6 MB   disk0s1
   2:                 Apple_APFS Container disk1         250.7 GB   disk0s2

/dev/disk1 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +250.7 GB   disk1
                                 Physical Store disk0s2
   1:                APFS Volume Machintosh HD - Data    132.2 GB   disk1s1
   2:                APFS Volume Preboot                 3.5 GB     disk1s2
   3:                APFS Volume Recovery                1.2 GB     disk1s3
   4:                APFS Volume Machintosh HD           11.9 GB    disk1s4
   5:              APFS Snapshot com.apple.os.update-... 11.9 GB    disk1s4s1
   6:                APFS Volume VM                      2.1 GB     disk1s5

/dev/disk3 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *8.1 GB     disk3
   1:                 DOS_FAT_32 UBUNTU2404              8.1 GB     disk3s1
```


* アンマウントする

```
% diskutil unMountDisk /dev/disk3
Unmount of all volumes on disk3 was successful
```

* ISOイメージを書き込む

```
% sudo dd if=ubuntu-24.04-live-server-amd64.iso of=/dev/rdisk3 bs=1m
2627+1 records in
2627+1 records out
2754981888 bytes transferred in 252.341345 secs (10917679 bytes/sec)
```




* Ubuntu Server install

"Choose the base for the installation"

  Ubuntu Server (minimized)

* "Configure a guided storage layout, or create a custom one"

  - "Use an entire disk"
	  -> SSDドライブを選択

	- "Set up this disk as an LVM group
	  -> LVMにしない

* SSH Configuration

  - "Install OpenSSH server" にチェック

* 最低限のパッケージ

```
$ sudo apt update
$ sudo apt upgrade
```

  - iputils-ping
	- less
	- vim

	



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


## スナップショットを作成する

* スナップショット用ディレクトリを作成する。

$ sudo mkdir -p /volume/disk1/@snapshot/@data
$ sudo mkdir -p /volume/disk2/@snapshot/@data

* スナップショットの作成

★ '-r'オプションでread-onlyになる。

/$ sudo btrfs subvolume snapshot -r /volume/disk1/@data /volume/disk1/@snapshot/@data/ss-20240414
Create a readonly snapshot of '/volume/disk1/@data' in '/volume/disk1/@snapshot/@data/ss-20240414'
$ sudo btrfs subvolume snapshot -r /volume/disk2/@data /volume/disk2/@snapshot/@data/ss-20240414
Create a readonly snapshot of '/volume/disk2/@data' in '/volume/disk2/@snapshot/@data/ss-20240414'

★コマンドを叩くごとにスナップショットが作られる

* スナップショットを消してみる

$ sudo ls -l /volume/disk1/@snapshot/@data/
total 0
drwxr-xr-x 1 root root 34 Mar 24 07:37 ss-20240414
$ sudo btrfs subvolume delete /volume/disk1/@snapshot/@data/ss-20240414
Delete subvolume (no-commit): '/volume/disk1/@snapshot/@data/ss-20240414'
$ sudo ls -l  /volume/disk1/@snapshot/@data/
total 0

★ ss-20240414 が消えている（@snapshot/@dataはそのまま）



## データ破壊の検出

scrubサブコマンドを使う

$ sudo btrfs scrub start /volume/disk1/@data
scrub started on /volume/disk1/@data, fsid 319ca66d-36e2-4541-af20-37a51053a0b3 (pid=79633)
$ sudo btrfs scrub status /volume/disk1/@data
UUID:             319ca66d-36e2-4541-af20-37a51053a0b3
Scrub started:    Sun Apr 14 10:02:22 2024
Status:           finished
Duration:         0:00:15
Total to scrub:   466.80MiB
Rate:             31.12MiB/s
Error summary:    no errors found     ← 破壊がなければこうなる？



## btrbkの導入

* インストール

> $ sudo apt install btrbk

* 設定ファイル (/etc/btrbk/btrbk.conf)

サンプルのvolume設定はコメントアウトし以下を追加。

```
volume /volume/disk1
  subvolume @data
    snapshot_dir   @snapshot/@data
    target send-receive /volume/disk2/
```

btrbkの実行。実行ログは /var/log/btrbk.log

```
$ btrbk -q run
```

結果として、スナップショットとバックアップが作成される。

```
/volume/disk1/@snapshot/@data/@data.20240414T1205    ← スナップショット
/volume/disk2/@data.20240414T1205                    ← バックアップ
```

しかしデフォルトでどちらともread-onlyになるらしい。
参考URLに柿記述あり。使う段になったらrwでスナップショットを作る？

> Note that btrbk snapshots and backups are read-only, this means you have to create
> a run-time (rw) snapshot before booting into it:
>
> # btrfs subvolume snapshot /mnt/btr_pool/backup/btrbk/rootfs-20150101 /mnt/btr_pool/rootfs_testing

$ sudo btrfs subvolume show /volume/disk2  ← 親ボリューム
/
	Name: 			<FS_TREE>
	UUID: 			35ba95b0-b05b-429a-a232-0df6e09cdf0a
	Parent UUID: 		-
	Received UUID: 		-
	Creation time: 		2024-03-24 07:21:46 +0000
	Subvolume ID: 		5
	Generation: 		14
	Gen at creation: 	0
	Parent ID: 		0
	Top level ID: 		0
	Flags: 			-
	Send transid: 		0
	Send time: 		2024-03-24 07:21:46 +0000
	Receive transid: 	0
	Receive time: 		-
	Snapshot(s):
ubuntu@ubuntu:~$ sudo btrfs subvolume show /volume/disk2/@data.20240414T1205  ← btrbkで作られたROサブボリューム
@data.20240414T1205
	Name: 			@data.20240414T1205
	UUID: 			724e7ce2-5c4d-bc44-b57b-376e377ea7e2
	Parent UUID: 		-
	Received UUID: 		4acc9007-be4f-7649-8e61-9443f81ed9c9
	Creation time: 		2024-04-14 12:05:45 +0000
	Subvolume ID: 		258
	Generation: 		23
	Gen at creation: 	14
	Parent ID: 		5
	Top level ID: 		5
	Flags: 			readonly
	Send transid: 		49
	Send time: 		2024-04-14 12:05:45 +0000
	Receive transid: 	21
	Receive time: 		2024-04-14 12:10:55 +0000
	Snapshot(s):
ubuntu@ubuntu:~$ sudo btrfs subvolume snapshot /volume/disk2/@data.20240414T1205 /volume/disk2/@data  ← RWスナップショット
Create a snapshot of '/volume/disk2/@data.20240414T1205' in '/volume/disk2/@data'
ubuntu@ubuntu:~$ sudo btrfs subvolume show /volume/disk2/@data  ← 読み書きできる？
@data
	Name: 			@data
	UUID: 			1cf3cf18-9fc2-7e4c-9813-cbd5f93e26c4
	Parent UUID: 		724e7ce2-5c4d-bc44-b57b-376e377ea7e2
	Received UUID: 		-
	Creation time: 		2024-04-14 12:40:08 +0000
	Subvolume ID: 		259
	Generation: 		26
	Gen at creation: 	26
	Parent ID: 		5
	Top level ID: 		5
	Flags: 			-
	Send transid: 		0
	Send time: 		2024-04-14 12:40:08 +0000
	Receive transid: 	0
	Receive time: 		-
	Snapshot(s):

* 強制的にバックアップしたサブボリュームをRWにする。

$ sudo btrfs property set -f -ts /volume/disk2/@data.20240414T1205 ro false
$ sudo btrfs property get -ts /volume/disk2/@data.20240414T1205
ro=false
$ sudo btrfs subvolume show /volume/disk2/@data.20240414T1205
@data.20240414T1205
	Name: 			@data.20240414T1205
	UUID: 			724e7ce2-5c4d-bc44-b57b-376e377ea7e2
	Parent UUID: 		-
	Received UUID: 		-                                     ← UUIDが消えた
	Creation time: 		2024-04-14 12:05:45 +0000
	Subvolume ID: 		258
	Generation: 		26
	Gen at creation: 	14
	Parent ID: 		5
	Top level ID: 		5
	Flags: 			-                                           ← readonlyが消えた
	Send transid: 		0
	Send time: 		2024-04-14 12:05:45 +0000
	Receive transid: 	30
	Receive time: 		2024-04-14 12:58:49 +0000
	Snapshot(s):


★試すべきこと
・masterをとりはずし、/mnt/nasdisk1_data へ /volume/disk2/@data をマウントしてみる→共有できるか？
★できた。@data.XXX に対し @data でシンボリックリンクを貼り、下記のようにfstabのLABELをサブ側に
変えることで実現可能。

$ cat /etc/fstab 
LABEL=writable	/	ext4	discard,errors=remount-ro	0 1
LABEL=system-boot       /boot/firmware  vfat    defaults        0       1
LABEL=NASDISK001	/volume/disk1	btrfs	noatime,compress-force=lzo,space_cache=v2	0	0
LABEL=NASDISK002	/volume/disk2	btrfs	noatime,compress-force=lzo,space_cache=v2	0	0
#LABEL=NASDISK001	/mnt/nasdisk1_data	btrfs	subvol=/@data,noatime,compress-force=lzo,space_cache=v2	0	0
LABEL=NASDISK002	/mnt/nasdisk1_data	btrfs	subvol=/@data,noatime,compress-force=lzo,space_cache=v2	0	0
#LABEL=NASDISK002	/mnt/nasdisk2_data	btrfs	subvol=/@data,noatime,compress-force=lzo,space_cache=v2	0	0


ファイルを増やした後、マウント設定を元に戻し再起動。その後 btrbk -q runを実行すると警告が出た。

---
$ sudo btrbk -q run
WARNING: Target subvolume "/volume/disk2/@data.20240414T1205" exists, but is not a receive target of "/volume/disk1/@snapshot/@data/@data.20240414T1205"
WARNING: Please delete stray subvolumes: "btrbk clean /volume/disk2"
WARNING: Skipping backup of: /volume/disk1/@snapshot/@data/@data.20240414T1205
---

警告に従い clean コマンドを実行

$ sudo btrbk clean /volume/disk2
--------------------------------------------------------------------------------
Cleanup Summary (btrbk command line client, version 0.31.3)

    Date:   Sun Apr 14 14:29:40 2024
    Config: /etc/btrbk/btrbk.conf
    Filter: /volume/disk2

Legend:
    ---  deleted subvolume (incomplete backup)
--------------------------------------------------------------------------------
/volume/disk2/@data.*
--- /volume/disk2/@data.20240414T1205


clean後はバックアップが消えた。@dataの名で作ったシンボリックリンクは消す。

$ ls -la /volume/disk2
total 24
drwxr-xr-x 1 root root   10 Apr 14 14:29 .
drwxr-xr-x 4 root root 4096 Mar 24 07:24 ..
lrwxrwxrwx 1 root root   19 Apr 14 14:18 @data -> @data.20240414T1205
$ sudo rm /volume/disk2/@data
$ ls -la /volume/disk2
total 20
drwxr-xr-x 1 root root    0 Apr 14 14:31 .
drwxr-xr-x 4 root root 4096 Mar 24 07:24 ..


再度 btrbk を実行

$ sudo btrbk -q run


別日にbtrbkを実行すると、サブボリュームが増えた。

$ sudo btrbk -q run
$ ls -l /volume/disk2/
total 0
drwxr-xr-x 1 root root 34 Apr 14 14:32 @data.20240414T1205
drwxr-xr-x 1 root root 34 Apr 14 14:32 @data.20240416T1450

ファイルシステムの消費量はdisk1, disk2とも同程度なので、btrbkの実行のたびに容量が
食われることはなさそう。

$ sudo btrfs filesystem show
Label: 'NASDISK002'  uuid: 82262bb6-3e8d-4187-b96b-7b1f8f834b80
	Total devices 1 FS bytes used 466.07MiB
	devid    1 size 28.91GiB used 1.56GiB path /dev/sda1

Label: 'NASDISK001'  uuid: 319ca66d-36e2-4541-af20-37a51053a0b3
	Total devices 1 FS bytes used 466.10MiB
	devid    1 size 28.91GiB used 1.57GiB path /dev/sdb1

* 主ディスクにファイル追加（btrbk実行前）

$ sudo btrfs filesystem show
Label: 'NASDISK002'  uuid: 82262bb6-3e8d-4187-b96b-7b1f8f834b80
	Total devices 1 FS bytes used 466.07MiB
	devid    1 size 28.91GiB used 1.56GiB path /dev/sda1

Label: 'NASDISK001'  uuid: 319ca66d-36e2-4541-af20-37a51053a0b3
	Total devices 1 FS bytes used 562.75MiB
	devid    1 size 28.91GiB used 1.57GiB path /dev/sdb1

※ btrbk実行 -> なぜか同じ大きさにならない

$ sudo btrfs filesystem show
Label: 'NASDISK002'  uuid: 82262bb6-3e8d-4187-b96b-7b1f8f834b80
	Total devices 1 FS bytes used 497.37MiB
	devid    1 size 28.91GiB used 1.56GiB path /dev/sda1

Label: 'NASDISK001'  uuid: 319ca66d-36e2-4541-af20-37a51053a0b3
	Total devices 1 FS bytes used 562.79MiB
	devid    1 size 28.91GiB used 1.57GiB path /dev/sdb1

時間が経つと完全に同期した

$ sudo btrfs filesystem show
Label: 'NASDISK002'  uuid: 82262bb6-3e8d-4187-b96b-7b1f8f834b80
	Total devices 1 FS bytes used 562.75MiB
	devid    1 size 28.91GiB used 1.56GiB path /dev/sda1

Label: 'NASDISK001'  uuid: 319ca66d-36e2-4541-af20-37a51053a0b3
	Total devices 1 FS bytes used 562.74MiB
	devid    1 size 28.91GiB used 1.57GiB path /dev/sdb1

---

## 参考URL

https://zenn.dev/yotan/articles/8082cfe1060860
https://qiita.com/hanaata/items/b4243f49e83baa20b425
https://zenn.dev/daddy_yukio/articles/8c6a15fc09548a
https://blog.bridgey.dev/2020/03/14/backup-using-btrbk/
https://blog.bridgey.dev/2020/02/23/convert-to-raid1-on-btrfs/
https://github.com/digint/btrbk/blob/master/doc/FAQ.md
https://cat-in-136.github.io/2021/09/zenzen-wakaranai-oretachi-funikide-btrfs-wo-tsukatteiru.html



---
