
# III. Distributed filesystem

**On va utiliser un outil qui peut fonctionner en environnement minimaliste pour ça : MooseFS.**

**Tout fichier créé sur cette partition Moose sera répliqué par le Master au sein de nos multiples Chunks Servers.**

## 1. Master server

> ***Sur `master.tp2.b3` uniquement.***

🌞 **Installer les paquets nécessaires pour avoir un Moose Master**


[root@master ~]# curl "https://repository.moosefs.com/RPM-GPG-KEY-MooseFS" > /etc/pki/rpm-gpg/RPM-GPG-KEY-MooseFS
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1796  100  1796    0     0   7390      0 --:--:-- --:--:-- --:--:--  7360
[root@master ~]# curl "http://repository.moosefs.com/MooseFS-3-el9.repo" > /etc/yum.repos.d/MooseFS.repo
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   175  100   175    0     0   1620      0 --:--:-- --:--:-- --:--:--  1620


[root@master ~]# yum install moosefs-master moosefs-cgi moosefs-cgiserv moosefs-cli


🌞 **Démarrez les services du Moose Master**

systemctl start moosefs-cgiserv
systemctl start moosefs-master

🌞 **Ouvrez les ports firewall**


[root@master ~]# firewall-cmd --add-port=80/tcp --permanent
success
[root@master ~]# firewall-cmd --add-port=9425/tcp --permanent
success
[root@master ~]# firewall-cmd --add-port=9419/tcp --permanent
success
[root@master ~]# firewall-cmd --add-port=9420/tcp --permanent
success
[root@master ~]# firewall-cmd --add-port=9421/tcp --permanent
success

➜ **WebUI moche dispo sur le port `9425/tcp`**

## 2. Chunk servers

🌞 **Installer les paquets nécessaires pour avoir un Moose Chunk Server**

[root@master ~]# curl "https://repository.moosefs.com/RPM-GPG-KEY-MooseFS" > /etc/pki/rpm-gpg/RPM-GPG-KEY-MooseFS
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1796  100  1796    0     0   7390      0 --:--:-- --:--:-- --:--:--  7360
[root@master ~]# curl "http://repository.moosefs.com/MooseFS-3-el9.repo" > /etc/yum.repos.d/MooseFS.repo
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   175  100   175    0     0   1620      0 --:--:-- --:--:-- --:--:--  1620

[root@localhost dev]# yum install moosefs-chunkserver


🌞 **Modifier la conf du Chunk Server**

[root@chunk2 iscsi]# cat /etc/mfs/mfschunkserver.cfg | grep MASTER_HOST
MASTER_HOST = master
[root@localhost dev]# cat /etc/hosts | tail -2
10.3.250.2 master


🌞 **Faire appartenir les partitions à partager à l'utilisateur `mfs`**

[root@localhost dev]# chown mfs:mfs /mnt/data_chunk1
[root@localhost dev]# chown mfs:mfs /mnt/data_chunk2

🌞 **Modifier la conf des disques du Chunk Server**

[root@localhost dev]# cat /etc/mfs/mfshdd.cfg | tail -2
/mnt/data_chunk1
/mnt/data_chunk2


🌞 **Démarrez les services du Moose Chunk Server**

[root@chunk3 ~]# systemctl start moosefs-chunkserver

## 3. Consume

## 1. Monter la partition Moose

> ***On est sur `web.tp2.b3`.***

🌞 **Installer les paquets nécessaires pour avoir un Moose Client**

[root@web ~]# curl "https://repository.moosefs.com/RPM-GPG-KEY-MooseFS" > /etc/pki/rpm-gpg/RPM-GPG-KEY-MooseFS
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1796  100  1796    0     0   7241      0 --:--:-- --:--:-- --:--:--  7241
[root@web ~]# curl "http://repository.moosefs.com/MooseFS-4-el9.repo" > /etc/yum.repos.d/MooseFS.repo
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   175  100   175    0     0   1174      0 --:--:-- --:--:-- --:--:--  1174

[root@web ~]# yum install moosefs-client

🌞 **Monter la partition Moose**

[root@web ~]# mfsmount /mnt/www -H master
mfsmaster accepted connection with parameters: read-write,restricted_ip,admin ; root mapped to root:root


🌞 **Preuve et test d'écriture**



[root@web www]# df -h | grep /mnt/www
mfs#master:9421       60G  2.2G   58G   4% /mnt/www
[root@web mnt]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   10G  0 disk
├─sda1        8:1    0    1G  0 part /boot
└─sda2        8:2    0    9G  0 part
  ├─rl-root 253:0    0    8G  0 lvm  /
  └─rl-swap 253:1    0    1G  0 lvm  [SWAP]
sr0          11:0    1 1024M  0 rom
[root@web mnt]# cd /mnt/www/
[root@web www]# touch toto
[root@web www]# ll
total 0
-rw-r--r--. 1 root root 0 Dec  6 16:17 toto


## 2. NGINX

🌞 **Installer et configurer NGINX sur la machine `web.tp2.b3`**

[root@web www]# dnf install nginx
[...]
[root@web www]# firewall-cmd --add-port=80/tcp --permanent
success
[root@web www]# systemctl enable nginx

[root@web www]# cat /etc/nginx/conf.d/new.conf
server {
    listen 80;
    server_name web.tp2.b3;

    root /mnt/www;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

🌞 **Démarrer le service NGINX**

systemctl start nginx

🌞 **Prouvez avec un `curl` que le site est actif**

[root@master ~]# curl 10.3.250.101
skafnefnfejfezjfezi

**Well that's a highly-available little storage setup !** :d
