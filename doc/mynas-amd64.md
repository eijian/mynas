# Ubuntu Linux と btrfs で NAS を作った話


### Ubuntu Server の構築

* Ubuntu Server 24.04 LTS

今回は2024年4月にリリースされた Ubuntu Server 24.04 LTS を使った。

参考URL：https://ubuntu.com/download/server

このページにある"Download 24.04 LTS"を押せばダウンロードが開始される（AMD64版）。

* ISOファイルをUSBメモリへ書き出し（Mac編）

USBメモリを挿したあとデバイス名を確認する。この時は /dev/disk3 がUSBメモリ。

```
 sudo diskutil list                                                   
/dev/disk0 (internal, physical):
  :

/dev/disk1 (synthesized):
  :

/dev/disk3 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *8.1 GB     disk3
   1:               Windows_NTFS PNY USB 2.0             8.1 GB     disk3s1
```

アンマウントしてからFAT32でフォーマットする。

```
% diskutil unMountDisk /dev/disk3
Unmount of all volumes on disk3 was successful
```

このあとCLIでフォーマットしようとしたがうまくいかなかったため、Mac標準ツールの
ディスクユーティリティーを使ってフォーマットした。

参考URL：https://4ddig.tenorshare.com/jp/mac-recovery-solutions/how-to-format-usb-to-fat32-on-mac.html

次にISOイメージを書き込む。

```
% sudo dd if=ubuntu-24.04-live-server-amd64.iso of=/dev/rdisk3 bs=1m
2627+1 records in
2627+1 records out
2754981888 bytes transferred in 252.341345 secs (10917679 bytes/sec)
```

これで起動用USBメモリができた。

* Ubuntu Server のインストール

今回は2024年4月にリリースされた Ubuntu Server 24.04 LTS を使った。
インストール手順はWebにいくらでも解説サイトがあるためそちらを参照願いたい。
インストール時の各種設定・選択ポイントで主要なものを列挙する。

    - "Choose the base for the installation"

      ここでUbuntu Serverの標準パッケージ込みか最低限のみ（minimized）かを選択する。
      不要なパッケージを入れたくない思いでminimizedを選んだが、後で本当に基本的なコマンドも
      入っておらずいちいちパッケージを入れるのが嫌になって"unminimized"コマンドを使ってしまった。
      最初から標準パッケージ込みで導入するのが無難。

    - "Configure a guided storage layout, or create a custom one"

      以下の選択をした。

        - 今回のサーバはNAS専用なのでドライブ全体を使うため "Use an entire disk" を選択
        - 3つあるストレージ装置のうち、OSを入れる予定のSSDを選択
        - "Set up this disk as an LVM group" はLVMが無駄なので使わない

なおネットワークの設定はインストールの途中でどうするか訊いてくるので、インストーラ内で
設定してみた（普段ならインストール後に設定ファイルをいじるが）。特に設定に困るところはなかった。

* sshdの設定

設定ファイルに少し変更を加える。ファイルは'/etc/ssh/sshd_config'である。

```
    :
PermitRootLogin no
    :
PasswordAuthentication yes     ← これは最近はデフォルトがyesのようなのでわざわざいらないと思うが念のため
    :
```


* タイムゾーン設定

デフォルトだとUTCになっている模様、変更しておく。

```
$ sudo timedatectl list-timezones
  :
  Asia/Tokyo
  :
$ sudo timedatectl set-timezone Asia/Tokyo
```

### データ用HDDにbtrfsを作り込む

* ファイルシステム構築

まずは各ドライブの状態を確認する。データ用のHDDが2本（sda, sdb）認識されている。ここにbtrfsを
構成する。

```
$ lsblk -f
NAME     FSTYPE   FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0    squashfs 4.0                                                    0   100% /snap/snapd/21465
loop1                                                                    0   100% /snap/core22/1380
loop2                                                                    0   100% /snap/lxd/28460
sda                                                                               
└─sda1   exfat    1.0   HV01  6383-22AB                                           
sdb                                                                               
└─sdb1   ntfs                 0D2F169E0D2F169E                                    
nvme0n1                                                                           
├─nvme0n1p1
│        vfat     FAT32       7E74-B0FD                                 1G     1% /boot/efi
└─nvme0n1p2
         ext4     1.0         be99d788-26de-496b-9636-19830222b3b2  428.3G     3% /
```

まずはフォーマット。

```
（1本目）
$ sudo mkfs.btrfs -L NASDISK001 -f /dev/sda1

btrfs-progs v6.6.3
See https://btrfs.readthedocs.io for more information.

Performing full device TRIM /dev/sda1 (7.28TiB) ...
NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              NASDISK001
UUID:               c6e20631-c324-4ea8-be3b-74d9216abf59
Node size:          16384
Sector size:        4096
Filesystem size:    7.28TiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP               1.00GiB
  System:           DUP               8.00MiB
SSD detected:       no
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes, free-space-tree
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH     
    1     7.28TiB  /dev/sda1

（2本目）
$ sudo mkfs.btrfs -L NASDISK001 -f /dev/sdb1
btrfs-progs v6.6.3
See https://btrfs.readthedocs.io for more information.

Performing full device TRIM /dev/sdb1 (9.09TiB) ...
NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              NASDISK001
UUID:               c1bb404f-d61e-41ad-9dfd-04de972f3f3d
Node size:          16384
Sector size:        4096
Filesystem size:    9.09TiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP               1.00GiB
  System:           DUP               8.00MiB
SSD detected:       no
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes, free-space-tree
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH     
    1     9.09TiB  /dev/sdb1
```

下記の通り2本ともbtrfsになっている（lsblkコマンドとbtrfsコマンドで確認）。

```
$ lsblk -f
NAME        FSTYPE   FSVER LABEL      UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0       squashfs 4.0                                                         0   100% /snap/snapd/21465
loop1                                                                            0   100% /snap/core22/1380
loop2                                                                            0   100% /snap/lxd/28460
sda                                                                                       
└─sda1      btrfs          NASDISK001 c6e20631-c324-4ea8-be3b-74d9216abf59    ← 1本目
sdb                                                                                       
└─sdb1      btrfs          NASDISK002 d32011c3-970c-4e6d-8ac1-37c2d3c9af2f    ← 2本目
nvme0n1                                                                                   
├─nvme0n1p1 vfat     FAT32            7E74-B0FD                                 1G     1% /boot/efi
└─nvme0n1p2 ext4     1.0              be99d788-26de-496b-9636-19830222b3b2  428.3G     3% /

$ sudo btrfs filesystem show
Label: 'NASDISK002'  uuid: d32011c3-970c-4e6d-8ac1-37c2d3c9af2f
	Total devices 1 FS bytes used 144.00KiB
	devid    1 size 9.09TiB used 2.02GiB path /dev/sdb1

Label: 'NASDISK001'  uuid: c6e20631-c324-4ea8-be3b-74d9216abf59
	Total devices 1 FS bytes used 144.00KiB
	devid    1 size 7.28TiB used 2.02GiB path /dev/sda1
```

つづいてbtrfsの「親ボリューム」のマウントポイントを作成する。普段は/mnt配下にマウントポイントを
作るが、btrfsは親ボリュームとサブボリュームの概念があるようで、混乱するので親ボリュームは/volumeの
下に置く形にする。

```
$ sudo mkdir /volume
$ sudo mkdir /volume/disk1
$ sudo mkdir /volume/disk2
```

/etc/fstabにエントリを追加する。

```
  :
#
# BTRFS
#
LABEL=NASDISK001        /volume/disk1   btrfs   noatime,compress-force=lzo,space_cache=v2       0       0
LABEL=NASDISK002        /volume/disk2   btrfs   noatime,compress-force=lzo,space_cache=v2       0       0
```

次に2つの親ボリュームをマウントする。

```
$ sudo mount /volume/disk1
$ sudo mount /volume/disk2
```

（注）fstabを編集すると、以下のメッセージが出た。マウントはできているようだが指示に従い
実行しておく。

```
$ sudo mount /volume/disk1
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
$ sudosystemctl daemon-reload
```

mountコマンドで確認してみる。問題なさそうだ。

```
$ mount
  :
/dev/sda1 on /volume/disk1 type btrfs (rw,noatime,compress-force=lzo,space_cache=v2,subvolid=5,subvol=/)
/dev/sdb1 on /volume/disk2 type btrfs (rw,noatime,compress-force=lzo,space_cache=v2,subvolid=5,subvol=/)
```

ディスク2本の各親ボリューム内にサブボリュームを作る。

```
$ sudo btrfs subvolume create /volume/disk1/@data
Create subvolume '/volume/disk1/@data'
$ sudo btrfs subvolume create /volume/disk2/@data
Create subvolume '/volume/disk2/@data'
```

サブボリュームをマウントする。まずはfstabにエントリを追加する。

```
LABEL=NASDISK001        /mnt/nasdisk1_data      btrfs   subvol=/@data,noatime,compress-force=lzo,space_cache=v2 0       1
LABEL=NASDISK002        /mnt/nasdisk2_data      btrfs   subvol=/@data,noatime,compress-force=lzo,space_cache=v2 0       1
```

マウントポイントを/mntに作成し、サブボリュームをマウントする。正しくマウントできたようだ。

```
$ sudo mkdir -p /mnt/nasdisk1_data
$ sudo mkdir -p /mnt/nasdisk2_data
$ sudo mount /mnt/nasdisk1_data
$ sudo mount /mnt/nasdisk2_data
$ sudo lsblk -f
NAME        FSTYPE   FSVER LABEL      UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0       squashfs 4.0                                                         0   100% /snap/snapd/21465
loop1                                                                            0   100% /snap/core22/1380
loop2                                                                            0   100% /snap/lxd/28460
sda                                                                                       
└─sda1      btrfs          NASDISK001 c6e20631-c324-4ea8-be3b-74d9216abf59    7.3T     0% /mnt/nasdisk1_data
                                                                                          /volume/disk1
sdb                                                                                       
└─sdb1      btrfs          NASDISK002 d32011c3-970c-4e6d-8ac1-37c2d3c9af2f    9.1T     0% /mnt/nasdisk2_data
                                                                                          /volume/disk2
nvme0n1                                                                                   
├─nvme0n1p1 vfat     FAT32            7E74-B0FD                                 1G     1% /boot/efi
└─nvme0n1p2 ext4     1.0              be99d788-26de-496b-9636-19830222b3b2  428.3G     3% /
```

### SAMBAで公開する

公開用のlinuxユーザを作成する（一般用、非公開用の2ユーザ）。

```
$ sudo groupadd -g 2000 akagi
$ sudo useradd -c 'Eiji' -d /home/eiji -g 2000 -m -s /usr/bin/bash -u 2001 eiji
$ sudo passwd eiji   ← パスワードを設定しておく
$ sudo useradd -c 'for edata' -d /home/redforest13 -g 2000 -m -s /usr/bin/bash -u 2100 redforest13
$ sudo passwd redforest13
```

disk1のサブボリュームに公開用ディレクトリをそれぞれのユーザで作成する。

```
$ cd /mnt/nasdisk1_data
$ sudo mkdir public
$ sudo chown eiji:akagi public
$ sudo mkdir edata
$ sudo chown redforest13 edata
$ sudo chmod 700 edata
$ ls -la
total 4
drwxr-xr-x 1 root        root    22 May  3 12:11 .
drwxr-xr-x 4 root        root  4096 May  3 11:01 ..
drwx------ 1 redforest13 root     0 May  3 12:11 edata
drwxr-xr-x 1 eiji        akagi    0 May  3 12:10 public
```

sambaパッケージを入れる。

```
$ sudo apt update
  :
$ sudo apt install samba
  :
$ samba --version
Version 4.19.5-Ubuntu
```

設定ファイル /etc/samba/smb.conf を編集しする。ポイントは以下の通り。

* グローバル設定：ゲスト接続不可、WIN用文字セットの設定、ベースはUTF8にする。
* 一般利用フォルダ： "Public"
* 非公開用フォルダ： "edata"

```
[global]
  :
  workgroup = AKAGI
  :
;   usershare allow guests = yes
   usershare allow guests = no      ← ゲストの接続を不可とする

  dos charset = CP932               ← Win系で文字が化けないため
  unix charset = UTF8
  display charset = UTF8

;[printers]                         ← プリンタ関係はないので全てコメントアウトする
  :
;[print$]
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


[edata]
  path = /mnt/nasdisk1_data/edata
  writeable = yes
  browseable = yes

  create mask = 0700
  directory mask = 0700

  case sensitive = yes
;  force user = redforest13
  guest ok = no
  hosts allow = 192.168.11.
  passdb backend = smbpasswd
```

sambaログイン用パスワードの設定をする。

```
$ sudo smbpasswd -a eiji 
Unknown parameter encountered: "display charset"
Ignoring unknown parameter "display charset"
New SMB password:
Retype new SMB password:
Added user eiji.
$ sudo smbpasswd -a redforest13
New SMB password:
Retype new SMB password:
Added user redforest13.
```

最後にsamba関連サービスを起動する。

```
$ sudo systemctl restart smbd
$ sudo systemctl restart nmbd
ubuntu@alexandra:~$ sudo systemctl enable smbd
Synchronizing state of smbd.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable smbd
ubuntu@alexandra:~$ sudo systemctl enable nmbd
Synchronizing state of nmbd.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable nmbd
```

### btrbkの導入

2本のディスク間のデータ同期を取る方法を検討していたが、現時点ではbtrbkが良さそうなので
採用する。パッケージのインストールから。次に設定ファイルをexampleファイルからコピーしておく。

```
$ sudo apt install btrbk
$ cd /etc/btrbk
$ sudo cp btrbk.conf.example btrbk.conf
```

設定ファイルの最後に設定を追加する。

```
# Added by E. Akagi 2024-05-03
#
volume /volume/disk1
  subvolume @data
    snapshot_dir   @snapshot/@data
    target send-receive /volume/disk2/
```

スナップショット置き場を作成しておく。

```
$ sudo mkdir -p /volume/disk1/@snapshot/@data
```

バックアップを実行してみる。

```
$ sudo btrbk -q run
```

















---
