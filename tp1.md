# TP1 : Single-machine storage

Dans ce TP on revoit **quelques fondamentaux** et on setup deux trois trucs rigolos, en utilisant une seule machine : du **RAID logiciel** et un serveur de **partage de fichiers réseau**.

![Look !](./img/free.jpg)

## Sommaire

- [TP1 : Single-machine storage](#tp1--single-machine-storage)
  - [Sommaire](#sommaire)
- [0. Prérequis](#0-prérequis)
- [I. Fundamentals](#i-fundamentals)
- [II. Partitioning](#ii-partitioning)
- [III. RAID](#iii-raid)
  - [1. Simple RAID](#1-simple-raid)
  - [2. Break it](#2-break-it)
  - [3. Spare disk](#3-spare-disk)
  - [4. Grow](#4-grow)
- [IV. NFS](#iv-nfs)

# 0. Prérequis

➜ **une VM avec l'OS Linux de ton choix**, quelques contraintes :

- ajouter six disques durs supplémentaires de 10G à la VM
- système à jour
- firewall activé
- la VM a un accès internet
- attribuez le hostname `storage.b3` à la VM
- uniquement du SSH pour avoir un shell sur la VM

> Je conseille une distrib orientée serveur au moins. Personal recommandation : Rocky Linux ou Alma Linux.

# I. Fundamentals

🌞 **Lister tous les périphériques de stockage branchés à la VM**
🌞 **Lister toutes les partitions des périphériques de stockage**

- uniquement les périphériques de stockage :

```bash
[dreams@localhost ~]$ lsblk

NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   10G  0 disk
sdb           8:16   0   10G  0 disk
sdc           8:32   0   10G  0 disk
├─sdc1        8:33   0    1G  0 part /boot
└─sdc2        8:34   0    9G  0 part
  ├─rl-root 253:0    0    8G  0 lvm  /
  └─rl-swap 253:1    0    1G  0 lvm  [SWAP]
sdd           8:48   0   10G  0 disk
sde           8:64   0   10G  0 disk
sdf           8:80   0   10G  0 disk
sdg           8:96   0   10G  0 disk
sr0          11:0    1  1.5G  0 rom
```


🌞 **Effectuer un test SMART sur le disque**

- avec `smartctl` : 

```bash
[dreams@localhost ~]$ sudo smartctl -a /dev/sdc

smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.14.0-427.42.1.el9_4.x86_64] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Device Model:     VBOX HARDDISK
Serial Number:    VB2bc03f31-f3c4883e
Firmware Version: 1.0
User Capacity:    10,762,977,280 bytes [10.7 GB]
Sector Size:      512 bytes logical/physical
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ATA/ATAPI-6 published, ANSI INCITS 361-2002
Local Time is:    Thu Nov 14 11:47:55 2024 CET
SMART support is: Unavailable - device lacks SMART capability.

A mandatory SMART command failed: exiting. To continue, add one or more '-T permissive' options.
```

🌞 **Espace disque...**

- afficher l'espace disque restant sur la partition `/` :

```bash
[dreams@localhost ~]$ df -h /

Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/rl-root  8.1G  1.5G  6.6G  19% /
```


🌞 **Inodes**

- afficher la quantité d'inodes restants et la quantité d'inodes utilisés :

````bash
[dreams@localhost ~]$ df -hi /

Filesystem          Inodes IUsed IFree IUse% Mounted on
/dev/mapper/rl-root   4.1M   37K  4.0M    1% /
```

🌞 **Latence disque**

- utilisez `ioping` pour déterminer la latence disque :

```bash 
[dreams@localhost ~]$ ioping /dev/sdc

ioping: failed to open "/dev/sdc": Permission denied
[dreams@localhost ~]$ sudo ioping /dev/sdc
4 KiB <<< /dev/sdc (block device 10.0 GiB): request=1 time=261.8 us (warmup)
4 KiB <<< /dev/sdc (block device 10.0 GiB): request=2 time=3.52 ms
4 KiB <<< /dev/sdc (block device 10.0 GiB): request=3 time=725.7 us
4 KiB <<< /dev/sdc (block device 10.0 GiB): request=4 time=2.24 ms
4 KiB <<< /dev/sdc (block device 10.0 GiB): request=5 time=913.7 us
^C
--- /dev/sdc (block device 10.0 GiB) ioping statistics ---
4 requests completed in 7.40 ms, 16 KiB read, 540 iops, 2.11 MiB/s
generated 5 requests in 4.64 s, 20 KiB, 1 iops, 4.31 KiB/s
min/avg/max/mdev = 725.7 us / 1.85 ms / 3.52 ms / 1.13 ms
```

🌞 **Déterminer la taille du cache *filesystem***

```bash
[dreams@localhost ~]$ cat /proc/meminfo | grep -i 'cache'

Cached:           345260 kB
```

# II. Partitioning

Ici on utilise un des disques supplémentaires branchés à la VM : `sdb`.

> Assurez-vous que LVM est installé sur votre OS avant de continuer.

🌞 **Ajouter `sdb` comme Physical Volume LVM**

```bash
[dreams@localhost mnt]$ sudo pvcreate /dev/sdb
WARNING: dos signature detected on /dev/sdb at offset 510. Wipe it? [y/n]: y
  Wiping dos signature on /dev/sdb.
  WARNING: adding device /dev/sdb with idname t10.ATA_VBOX_HARDDISK_VB3e3120ca-f1ddc01e which is already used for missing device.
  Physical volume "/dev/sdb" successfully created.
```

🌞 **Créer un Volume Group LVM nommé `storage`**

```bash
[dreams@localhost ~]$ sudo vgcreate storage /dev/sdb
  Volume group "storage" successfully created
```

🌞 **Créer un Logical Volume LVM**

- dans le VG `storage`
- nommé `smol_data`
- sa taille : 2G


```bash
[dreams@localhost ~]$ sudo lvcreate -L 2G -n smol_data storage
  Logical volume "smol_data" created.
```

🌞 **Créer un deuxième Logical Volume LVM**

- dans le VG `storage`
- nommé `big_data`
- sa taille : tout le reste du VG

```
[dreams@localhost ~]$ sudo lvcreate -l 100%FREE -n big_data storage
  Logical volume "big_data" created.
```

🌞 **Créez un système de fichiers sur les deux LVs**

- sur `smol_data` et `big_data`
- utilisez ext4

```
[dreams@localhost ~]$ sudo mkfs.ext4 /dev/storage/smol_data
[...]

[dreams@localhost ~]$ sudo mkfs.ext4 /dev/storage/big_data
[...]  
```

🌞 **Montez la partition**

- sur le point de montage `/mnt/lvm_storage`
- il faudra le créer avant

🌞 **Configurer un *automount***

```
[dreams@localhost lvm_storage]$  sudo cat /etc/systemd/system/mnt-lvm_storage.mount
[Unit]
Description=Mount LVM Storage
After=local-fs.target

[Mount]
What=/dev/mapper/storage-big_data
Where=/mnt/lvm_storage
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target
[dreams@localhost lvm_storage]$  sudo cat /etc/systemd/system/mnt-lvm_storage.automount
[Unit]
Description=Automount LVM Storage on Demand
After=local-fs.target

[Automount]
Where=/mnt/lvm_storage
TimeoutIdleSec=5min

[Install]
WantedBy=multi-user.target
```

🌞 **Prouvez que l'*automount* est effectif**

```
[dreams@localhost lvm_storage]$ systemctl list-units --type automount
  UNIT                              LOAD   ACTIVE SUB     DESCRIPTION
  mnt-lvm_storage.automount         loaded active running Automount LVM Storage on Demand
```

# III. RAID

Dans cette section, vous allez vous servir des disques supplémentaires ajoutés à la VM.

Installez `mdadm` sur votre VM, on va avoir besoin de lui pour mettre en place un RAID logiciel.

Ici, pas de carte RAID physique (ou virtuelle...) qui gère un RAID bas niveau, c'est un RAID géré par un programme une fois l'OS lancé. L'OS a donc la visibilité sur les disques sous-jacents.

> En effet, dans le cas d'un RAID logiciel, l'OS ne voit qu'un disque unique, qui est en réalité le RAID mis en place par la carte RAID physique.

## 1. Simple RAID

🌞 **Mettre en place un RAID 5**

- avec TROIS disques parmis les disques supplémentaires branchés à la VM : `sdc` `sdd` et `sde`
- en utilisant `mdadm` :

```
[dreams@localhost lvm_storage]$ sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdd /dev/sde /dev/sdf
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 10476544K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

🌞 **Prouvez que le RAID5 est en place**, on doit voir :

```
[dreams@localhost lvm_storage]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sdf[3] sde[1] sdd[0]
      20953088 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

unused devices: <none>
```

🌞 **Rendre la configuration automatique au boot de la machine**

```
sudo mdadm --detail --scan --verbose >> /etc/mdadm/mdadm.conf
```

🌞 **Créez un système de fichiers sur la partition proposé par le RAID**

- utilisez ext4 (on verra ptet un peu de zfs une prochaine fois) : 

```
sudo mkfs.ext4 /dev/md0
```

🌞 **Monter la partition sur `/mnt/raid_storage`**

- il faudra créer le point de montage `/mnt/raid_storage` au préalable
- puis une commande pour monter la partition

🌞 **Prouvez que...**

- la partition est bien montée : 

```
[dreams@localhost ~]$ lsblk | grep raid
└─md0                 9:0    0   20G  0 raid5 /mnt/raid_storage
└─md0                 9:0    0   20G  0 raid5 /mnt/raid_storage
└─md0                 9:0    0   20G  0 raid5 /mnt/raid_storage
```

- il y a bien l'espace disponible attendue sur la partition :

```
[dreams@localhost ~]$ df -h /mnt/raid_storage/
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         20G   24K   19G   1% /mnt/raid_storage
```

- vous pouvez lire et écrire sur la partition : 

```
[dreams@localhost raid_storage]$ sudo touch toto
[sudo] password for dreams:
[dreams@localhost raid_storage]$ ls
lost+found  toto
```

> *Alors combien de Go dispo avec un RAID5 sur des disques de 10G ?*

5GO

🌞 **Mini benchmark**

- faites un test de vitesse d'écriture sur la partition `mdadm` montée : `/mnt/raid_storage` :

```
[dreams@localhost raid_storage]$ sudo dd if=/dev/zero of=/mnt/raid_storage/testfile bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 3.65768 s, 294 MB/s
```

- idem sur la partition créée avec LVM à la partie précédente : `/mnt/lvm_storage` :

```
[dreams@localhost raid_storage]$ sudo dd if=/dev/zero of=/mnt/lvm_storage/testfile bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.917457 s, 1.2 GB/s
```

> Bon on est en environnement virtuel VirtualBox, avec des PC portables et des OS pas toujours stables, alors à prendre avec des pincettes le résultat.

## 2. Break it

![Gone](./img/sto.jpg)

🌞 **Simule une panne**

- débranche un disque du RAID ! :

```
[dreams@localhost ~]$ lsblk |  grep md0
└─md0                 9:0    0   20G  0 raid5
└─md0                 9:0    0   20G  0 raid5
```

🌞 **Montre l'état du RAID dégradé**

- avec une commande, on doit voir que le RAID est dégradé :

```
[dreams@localhost ~]$ sudo mdadm --monitor /dev/md0
mdadm: Value "localhost.localdomain:0" cannot be set as name. Reason: Not POSIX compatible. Value ignored.
mdadm: DegradedArray event detected on md device /dev/md0
mdadm: SparesMissing event detected on md device /dev/md0
```

🌞 **Remonte le disque dur**

- il faut retrouver un RAID5 fonctionnel
- avec un `cat /proc/mdstat` on peut voir si une recovery est en cours
- prouvez que le RAID est de nouveau fonctionnel

## 3. Spare disk

Un disque en *spare* c'est un terme qu'on utilise pour désigner un disque qui ne fait rien, mais est prêt à prendre la place d'un disque défaillant.

🌞 **Ajoutez un disque encore inutilisé au RAID5 comme disque de *spare***


```
[dreams@localhost ~]$ sudo mdadm --manage /dev/md0 --add-spare /dev/sde
[sudo] password for dreams:
mdadm: Value "localhost.localdomain:0" cannot be set as name. Reason: Not POSIX compatible. Value ignored.
mdadm: added /dev/sde
[dreams@localhost ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sde[4](S) sdb[0] sdc[1] sdd[3]
      20953088 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

unused devices: <none>
```

🌞 **Simuler une panne**

- débrancher un disque du RAID
- observer le comportement, la recovery
- déterminer le temps entre le début de la panne, et le RAID de nouveau complètement opérationnel

🌞 **Remonter le disque débranché**

- débrouillez-vous, on veut retrouver l'état initial : RAID5 avec un quatrième disque en *spare*

## 4. Grow

🌞 **Ajoutez un disque encore inutilisé au RAID5 comme disque de *spare***

- should be `sdg`
- prouvez que vous avez bien un nouveau disque en *spare*
- RAID5 avec 2 disques *spare* pour le moment !

🌞 **Grow !**

- agrandissez le RAID5 pour utiliser désormais 4 disques actifs dans le RAID
- on garde donc un seul *spare*

🌞 **Prouvez que le RAID5 propose désormais 4 disques actifs**

- l'espace proposé devrait aussi avoir grandi
- alors combien d'espace sur un RAID5 de 4 disques ?

🌞 **Euuuh wait a sec... `/mnt/raid_storage` ???**

- la partition montée sur `/mnt/raid_storage` fait toujours l'ancienne taille
- prouvez que la partition `/mnt/raid_storage` n'a pas changé de taille
- il faut indiquer au filesystem ext4 qu'il doit rescanner la partition pour s'adapter à la nouvelle taille
- prouvez que la partition `/mnt/raid_storage` a bien la nouvelle taille après votre opération

# IV. NFS

Enfin, on clôt le TP avec un **partage réseau simple à setup et relativement efficace : NFS.** Il est assez répandu dans le monde opensource pour des charges faibles.

C'est **un modèle de client/serveur** : on installe un serveur NFS sur une machine, on le configure pour indiquer quel(s) dossier(s) on veut partager sur le réseau, et d'autres machines peuvent s'y connecter pour accéder à ces dossiers.

🌞 **Installer un serveur NFS**

- il devra exposer les deux points de montage créés des parties précédentes : `/mnt/raid_storage` et `/mnt/lvm_storage`
- seul le réseau local du serveur NFS doit pouvoir y accéder
- il sera nécessaire de configurer le firewall de la machine

🌞 **Pop une deuxième VM en vif**

- installer un client NFS pour se connecter au serveur
- pour `/mnt/raid_storage`
  - monter le dossier partagé `/mnt/raid_storage` du serveur
  - sur le point de montage que vous aurez créé sur le client `/mnt/raid_storage`
  - (même chemin aussi sur le client)
- et idem : partage `/mnt/lvm_storage` du serveur monté sur `/mnt/lvm_storage` côté client

🌞 **Benchmarkz**

- faites un test de vitesse d'écriture sur la partition `mdadm` montée en NFS : `/mnt/raid_storage`
- faites un test de vitesse d'écriture sur la partition LVM montée en NFS : `/mnt/lvm_storage`

> Là même avec l'environnement qui n'est pas idéal, la différence devrait être visible. Because réseau.

![Network](./img/network.webp)