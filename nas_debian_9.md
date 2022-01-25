![cloud01](/images/nas_debian_9/nas_debian_9_01.jpg)


## 1 - Matériel

### 1.1 - Boitier 

InWin : [IW-MS04](https://www.in-win.com/en/ipc-server/ms04)
![NAS](/images/nas_debian_9/nas_debian_9_02.png)
![NAS](/images/nas_debian_9/nas_debian_9_03.png)

### 1.2 - Carte mère 

MSI : [MS-S0891-010](http://eps.msi.com/server/MS-S0891)
![NAS](/images/nas_debian_9/nas_debian_9_04.png)
![NAS](/images/nas_debian_9/nas_debian_9_05.png)

### 1.3 - CPU

[Intel Xeon E3-1220L v3](https://ark.intel.com/fr/products/75051/Intel-Xeon-Processor-E3-1220L-v3-4M-Cache-1_10-GHz)

### 1.4 - Refroidissement

Akasa : [AK-CC7108EP01](http://www.akasa.com.tw/search.php?seed=AK-CC7108EP01)
![NAS](/images/nas_debian_9/nas_debian_9_06.png)

### 1.5 - Accessoires divers

[Convertisseur USB2 (carte mère) vers USB3 (boitier)](https://www.amazon.fr/gp/product/B00EIE1VV6/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
![DeLOCK cable USB 3.0 Pinheader St>USB2.0 Pinheader - 30cm83281](/images/nas_debian_9/nas_debian_9_07.jpg)

### 1.6 - Mémoire

2x2Go DDR3 ECC

### 1.7 - SSD système

[Kingston SSDNow V+200 - 120 Go](https://www.kingston.com/fr/support/technical/products?model=svp200s3)
![NAS](/images/nas_debian_9/nas_debian_9_08.png)

### 1.8 - Disques durs

4x2To Western Digital Green
![NAS](/images/nas_debian_9/nas_debian_9_09.png)

10 Gb [Chelsio N320E Dual Port (cc2-n320e-sr)](https://www.chelsio.com/legacy-adapters/)
![NAS](/images/nas_debian_9/nas_debian_9_10.png)

### 1.10 - Sauvegarde locale

WD My Book Home Edition 1 TB USB 2.0/FireWire 400/eSATA Desktop External Hard Drive
![NAS](/images/nas_debian_9/nas_debian_9_11.jpg)

### 1.11 - Onduleur

[APC Back-UPS 700](https://www.apc.com/shop/fr/fr/products/APC-BACK-UPS-700VA-230V-AVR-French-Sockets/P-BX700U-FR)
![NAS](/images/nas_debian_9/nas_debian_9_12.png)
![NAS](/images/nas_debian_9/nas_debian_9_13.png)

### 1.12 - Évolutions programmées

4x6To Western Digital Red 



## Installation Debian 9

### 2.1 - ISO

Je fais une installation via une iso netinstall

Download de l’ISO : [page officiel de Debian](https://www.debian.org/CD/netinst/)

### 2.2 - Installation

![NAS](/images/nas_debian_9/nas_debian_9_13.png)
![NAS](/images/nas_debian_9/nas_debian_9_14.png)
![NAS](/images/nas_debian_9/nas_debian_9_15.png)
![NAS](/images/nas_debian_9/nas_debian_9_16.png)
![NAS](/images/nas_debian_9/nas_debian_9_17.png)
![NAS](/images/nas_debian_9/nas_debian_9_18.png)
![NAS](/images/nas_debian_9/nas_debian_9_19.png)
![NAS](/images/nas_debian_9/nas_debian_9_20.png)
![NAS](/images/nas_debian_9/nas_debian_9_21.png)
![NAS](/images/nas_debian_9/nas_debian_9_22.png)
![NAS](/images/nas_debian_9/nas_debian_9_23.png)

## 3 - Configuration post installation (root)

### 3.1 - Réseau

/etc/network/interfaces

```shell
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# 1Gb
# enp3s0 on
# enp4s0 off
allow-hotplug enp3s0
iface enp3s0 inet static
	address 192.168.1.2
	netmask 255.255.255.0
	network 192.168.1.0
	broadcast 192.168.1.255
	gateway 192.168.1.254



# Chelsio 10G
# enp1s0 off
# enp1s0d1 on
allow-hotplug enp1s0d1
iface enp1s0d1 inet static
	address 192.168.1.3
	netmask 255.255.255.0
	network 192.168.1.0
	broadcast 192.168.1.255
	gateway 192.168.1.254
```



### 3.2 - Installation des paquets utiles pour le serveur

```shell
apt-get install xfsprogs gdisk sudo htop hddtemp nmon lm-sensors mdadm smartmontools samba samba-common smbclient nfs-kernel-server locate vim python-dev python-pip cron-apt curl gnupg2 libc-ares2 libcrypto++6 libmediainfo0v5 libpcrecpp0v5 libzen0v5 apt-transport-https firmware-misc-nonfree -y
```

### 3.3 - Ajout d’un utilisateur au groupe sudo

```shell
adduser utilisateur sudo
```

### 3.4 - Authentification SSH par clef

J’utilise pour utilisateur l'authentification par clé SSH

Créer le répertoire et le fichier pour inscrire la clef ssh de l’utilisateur

```shell
mkdir /home/utilisateur/.ssh
touch /home/utilisateur/.ssh/authorized_keys
```

Ensuite, inscrire sa clé public ssh dans /home/utilisateur/.ssh/authorized_keys

### 3.5 - Mise à jour du microcode CPU

La carte mère n’a pas de BIOS/UEFI récent. Donc, va falloir mettre à jours le microcode pour Spectre et Meltdown.

Il faut télécharger sur le site d’intel l’archive qui contients les microcodes:
[https://downloadcenter.intel.com/download/27945?v=t

](https://downloadcenter.intel.com/download/27945?v=t)L’archive se nomme : microcode-20180703.tgz et sera mis dans mon dossier root.

Avant de commencer la mise à jour, voir quelle version est installé :

```shell
root@srv01-nas:~# dmesg | grep -i microcode
[    1.984753] microcode: sig=0x306c3, pf=0x2, revision=0x17
[    1.984868] microcode: Microcode Update Driver: v2.01 <tigran@aivazian.fsnet.co.uk>, Peter Oruba
root@srv01-nas:~# grep microcode /proc/cpuinfo
microcode       : 0x17
microcode       : 0x17
microcode       : 0x17
microcode       : 0x17
```

Procédons à la mise a jours, je décompresse l’archive dans un répertoire que je crée et nomme firmware. L’archive microcode-20180703.tgz est mon root.

```shell
mkdir firmware
cd firmware
tar xvf /root/microcode-20180703.tgz
```

Vérifier que le répertoire suivant existe : /sys/devices/system/cpu/microcode/reload exits

```shell
ls -l /sys/devices/system/cpu/microcode/reload
```

Copie des fichiers de microcode

```shell
cp -v intel-ucode/* /lib/firmware/intel-ucode/
```

Bla bla de la mise à jours au boot

```shell
echo 1 > /sys/devices/system/cpu/microcode/reload
```

Update de l’initamfs

```shell
update-initramfs -u
reboot
```

Vérification que la mise à jour est bien appliquée

```shell
root@srv01-nas:~# dmesg | grep -i microcode
[    0.000000] microcode: microcode updated early to revision 0x24, date = 2018-01-21
[    1.986878] microcode: sig=0x306c3, pf=0x2, revision=0x24
[    1.987022] microcode: Microcode Update Driver: v2.01 <tigran@aivazian.fsnet.co.uk>, Peter Oruba
root@srv01-nas:~# grep microcode /proc/cpuinfo
microcode       : 0x24
microcode       : 0x24
microcode       : 0x24
microcode       : 0x24
```

### 3.6 - Monitoring via Glances (facultatif)

[Glances](https://nicolargo.github.io/glances/) est un script python créé par le français [Nicolargo](https://linuxfr.org/users/nicolargo).
Installation de Glances via pip :

```shell
pip install glances
```

Pour mettre à jour Glances par la suite :

```shell
pip install -- update glances
```

Utiliser Glances

```shell
glances
```

![Glance](/images/nas_debian_9/nas_debian_9_24.png)

### 3.7 - Antivirus Clamav (facultatif)

Installation des paquets

```shell
apt-get install clamav
```

Stopper le service pour mettre a jours la base

```shell
service clamav-freshclam stop
```

Mise à jour

```shell
freshclam
ClamAV update process started at Thu Jun 28 13:02:46 2018
WARNING: Your ClamAV installation is OUTDATED!
WARNING: Local version: 0.99.4 Recommended version: 0.100.0
DON'T PANIC! Read http://www.clamav.net/documents/upgrading-clamavDownloading main.cvd [ 51%]
```

### 3.8 - Réinitialisation des disques

Dans mon cas, les disques durs ont déjà été utilisés. Le plus simple est de détruire l’ancienne table de partitionnement GPT.

Liste des disques durs

```shell
fdisk -l
```

Disque en GPT , mise à zero

```shell
gdisk /dev/sdX
```



- **x** pour le menu des fonction suplémentaire de gdisk
- **z** pour détruire l’ancienne table de partitionnement GPT
- **w** pour enregistrer les changements et quitter gdisk

### 3.9 - Préparation des disques avec une partition Linux RAID

Liste des disques durs

```shell
fdisk -l
```

Disque en GPT et partition Linux RAID

```shell
gdisk /dev/sdX
```

- **o** pour créer une nouvelle table de partition GPT puis sur Entrée
- **n** pour créer une nouvelle partition. Pour le numéro de partition, laissez par défaut
- First Sector, par défaut
- Last sector, par défaut
- code de la partition (Linux RAID) : **fd00**
- **w** pour enregistrer les changements et quitter gdisk

### 3.10 - Création du raid 10 via mdadm

Dans mon cas, je ne prépare pas les disques, ni de LVM. Je laisse mdadm ce débrouiller. Cela n’a aucun impact pour la suite, comme étendre le raid en augmentant la capacité des disques durs.

Création de la grappe raid10

```shell
mdadm --create /dev/md0 --name=raid10 --level=10 --raid-devices=4 /dev/sd[b-e]
```

Contrôle que la grappe raid est en cour de construction 

```shell
cat /proc/mdstat
```

Création du filesystem xfs

```shell
mkfs.xfs /dev/md0
```

Point de montage du raid

```shell
mkdir /media/raid10
mount /dev/md0 /media/raid10/
```

Sauvegarde la configuration raid

```shell
cp /etc/mdadm/mdadm.conf /etc/mdadm/mdadm.conf.old
mdadm --examine --scan --verbose >> /etc/mdadm/mdadm.conf
update-initramfs -u -k all
```

Montage automatique du raid au boot.
/etc/fstab

```shell
echo “/dev/md0      /media/raid10      xfs      defaults      0      0” >> /etc/fstab
```

### 3.11 - Configuration exim4

/etc/default/mdadm

```shell
START_DAEMON=true
```

/etc/mdadm/mdadm.conf

```shell
MAILADDR xxxxxxxxxx@gmail.com
```

Relancer le service mdadm pour la prise en charge

```shell
/etc/init.d/mdadm restart
```

Test

```shell
mdadm --monitor --scan --test -1
```

### 3.12 - Configuration du partage Samba

Ajout d’un utilisateur

```shell
smbpasswd -a nom_utilisateur
```

Créations des dossiers partagés

```shell
mkdir /media/raid10/films
mkdir /media/raid10/series
mkdir /media/raid10/musiques
mkdir /media/raid10/documents
mkdir /media/raid10/docs
```

Rendre accessible un répertoire en lecture écriture sans authentification

```shell
chmod 777 /media/docs
```

Pour restreindre avec des groupes
\- Créer un groupe

```shell
groupadd nom_du_groupe
```

Attribuer un répertoire a un groupe

```shell
chown -R root:nom_du_groupe /media/raid10/documents
chmod -R 770 /media/raid10/documents
```

/etc/samba/smb.conf

```shell
[global] 
	workgroup = WORKGROUP
	server string = %h server
	netbios name = NAS
	dns proxy = no


[network]
#	hosts allow = 192.168.0. 127. # ajouté et à modifier selon l'IP du réseau local
#	bind interfaces only = yes    # mettre "yes" pour utiliser le choix des interfaces (ligne au dessous)
#	interfaces = 192.168.0.15/24  # Pour éviter le wifi, on peut mettre à la place de l'IP/masque, le nom de l'interface "eth0:0"

[log]
	log file = /var/log/samba/log.%m
	max log size = 1000
	syslog = 0
	panic action = /usr/share/samba/panic-action %d

[security]
	security = user
	map to guest = Bad User
	encrypt passwords = true
	passdb backend = smbpasswd
	obey pam restrictions = no
	unix password sync = no
	passwd program = /usr/bin/passwd %u
	passwd chat = *Entersnews*spassword:* %nn *Retypesnews*spassword:* %nn *passwordsupdatedssuccessfully* .
	pam password change = yes

[documents] 
	path = /media/raid10/documents
	comment = pour gestion
	valid users = @partage
	read only = no
	browseable = yes
	writeable = yes
	create mask = 0777
	directory mask = 0777

[docs]
	path = /media/raid10/docs
	comment = Documentaires
	browseable = yes
	read only = no
	create mask = 0777
	directory mask = 0777
	valid users = nobody
	admin users = nobody
	guest ok = yes

[films]
	path = /media/raid10/films
	comment = raid 10
	browseable = yes
	read only = no
	create mask = 0777
	directory mask = 0777
	valid users = nobody
	admin users = nobody
	guest ok = yes
	
[musiques]
	path = /media/raid10/musiques
	comment = raid 10
	browseable = yes
	read only = no
	create mask = 0777
	directory mask = 0777
	valid users = nobody
	admin users = nobody
	guest ok = yes

[series]
	path = /media/raid10/series
	comment = raid 10
	browseable = yes
	read only = no
	create mask = 0777
	directory mask = 0777
	valid users = nobody
	admin users = nobody
	guest ok = yes

```

### 3.13 - Installation de Plex

Téléchargement

```shell
wget https://downloads.plex.tv/https://downloads.plex.tv/plex-media-server/1.13.3.5223-cd1e0da1b/plexmediaserver_1.13.3.5223-cd1e0da1b_amd64.deb
```

Installation

```shell
dpkg -i plexmediaserver_1.13.3.5223-cd1e0da1b_amd64.deb
```

Lancement du service

```shell
systemctl status plexmediaserver
```

Interface de management

```shell
http://ip:32400/manage
```

### 3.14 - Gestion de l’onduleur

Installation

```shell
apt-get install apcupsd
```

Voir si l'onduleur est bien détecté sur le port USB

```shell
lsusb
```

/etc/apcupsd/apcupsd.conf

```shell
UPSTYPE usb
UPSCABLE usb
DEVICE
```

/etc/default/apcupsd

```shell
ISCONFIGURED=yes
```



## 4 - Gestion du serveur

### 4.1 - Remplacer disques dur défaillant

Passer le disque dur en défaut pour le retirer

```shell
mdadm --manage /dev/md0 --set-faulty /dev/sd**X**
mdadm --manage /dev/md0 --remove /dev/sd**X**
```

Ajout du disque dur

```shell
mdadm --manage /dev/md0 --add /dev/sd**X**
```

Contrôle de la reconstruction

```shell
cat /proc/mdstat
```



### 4.2 - Étendre le RAID 10

Ajout de disques dur de plus grande capacité comme la procédure du dessus. Puis étendre le volume raid.

```shell
mdadm --grow /dev/md0 --size=max
```

Étendre le filesystem

```shell
xfs_growfs /dev/md0
```

Example :

```shell
/dev/md0:
        Version : 1.2
  Creation Time : Thu Jun 28 11:23:53 2018
     Raid Level : raid10
     Array Size : 41910272 (39.97 GiB 42.92 GB)
  Used Dev Size : 20955136 (19.98 GiB 21.46 GB)
   Raid Devices : 4
  Total Devices : 4
    Persistence : Superblock is persistent

    Update Time : Thu Jun 28 11:48:21 2018
          State : clean, degraded, recovering
 Active Devices : 2
Working Devices : 4
 Failed Devices : 0
  Spare Devices : 2

         Layout : near=2
     Chunk Size : 512K

 Rebuild Status : 1% complete

           Name : nas:0  (local to host nas)
           UUID : 17a39bf3:d2f61aad:598079f6:dfd5c6d5
         Events : 58

    Number   Major   Minor   RaidDevice State
       4       8       16        0      active sync set-A   /dev/sdb
       5       8       32        1      spare rebuilding   /dev/sdc
       6       8       48        2      spare rebuilding   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde







root@nas:~# mdadm --grow /dev/md0 --size=max
mdadm: component size of /dev/md0 has been set to 62898176K
root@nas:~# xfs_growfs /dev/md0
meta-data=/dev/md0               isize=512    agcount=16, agsize=654720 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1 spinodes=0 rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=10475520, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=5120, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 10475520 to 31449088
root@nas:~# mdadm --detail /dev/md0
/dev/md0:
        Version : 1.2
  Creation Time : Thu Jun 28 11:23:53 2018
     Raid Level : raid10
     Array Size : 125796352 (119.97 GiB 128.82 GB)
  Used Dev Size : 62898176 (59.98 GiB 64.41 GB)
   Raid Devices : 4
  Total Devices : 4
    Persistence : Superblock is persistent

    Update Time : Thu Jun 28 11:51:35 2018
          State : active, degraded, recovering
 Active Devices : 2
Working Devices : 4
 Failed Devices : 0
  Spare Devices : 2

         Layout : near=2
     Chunk Size : 512K

 Rebuild Status : 27% complete

           Name : nas:0  (local to host nas)
           UUID : 17a39bf3:d2f61aad:598079f6:dfd5c6d5
         Events : 73

    Number   Major   Minor   RaidDevice State
       4       8       16        0      active sync set-A   /dev/sdb
       5       8       32        1      spare rebuilding   /dev/sdc
       6       8       48        2      spare rebuilding   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde


```



### 4.3 - Remplacement carte mère

Contrôle de l’état des disques durs du raid installé précédemment sur une autre carte mère

```shell
mdadm --examine /dev/sd[bcde] >> raid.status
```

Création du fichier de configuration et de son initialisation

```shell
cp /etc/mdadm/mdadm.conf /etc/mdadm/mdadm.conf.old
mdadm --examine --scan --verbose >> /etc/mdadm/mdadm.conf
update-initramfs -u -k all
```

