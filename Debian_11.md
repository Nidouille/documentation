# Introduction

Pour faire suite à mon ancien billet de blog sur la création d'un [NAS sous Debian 9](https://blog.labperso.ovh/post/nas_debian_9/), vu que l'on est maintenant sous Debian 11, pas mal d'évolution sur installation du système d'exploitation. Voici un mix de ce que je fais entre le personnel et le travail lors d'une installation de Debian. Au travail, c'est une logique installation dite durci.
Dans le cas présent, comme c'est chez moi, je me laisse l'accès à l'utilisateur root et pas de sudo.

# Partitionnement

Suivant mes besoins, je ne partitionne pas de la même manière. Il est souvent conseillé de découper au maximum, mais je trouve que cela augmente la perte de disque exploitable. Mais une partition est forcément isolée, **/boot**.

Suivant les utilisations, je crée :

- var/log : pour éviter que des logs trop verbeux viennent à remplir le stockage
- /var/lib/mysql : isoler la base de données du reste du système 
- /var/lib/postgresql : isoler la base de données du reste du système 
- /var/www : isoler la partie publiée par le serveur html
- /home : si pas d'utilisateur, cela permet de configurer la partition en lecture seule

| Point de montage | Options                    | Notes                              |
| ---------------- | -------------------------- | ---------------------------------- |
| /boot            | nosuid,nodev,noexec,noauto | Partition à protéger               |
| /var/log         | nosuid,nodev,noexec        | Logs du système                    |
| /home            | nosuid,nodev,noexec        | En cas de non utilisation de /home |

- nosuid : ignore les bits setuid et setgid
- nodev : ignore les fichiers de périphériques
- noexec : empêche les binaires d'être exécutés
- noauto : ne monte pas la partition
- nouser : seul root pourra monter la partition

Pour plus d'information : https://wiki.debian.org/fr/fstab

# Installation de Debian 11

## Partitionnement

Les tailles pour /var/log, /home et swap sont indicatif.
Car dans mon cas /home sera en lecture seule et que la taille peut être petite.

### BIOS

| Point de montage | LVM  | Taille | Système de fichier | flag boot |          Options           |
| :--------------: | :--: | :----: | :----------------: | :-------: | :------------------------: |
|      /boot       |  -   | 512 Mo |        ext4        |     O     | nosuid,nodev,noexec,noauto |
|        /         |  O   |        |                    |     -     |          defaults          |
|     /var/log     |  O   |  4 Go  |        ext4        |     -     |    nosuid,nodev,noexec     |
|      /home       |  O   | 100 Mo |        ext4        |     -     |    nosuid,nodev,noexec     |
|       swap       |  O   |  1 Go  |         -          |     -     |             -              |

### UEFI

| Point de montage | LVM  | Taille | Système de fichier | flag boot |          Options           |
| :--------------: | :--: | :----: | :----------------: | :-------: | :------------------------: |
|    /boot/efi     |  -   | 512 Mo |       FAT32        |     O     |         umask=0077         |
|      /boot       |  -   | 512 Mo |        ext4        |     -     | nosuid,nodev,noexec,noauto |
|        /         |  O   |        |                    |     -     |          defaults          |
|     /var/log     |  O   |  4 Go  |        ext4        |     -     |    nosuid,nodev,noexec     |
|      /home       |  O   | 512 Mo |        ext4        |     -     |    nosuid,nodev,noexec     |
|       swap       |  O   |  1 Go  |         -          |     -     |             -              |

![](/images/debian_11/01.png)



## Logiciels

C'est simple, on installe rien, cela sera fait par la suite..

![](/images/debian_11/02.png)

# Post installation

## OpenSSH

### Installation du server OpenSSH

```shell
apt install openssh-server
```

### Configuration du serveur

Changement du port par défaut : Port XX

```
Port 9227
```

Forcer le protocol ssh en V2

```shell
Protocol 2
```

Activer les log

```shell
SyslogFacility AUTH
LogLevel INFO
```

Autoriser la connexion par clefs publique

```shell
PubkeyAuthentication yes
```

Limiter le nombre de tentatives de mot de passe : MaxAuthTries X


```shell
MaxAuthTries 2
```

Désactiver le forwarding

```shell
AllowTcpForwarding no
X11Forwarding no
```

## Firewall

Pour le firewall, j'utilise UFW.

```shell
apt install ufw
```

- Autoriser le ssh

```shell
ufw allow port_ssh/tcp
```

ou

```shell
ufw allow proto tcp from 192.168.1.0/24 port port_ssh
```

- Refuser les connexions entrantes

```shell
ufw default deny
```

- Activer la journalisation

```shell
ufw logging on
```

- Activer le firewall

```shell
ufw enable
```

- Stratus du firewall

```Shell
ufw status
```

- Désactiver le firewall

```shell
ufw disable
```

- Afficher les règles du firewall

```shell
ufw status verbose
```

## Mises a jours automatique

Comme je suis faignante à la maison, j'automatise les mises à jour critique du système via le paquet unattended-upgrades.

##### Installation

```shell
apt install unattended-upgrades apt-listchanges
```

##### /etc/apt/apt.conf.d/50unattended-upgrades

```shell
Unattended-Upgrade::Origins-Pattern {
        // Codename based matching:
        // This will follow the migration of a release through different
        // archives (e.g. from testing to stable and later oldstable).
        // Software will be the latest available for the named release,
        // but the Debian release itself will not be automatically upgraded.
//      "origin=Debian,codename=${distro_codename}-updates";
//      "origin=Debian,codename=${distro_codename}-proposed-updates";
        "origin=Debian,codename=${distro_codename},label=Debian";
        "origin=Debian,codename=${distro_codename},label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";

        // Archive or Suite based matching:
        // Note that this will silently match a different release after
        // migration to the specified archive (e.g. testing becomes the
        // new stable).
//      "o=Debian,a=stable";
//      "o=Debian,a=stable-updates";
//      "o=Debian,a=proposed-updates";
//      "o=Debian Backports,a=${distro_codename}-backports,l=Debian Backports";
};
```

Par défaut ce concentre que sur le coté sécurité

La section **Unattended-Upgrade::Package-Blacklist** est intéressante pour ne pas mettre à jour certains paquets

```shell
// The following matches all packages starting with linux-
//  "linux-";

    // Use $ to explicitely define the end of a package name. Without
    // the $, "libc6" would match all of them.
//  "libc6$";
//  "libc6-dev$";
//  "libc6-i686$";

    // Special characters need escaping
//  "libstdc\+\+6$";

    // The following matches packages like xen-system-amd64, xen-utils-4.1,
    // xenstore-utils and libxenstore3.0
//  "(lib)?xen(store)?";

    // For more information about Python regular expressions, see
    // https://docs.python.org/3/howto/regex.html
};
```

Les notifications

```shell
// Send email to this address for problems or packages upgrades
// If empty or unset then no email is sent, make sure that you
// have a working mail setup on your system. A package that provides
// 'mailx' must be installed. E.g. "user@example.com"
Unattended-Upgrade::Mail "root";

// Set this value to one of:
//    "always", "only-on-error" or "on-change"
// If this is not set, then any legacy MailOnlyOnError (boolean) value
// is used to chose between "only-on-error" and "on-change"
Unattended-Upgrade::MailReport "only-on-error";
```

Dans mon cas, c'est sur l'utilisateur root que seront reçus les mails de notification et uniquement pour les erreurs.

Dans le cadre professionnel, j'utilise un e-mail dédié et le mail sera avec l'option *on-change* pour garder l'historique. Et il faut bien sûr avoir de configuré un serveur de messagerie comme exim et d'avoir installé le paquet *bsd-mailx* ou *mailutils*

##### /etc/apt/apt.conf.d/20auto-upgrades

```shell
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "30";
```

- Update-Package-Lists: 1 activer l'auto-update, 0 désactiver
- Unattended-Upgrade: 1 activer l'auto-upgrade, 0 désactiver
- Download-Upgradeable-Packages : Définit la fréquente en jours pour le téléchargement des paquets
- AutocleanInterval:  Active le nettoyage automatique des paquets pendant X jours. Cette configuration affiche 30 jours

##### Tester la configuration

```shell
unattended-upgrades --debug --dry-run 
```

## Kernel pour les VM

Si on a la flemme de compiler son kernel, debian fournit un kernel pour les environnements virtualisés qui enlève pas mal de module n'ayant aucun intérêt.

```shell
apt install linux-image-cloud-amd64
```

## sysctl 

Pour cette section qui est plus d'ordre informatif, je fais référence au guide de configuration de l'[ANSSI](https://www.ssi.gouv.fr/uploads/2016/01/linux_configuration-fr-v1.2.pdf) (page 21 du document), document qui est utilisé comme référence (tout comme ceux du [CIS](https://www.cisecurity.org) ou du [CISA](https://www.cisa.gov)) dans mon travail. 

### IP Forwarding

Je désactive la possibilité de faire transiter des paquets IPv4 d'une interface à une autre. A ne pas faire sur une installation pour un routeur ;)

- Méthode temporaire : via systemctl 

```shell
sysctl -w net.ipv4.ip_forward = 0
```

- Méthode 1 : /etc/sysctl.conf

```shell
net.ipv4.ip_forward = 0
```

Pour que les modifications soient prisent en charges

```shell
sysctl -p
```

- Méthode 2 : /etc/sysctl.d/60-disable-ipv4-forward.conf

```shell
net.ipv4.ip_forward = 0
```

Pour que les modifications soient prisent en charges

```shell
sysctl -p -f /etc/sysctl.d/60-disable-ipv4-forward.conf
```

### Désactivation de l'IPv6

Je vais me faire réprimander par les défenseurs de cette technologie, mais si le réseau sur lequel vous avez vos serveurs n'est que IPv4, on peut désactiver le protocole IPv6.

- Méthode temporaire : via systemctl 

```shell
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
```

- Méthode 1 : /etc/sysctl.conf

```shell
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1
```

Si ca ne concerne qu'une seule interface réseau

```shell
net.ipv6.conf.enp0s3.disable_ipv6 = 1
```

Pour que les modifications soient prisent en charges

```shell
sysctl -p
```

- Méthode 2 : /etc/sysctl.d/70-disable-ipv6.conf

Toute les interfaces

```shell
net.ipv6.conf.all.disable_ipv6 = 1
```

Une interface en particulier

```shell
net.ipv6.conf.enp0s3.disable_ipv6 = 1
```

Pour que les modifications soient prisent en charges

```shell
sysctl -p -f /etc/sysctl.d/70-disable-ipv6.conf
```

### Restreindre accès au buffer de dmesg

Seul un utilisateur privilégié pourra accéder a certaines informations collecter par dsmeg.

```shell
kernel.dmesg_restrict = 1
```

