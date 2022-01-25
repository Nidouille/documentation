Je vois sur Internet des serveurs Proxmox pas sécurisés voir pire, des conseils qui préconisent d'utiliser un firewall comme pfSense installé en machine virtuelle pour sécuriser l'hyperviseur alors que ce dernier est hébergé chez un fournisseur dont on ne sait rien du réseau. Votre fournisseur peut proposer un firewall comme chez OVH avec son système [anti-DDoS](https://www.ovh.com/fr/anti-ddos/), mais cela ne vous protège que de l'extérieur pas de l'intérieur du réseau qui comporte une multitude de serveurs d'autre client. 

La logique pour un réseau en entreprise ou chez soit pour un home lab ou de l'auto hébergement sera la même. 

**Quel que soit l'hyperviseur, il faut que celui-ci soit sécurisé. S'il n'est pas sécurisé, toutes vos machines virtuelles ou conteneurs sont potentiellement compromis.**


## Firewall

Proxmox intègre un firewall agissant sur 3 parties distinctes :

- Datancenter
- Serveur Proxmox alias pve
- Machines virtuelles et conteneurs LXC

La partie machine virtuelle et conteneur est indépendante des deux autres. Elle n'a pas intérêt dans la sécurisation du serveur Proxmox.

Je me base que pour un seul hot Proxmox, qui dans ce cas le fait de faire les règles au niveau du data center ou du node n'aura pas forcément d'impact.


### Alias

Alias se trouve dans la partie firewall du Datacenter et va permettre de nommer les IP ou les plages d'IP a utiliser dans le firewall.

C'est une habitude de travail de créer des alias, ça évite les oublies, ça permet aussi d'aller plus vite quand on a une correction à effectuer sur une grosse quantité de règles de filtrage.

![](/images/Proxmox_secu/01.png)
#### Plage d'IP

![](/images/Proxmox_secu/02.png)
#### Une IP

![](/images/Proxmox_secu/03.png)


### Règles de firewall

Rien de compliqué, on autorise en entrée le port 8006, le SSH, le ping. Et comme c'est qu'un seul node, pas besoin de pressiser la destination (pas idéal, mais cela simplifie les choses) ni l'interface sur laquelle le trafic doit passer qui de toute façons pour un seul node sera vmbr0.

| Direction | Action | Source | Protocol | Destination Port | Log Level | Comment |
| :-------- | :----- | :----- | :------- | :--------------- | :-------- | :------ |
| in        | ALLOW  |        | tcp      | 8006             | nolog     | web GUI |
| in        | ALLOW  |        | tcp      | 22               | nolog     | ssh     |
| in        | ALLOW  |        | icmp     |                  | nolog     | ping    |

#### Macro

Il est possible d'utiliser des macro de configuration pour certains protocoles comme le SSH ou le protocole SMB qui a besoin d'ouverture de plusieurs ports (TCP 445, TCP 139, UDP 138-139), cela facilite grandement la lecture des règles si vous devez l'utiliser.

![](/images/Proxmox_secu/04.png)

#### Protocol

![](/images/Proxmox_secu/05.png)

### Chef je me suis coupé la main

En console, il est possible de désactiver le firewall

Editez le fichier */etc/pve/firewall/cluster.fw* et remplacez la valeur **1** par **0**.

```shell
[OPTIONS]

enable: 1
```



## Fail2Ban

### Installation de Fail2Ban

```shell
apt install fail2ban
```

### Configuration de fail2Ban 

Editez le fichier :/etc/fail2ban/jail.local

```shell
[proxmox]
enabled = true
port = https,http,8006
filter = proxmox
logpath = /var/log/daemon.log
maxretry = 3
bantime = 3600 #1 heure

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
findtime = 300
bantime = 86400 #24 heures
ignoreip = 127.0.0.1
```

/etc/fail2ban/filter.d/proxmox.conf

```shell
[Definition]
failregex = pvedaemon\[.*authentication failure; rhost=<HOST> user=.* msg=.*
ignoreregex =
```

Relancer le service de Fail2Ban

```shell
systemctl restart fail2ban.service
```

Sources : [wiki Proxmox](https://pve.proxmox.com/wiki/Fail2ban)

### Les commandes utiles de Fail2Ban

#### Bannir une IP

```shell
fail2ban-client set [nom du jail] banip [IP à bannir]
```

#### 3.3.2 Enlever le ban d'une IP

```shell
fail2ban-client set [nom du jail] unbanip [IP concerné]
```

#### Lister les règles

```shell
fail2ban-client status

Status
|- Number of jail:      1
`- Jail list:   sshd
```

#### Détails d'une règle

```shell
fail2ban-client status sshd

Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     5
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   192.168.1.21
```

 Et si l'on veut en savoir plus sur les tentatives de connexion, il faut regarder dans /var/log/auth.log

```shell
tail /var/log/auth.log

Dec  9 12:46:14 pve sshd[3769206]: Failed password for nidouille from 192.168.1.21 port 39516 ssh2
Dec  9 12:46:18 pve sshd[3769206]: Failed password for nidouille from 192.168.1.21 port 39516 ssh2
Dec  9 12:46:22 pve sshd[3769206]: Failed password for nidouille from 192.168.1.21 port 39516 ssh2
Dec  9 12:46:23 pve sshd[3769206]: Connection closed by authenticating user nidouille 192.168.1.21 port 39516 [preauth]
Dec  9 12:46:23 pve sshd[3769206]: PAM 2 more authentication failures; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.1.21  user=nidouille
```



## SSH

Par défaut Proxmox ne propose qu'un compte utilisateur : root. On va le sécuriser à minima pour les connexions SSH.

Voici les options d'activées après une installation d'un node dans /etc/ssh/sshd_config et ce n'est pas folichon.

```shell
PermitRootLogin yes

# Change to yes to enable challenge-response passwords (beware issues with
# some PAM modules and threads)
ChallengeResponseAuthentication no

UsePAM yes
X11Forwarding yes
PrintMotd no

# Allow client to pass locale environment variables
AcceptEnv LANG LC_*

# override default of no subsystems
Subsystem       sftp    /usr/lib/openssh/sftp-server
```

Fail2Ban amène une protection pour le brut force mais si on garde l'utilisateur root pour accès distant en ssh, on va monté d'un cran la sécurité en obligeant la connexion via clefs. Je ne serais que trop conseiller la désactivation de l'utilisateur SSH au profit d'un autre compte système.

Je part du principe que vous avez vos clefs SSH privés et publique.

Dans /root/.ssh/authorized_keys vous allez renseigner votre clef publique

Puis modifier /etc/ssh/sshd_config pour forcer authentification par clefs.

```shell
#PermitRootLogin yes
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPassword no

# Change to yes to enable challenge-response passwords (beware issues with
# some PAM modules and threads)
ChallengeResponseAuthentication no

UsePAM yes
X11Forwarding no
PrintMotd no

# Allow client to pass locale environment variables
AcceptEnv LANG LC_*

# override default of no subsystems
Subsystem       sftp    /usr/lib/openssh/sftp-server
```


## Authentification à double facteurs TOTP

Pour la mise en place rapide d'une double authentification, la solution du TOTP est idéal. Il faut juste un gestionnaire de mot de passe qui possède cette fonctionnalité comme Bitwarden, LastPass via son application dédié Authentificator, NordPass, etc., ou des applications dédiées comme LastPass Authentificator, etc.

Pour en savoir plus sur le TOTP: 

- site de la société française Synestis spécialisé en cybersécurité :  [lien](https://www.synetis.com/authentification-totp/)
- blog de l'hébergeur IONOS  : [lien](https://www.ionos.fr/digitalguide/serveur/securite/totp/)  

Pour le tutoriel, j'utilise Bitwarden (produit très utilisé) pour la génération du code aléatoire. Mais il n'est pas possible sur l'application d'effectuer des captures d'écran, les captures faite le sont sur mon client Bitwarden installé sur mon PC.

**Note importante : ne pas utiliser le même logiciel pour le TOTP et les mots de passe !**

![](/images/Proxmox_secu/06.png)

Ouvrir Bitwarden sur votre téléphone

Créer un nouvel élément de type identifiant

- Nom pour Bitwarden
- Nom d'utilisateur
- Le mot de passe de l'utilisateur (facultatif)
- Clé authentification (TOTP) : prenez en photo le QR code généré et cela remplira la ligne

![](/images/Proxmox_secu/07.png)
Sauvegarder identifiant créé et ensuite ouvrez le. Vous verrez un code généré avec un temps avant la génération d'un nouveau code (30 secondes).

![](/images/Proxmox_secu/08.png)
Rentrer le code dans la boite de dialogue de création de compte TOTP de Proxmox pour activer le TOTP du compte.

![](/images/Proxmox_secu/09.png)

![](/images/Proxmox_secu/10.png)
![](/images/Proxmox_secu/11.png)