
# II. SAN network

## 1. Storage machines

### A. Disks and RAID

🌞 **Configurer des RAID**

voilà les commandes :

```
#creation des raids
[root@sto1 mdadm]# mdadm --create /dev/md4 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
[root@sto1 mdadm]# mdadm --create /dev/md5 --level=1 --raid-devices=2 /dev/sdd /dev/sde
[root@sto1 mdadm]# mdadm --create /dev/md6 --level=1 --raid-devices=2 /dev/sdf /dev/sdg

#formatage des raids
[root@sto1 mdadm]#mkfs.ext3 /dev/md4
[root@sto1 mdadm]#mkfs.ext3 /dev/md5
[root@sto1 mdadm]#mkfs.ext3 /dev/md6

#creation des répertoires de mount
[root@sto1 mdadm]# mkdir /media/md4
[root@sto1 mdadm]# mkdir /media/md5
[root@sto1 mdadm]# mkdir /media/md6

#enregistrement dans fstab pour l'automount
[root@sto1 mdadm]# cat /etc/fstab | tail -3
/dev/md4   /media/md4   ext4      defaults      0   0
/dev/md5   /media/md5   ext4      defaults      0   0
/dev/md6   /media/md6   ext4      defaults      0   0

#rechargement du daemon
[root@sto1 mdadm]# systemctl daemon-reload

# montage des raids
[root@sto1 mdadm]# mount /dev/md4
[root@sto1 mdadm]# mount /dev/md5
[root@sto1 mdadm]# mount /dev/md6

# enregistrement de la conf des raids
[root@sto1 mdadm]# cd /etc/mdadm
[root@sto1 mdadm]# mkdir mdadm
[root@sto1 mdadm]# mdadm --detail --scan > /etc/mdadm/mdadm.conf
[root@sto1 mdadm]# cat mdadm.conf
ARRAY /dev/md4 metadata=1.2 UUID=8392a30d:2a38878c:e8c45885:d414b839
ARRAY /dev/md5 metadata=1.2 UUID=77f3a2ad:f80c3f6b:6b1ce009:ceaec8c8
ARRAY /dev/md6 metadata=1.2 UUID=857d9c2e:b6debbfd:a2580451:91b8c746
```

🌞 **Prouvez que vous avez 3 volumes RAID prêts à l'emploi**

# pour sto1
```
[root@sto1 mdadm]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   10G  0 disk  
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    9G  0 part  
  ├─rl-root 253:0    0    8G  0 lvm   /
  └─rl-swap 253:1    0    1G  0 lvm   [SWAP]
sdb           8:16   0   10G  0 disk  
└─md4         9:4    0   10G  0 raid1 /media/md4
sdc           8:32   0   10G  0 disk  
└─md4         9:4    0   10G  0 raid1 /media/md4
sdd           8:48   0   10G  0 disk  
└─md5         9:5    0   10G  0 raid1 /media/md5
sde           8:64   0   10G  0 disk  
└─md5         9:5    0   10G  0 raid1 /media/md5
sdf           8:80   0   10G  0 disk  
└─md6         9:6    0   10G  0 raid1 /media/md6
sdg           8:96   0   10G  0 disk  
└─md6         9:6    0   10G  0 raid1 /media/md6
sr0          11:0    1 1024M  0 rom   
```

# pour sto2
```
[root@sto2 mdadm]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   10G  0 disk  
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    9G  0 part  
  ├─rl-root 253:0    0    8G  0 lvm   /
  └─rl-swap 253:1    0    1G  0 lvm   [SWAP]
sdb           8:16   0   10G  0 disk  
└─md4         9:4    0   10G  0 raid1 /media/md4
sdc           8:32   0   10G  0 disk  
└─md4         9:4    0   10G  0 raid1 /media/md4
sdd           8:48   0   10G  0 disk  
└─md5         9:5    0   10G  0 raid1 /media/md5
sde           8:64   0   10G  0 disk  
└─md5         9:5    0   10G  0 raid1 /media/md5
sdf           8:80   0   10G  0 disk  
└─md6         9:6    0   10G  0 raid1 /media/md6
sdg           8:96   0   10G  0 disk  
└─md6         9:6    0   10G  0 raid1 /media/md6
sr0          11:0    1 1024M  0 rom   
```
### B. iSCSI target

🌞 **Installer `target`**

dnf install targetcli

🌞 **Démarrer le service `target`**

systemctl start target
systemctl enable target

🌞 **Configurer les *targets iSCSI***

```
/>  /backstores/fileio create name=data-chunk1 file_or_dev=/dev/md4
Note: block backstore preferred for best results
Created fileio data-chunk1 with size 10727981056

/>  /backstores/fileio create name=data-chunk2 file_or_dev=/dev/md5
Note: block backstore preferred for best results
Created fileio data-chunk2 with size 10727981056

/>  /backstores/fileio create name=data-chunk3 file_or_dev=/dev/md6
Note: block backstore preferred for best results
Created fileio data-chunk3 with size 10727981056

/> /iscsi create iqn.2024-12.tp2.b3:data-chunk1
Created target iqn.2024-12.tp2.b3:data-chunk1.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.

/> /iscsi create iqn.2024-12.tp2.b3:data-chunk2
Created target iqn.2024-12.tp2.b3:data-chunk2.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.

/> /iscsi create iqn.2024-12.tp2.b3:data-chunk3
Created target iqn.2024-12.tp2.b3:data-chunk3.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.

/> /iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
Created Node ACL for iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator

/> /iscsi/iqn.2024-12.tp2.b3:data-chunk2/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
Created Node ACL for iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator

/> /iscsi/iqn.2024-12.tp2.b3:data-chunk3/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
Created Node ACL for iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator

/> /iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/luns/ create /backstores/fileio/data-chunk1 
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator   

/> /iscsi/iqn.2024-12.tp2.b3:data-chunk2/tpg1/luns/ create /backstores/fileio/data-chunk2 
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator      

/> /iscsi/iqn.2024-12.tp2.b3:data-chunk3/tpg1/luns/ create /backstores/fileio/data-chunk3 
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator

/> saveconfig
Last 10 configs saved in /etc/target/backup/.
Configuration saved to /etc/target/saveconfig.json
/> exit
Global pref auto_save_on_exit=true
Last 10 configs saved in /etc/target/backup/.
Configuration saved to /etc/target/saveconfig.json
```

## 2. Chunks machine

### A. Simple iSCSI

🌞 **Installer les tools iSCSI sur `chunk1.tp2.b3`**

[root@chunk1 mdadm]# dnf install iscsi-initiator-utils

🌞 **Configurer un iSCSI initiator**

[root@localhost ~]# cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator

[root@localhost network-scripts]# systemctl start iscsi
[root@localhost network-scripts]# systemctl start iscsid

sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk2 --portal 10.3.1.1 -o new
sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk2 --portal 10.3.1.2 -o new
sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk2 --portal 10.3.2.1 -o new
sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk2 --portal 10.3.2.2 -o new

sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk2 --portal 10.3.1.1 --login
sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk2 --portal 10.3.1.2 --login
sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk2 --portal 10.3.2.1 --login
sudo iscsiadm -m node --targetname iqn.2024-12.tp2.b3:data-chunk2 --portal 10.3.2.2 --login

🌞 **Modifier la configuration du démon iSCSI**

[root@localhost ~]# cat /etc/iscsi/iscsid.conf | grep node.session.timeo
node.session.timeo.replacement_timeout = 0

🌞 **Prouvez que la configuration est prête**

[root@localhost ~]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   10G  0 disk
├─sda1        8:1    0    1G  0 part /boot
└─sda2        8:2    0    9G  0 part
  ├─rl-root 253:0    0    8G  0 lvm  /
  └─rl-swap 253:1    0    1G  0 lvm  [SWAP]
sdb           8:16   0   10G  0 disk
sdc           8:32   0   10G  0 disk
sdd           8:48   0   10G  0 disk
sde           8:64   0   10G  0 disk
sr0          11:0    1 1024M  0 rom

### B. Multipathing

🌞 **Installer les outils multipath sur `chunk1.tp2.b3`**

[root@localhost ~]# dnf install device-mapper-multipath

🌞 **Configurer le fichier `/etc/multipath.conf`**

[root@localhost ~]# cat /etc/multipath.conf
defaults {
  user_friendly_names yes
  find_multipaths yes
  path_grouping_policy failover
  features "1 queue_if_no_path"
  no_path_retry 100
}

🌞 **Démarrer le service `multipathd`**

systemctl start multipathd

🌞 **Et euh c'est tout, il est smart enough**

[root@localhost ~]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   10G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    9G  0 part
  ├─rl-root 253:0    0    8G  0 lvm   /
  └─rl-swap 253:1    0    1G  0 lvm   [SWAP]
sdb           8:16   0   10G  0 disk
└─mpatha    253:2    0   10G  0 mpath
sdc           8:32   0   10G  0 disk
└─mpathb    253:3    0   10G  0 mpath
sdd           8:48   0   10G  0 disk
└─mpathb    253:3    0   10G  0 mpath
sde           8:64   0   10G  0 disk
└─mpatha    253:2    0   10G  0 mpath
sr0          11:0    1 1024M  0 rom

## 3. Formatage et montage

🌞 **Créez une partition sur les devices `mpatha` et `mpathb`**

fdisk /dev/mapper/mpatha
fdisk /dev/mapper/mpathb



🌞 **Formatez en `xfs` les partitions**

mkfs.xfs /dev/mapper/mpatha1
mkfs.xfs /dev/mapper/mpathb1

🌞 **Point de montage `/mnt/data_chunk1`**
🌞 **Point de montage `/mnt/data_chunk2`**

mkdir /mnt/data_chunk1
mkdir /mnt/data_chunk2
mount /dev/mapper/mpatha1 /mnt/data_chunk1
mount /dev/mapper/mpathb1 /mnt/data_chunk2


[root@localhost dev]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   10G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    9G  0 part
  ├─rl-root 253:0    0    8G  0 lvm   /
  └─rl-swap 253:1    0    1G  0 lvm   [SWAP]
sdb           8:16   0   10G  0 disk
└─mpatha    253:2    0   10G  0 mpath
  └─mpatha1 253:4    0   10G  0 part  /mnt/data_chunk1
sdc           8:32   0   10G  0 disk
└─mpathb    253:3    0   10G  0 mpath
  └─mpathb1 253:5    0   10G  0 part  /mnt/data_chunk2
sdd           8:48   0   10G  0 disk
└─mpathb    253:3    0   10G  0 mpath
  └─mpathb1 253:5    0   10G  0 part  /mnt/data_chunk2
sde           8:64   0   10G  0 disk
└─mpatha    253:2    0   10G  0 mpath
  └─mpatha1 253:4    0   10G  0 part  /mnt/data_chunk1
sr0          11:0    1 1024M  0 rom
[root@localhost dev]# df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                1.1G     0  1.1G   0% /dev/shm
tmpfs                445M  6.2M  439M   2% /run
/dev/mapper/rl-root  8.1G  1.5G  6.6G  19% /
/dev/sda1           1014M  395M  620M  39% /boot
tmpfs                223M     0  223M   0% /run/user/1000
/dev/mapper/mpathb1   10G  104M  9.9G   2% /mnt/data_chunk2
/dev/mapper/mpatha1   10G  104M  9.9G   2% /mnt/data_chunk1

## 4. Tests

### A. Simulation de panne

🌞 **Simuler une coupure réseau**

[root@localhost dev]# multipath -ll | grep active
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
  `- 6:0:0:0 sde 8:64 active ready running
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdc 8:32 active ready running
  `- 5:0:0:0 sdd 8:48 active ready running
[root@localhost dev]# multipath -ll | grep active
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
  `- 6:0:0:0 sde 8:64 active faulty running
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdc 8:32 active ready running
  `- 5:0:0:0 sdd 8:48 active faulty running
[root@localhost dev]# multipath -ll | grep active
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdc 8:32 active ready running

### B. Jouer avec les paramètres

🌞 **Resimuler une panne**

➜ **Répliquer le setup de `chunk1` sur les machines `chunk2` et `chunk3`**

🌞 **Preuve du setup**

-> pour chunk2


[root@chunk2 iscsi]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   10G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    9G  0 part
  ├─rl-root 253:0    0    8G  0 lvm   /
  └─rl-swap 253:1    0    1G  0 lvm   [SWAP]
sdb           8:16   0   10G  0 disk
└─mpatha    253:2    0   10G  0 mpath
  └─mpatha1 253:4    0   10G  0 part  /mnt/data_chunk1
sdc           8:32   0   10G  0 disk
└─mpathb    253:3    0   10G  0 mpath
  └─mpathb1 253:5    0   10G  0 part  /mnt/data_chunk2
sdd           8:48   0   10G  0 disk
└─mpathb    253:3    0   10G  0 mpath
  └─mpathb1 253:5    0   10G  0 part  /mnt/data_chunk2
sde           8:64   0   10G  0 disk
└─mpatha    253:2    0   10G  0 mpath
  └─mpatha1 253:4    0   10G  0 part  /mnt/data_chunk1
sr0          11:0    1 1024M  0 rom

-> pour chunk3


[root@chunk3 ~]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0   10G  0 disk
├─sda1        8:1    0    1G  0 part  /boot
└─sda2        8:2    0    9G  0 part
  ├─rl-root 253:0    0    8G  0 lvm   /
  └─rl-swap 253:1    0    1G  0 lvm   [SWAP]
sdb           8:16   0   10G  0 disk
└─mpatha    253:2    0   10G  0 mpath
  └─mpatha1 253:4    0   10G  0 part  /mnt/data_chunk1
sdc           8:32   0   10G  0 disk
└─mpathb    253:3    0   10G  0 mpath
  └─mpathb1 253:5    0   10G  0 part  /mnt/data_chunk2
sdd           8:48   0   10G  0 disk
└─mpathb    253:3    0   10G  0 mpath
  └─mpathb1 253:5    0   10G  0 part  /mnt/data_chunk2
sde           8:64   0   10G  0 disk
└─mpatha    253:2    0   10G  0 mpath
  └─mpatha1 253:4    0   10G  0 part  /mnt/data_chunk1
sr0          11:0    1 1024M  0 rom
