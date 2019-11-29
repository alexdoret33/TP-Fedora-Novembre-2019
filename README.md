## 1. First steps

* 🌞 s'assurer que `systemd` est PID1

`ps -ef`

* 🌞 check tous les autres processus système (**PAS les processus kernel)
  * décrire brièvement au moins 5 autres processus système
  
`ps
polkitd      874  0.0  0.2 1542116 21480 ?       Ssl  15:26   0:00 /usr/lib/polkit-1/polkitd --no-debug
dbus         772  0.0  0.0 277380  4672 ?        Ss   15:26   0:00 /usr/bin/dbus-broker-launch --scope system --audit
chrony       750  0.0  0.0  86996  2844 ?        S    15:26   0:00 /usr/sbin/chronyd
root         785  0.0  0.2 473940 20244 ?        Ssl  15:26   0:00 /usr/sbin/NetworkManager --no-daemon
root         712  0.0  0.0   6400  3168 ?        S<   15:26   0:00 /usr/sbin/sedispatch
`
## 2. Gestion du temps 

La gestion du temps est désormais gérée avec `systemd` avec la commande `timedatectl`.
`[alex@localhost ~]$ timedatectl
               Local time: ven. 2019-11-29 15:57:27 CET
           Universal time: ven. 2019-11-29 14:57:27 UTC
                 RTC time: ven. 2019-11-29 14:56:35
                Time zone: Europe/Paris (CET, +0100)
System clock synchronized: no
              NTP service: active
          RTC in local TZ: no`
UTC: Temps universel
RTC: Heure locale (du Système) 
CET: Heure Centrale Européenne

* 🌞 changer de timezone pour un autre fuseau horaire européen

`cd /etc` > 
`rm localtime` >
`ls /usr/share/zoneinfo`
`European`

* on peut activer ou désactiver l'utilisation de la syncrhonisation NTP avec `timedatectl set-ntp <BOOLEAN>`
  * 🌞 désactiver le service lié à la synchronisation du temps avec cette commande, et vérifier à la main qu'il a été coupé
 
`timedatectl set-ntp false`

Vérification:

`sudo nano /etc/systemd/timesyncd.conf`
 
## 3. Gestion de noms

* changement des noms de la machine avec `hostnamectl --set-hostname`
* il est possible de changer trois noms avec `--pretty`, `--static` et `--transient`
* 🌞 expliquer la différence entre les trois types de noms. Lequel est à utiliser pour des machines de prod?  

--pretty.   
--static.   
--transient.  

## 4. Gestion du réseau (et résolution de noms)

Pour gérer la stack réseau, deux outils sont livrés avec `systemd` :
* `NetworkManager`
  * souvent activé par défaut
  * réagit dynamiquement aux changements du réseau (mise à jour de `/etc/resolv.conf` en fonction des réseaux connectés par exemple)
  * idéal pour un déploiement desktop
  * expose une API dbus
* `systemd-networkd`
  * permet une grande flexibilité de configuration
    * configuration de plusieurs interfaces simultanément (wildcards)
    * fonctionnalités avancées
  * utilise une syntaxe standard `systemd`
  * complètement intégré à `systemd` (gestion, logs, etc)
  * idéal en déploiement cloud

### NetworkManager

NetworkManager est l'utilitaire réseau souvent démarré par défaut sur tous les OS GNU/Linux équipés de `systemd`. Il est utilisé pour configurer au cas par cas les interfaces réseaux d'une machine.
* il pilote les fichiers existants et introduit des fonctionnalités supplémentaires
  * il conserver et pilote le fichier `/etc/resolv.conf` par exemple
* il existe des outils pour interagir avec les interfaces qu'il gère
  * similaire à la suite `iproute2` (`ip a`, `ip route show`, `ip neigh show`, `ip net add`, etc)
  * comme l'outil en ligne de commande `nmcli`

Utilisation basique en ligne de commande :
* lister les interfaces et des informations liées
  * `nmcli con show`
  * `nmcli con show <INTERFACE>`
* modifier la configuration réseau d'une interface
  * éditer le fichier de configuratation d'une interface `/etc/sysconfig/network-scripts/ifcfg-<INTERFACE_NAME>`
  * recharger NetworkManager : `sudo systemctl reload NetworkManager` (relire les fichiers de configuration)
  * redémarrer l'interface `sudo nmcli con up <INTERFACE_NAME>`
* 🌞 afficher les informations DHCP récupérées par NetworkManager (sur une interface en DHCP)

Sinon une bonne interface curses des familles : avec la commande `nmtui`

> Avec le gestionnaire de paquet `dnf`, vous pouvez utiliser `dnf provides <COMMANDE>` pour trouver le nom du paquet à installer pour avoir une commande donnée

### `systemd-networkd`

Il est aussi possible que la configuration des interfaces réseau et de la résolution de noms soit enitèrement gérée par `systemd`, à l'aide du démon `systemd-networkd`.  

Dans le cas d'utilisation de `systemd-networkd`, il est préférable de désactiver NetworkManager, afin d'éviter les conflits et l'ajout de complexité superflue :
* 🌞 stopper et désactiver le démarrage de `NetworkManager`
* 🌞 démarrer et activer le démarrage de `systemd-networkd`

Il est alors possible de configurer des interfaces réseau dans `/etc/systemd/network` avec des fichiers `.network`.  
C'est le rôle du démon `systemd-networkd` que de surveiller ces fichiers et réagir aux changement d'état des cartes réseaux (comme un redémarrage).

La structure des fichiers `/etc/systemd/network/*.network` (le nom des fichiers est arbitraire) est la suivante : 
```
[Match]
Key=value

[Network]
Key=Value
```

La section `[Match]` permet de cibler une ou plusieurs interfaces (regex shells, ou liste avec des espaces) selon plusieurs critères comme le nom, l'ID udev, l'adresse MAC, etc. 

Par exemple, pour configurer une interface avec une IP statique : 
```
[Match]
Key=enp0s8

[Network]
Address=192.168.1.110/24
DNS=1.1.1.1
```

> Oh un fichier de configuration d'interface qui fonctionnent sous tous les OS GNU/Linux (qui sont équipés de `systemd`) 🔥🔥🔥

La [doc officielle détaille](https://www.freedesktop.org/software/systemd/man/systemd.network.html) (**faites-y un tour**) l'ensemble des clauses possibles et peut amener à réaliser des configurations extrêmement fines et poussées : 
* configuration de zone firewall, de serveurs DNS, de serveurs NTP, etc. par interface
* utilisation des protocoles liés aux VLAN, VXLAN, IPVLAN, VRF, etc
* configuration de fonctionnalités réseau comme le bonding/teaming, bridge, MacVTap etc
* création de tunnel ou interface virtuelle
* détermination manuelle de voisins L2 (table ARP)
* etc

* 🌞 éditer la configuration d'une carte réseau de la VM avec un fichier `.network`

> A noter qu'un outil comme `nmtui` verra les configurations réalisées avec NetworkManager et avec `systemd-networkd`, car ils pilotent tous les deux la même API. 

### `systemd-resolved`

L'activation de `systemd-resolved` permet une résolution des noms de domaines avec un serveur DNS local sandboxé (sur certaines distributions c'est fait par défaut). Certains bénéfices de l'utilisation de `systemd-resolved` sont :
* configuration de DNS par interface
  * aucune requête sur des DNS potentiellement injoignables 
    * = pas de leak d'infos
    * = optimisation du temps de lookup
* résolution robuste avec un serveur DNS local sandboxé
* support natif de fonctionnalités comme DNSSEC, DNS over TLS, caching DNS

* 🌞 activer la résolution de noms par `systemd-resolved` en démarrant le service (maintenant et au boot)
  * vérifier que le service est lancé
* 🌞 vérifier qu'un serveur DNS tourne localement et écoute sur un port de l'interfce localhost (avec `ss` par exemple)
  * effectuer une commande de résolution de nom en utilisant explicitement le serveur DNS mis en place par `systemd-resolved` (avec `dig`)
  * effectuer une requête DNS avec `systemd-resolve`
* on peut utiliser `resolvectl` pour avoir des infos sur le serveur local

> `systemd-resolved` apporte beaucoup de fonctionnalités comme du caching, le support de DNSSEC ou encore de DNS over TLS. Une fois qu'il est en place, tout passe par lui, le fichier `/etc/resolv.conf est donc obsolète`

* 🌞 Afin d'activer de façon permanente ce serveur DNS, la bonne pratique est de remplacer `/etc/resolv.conf` par un lien symbolique pointant vers `/run/systemd/resolve/stub-resolv.conf`
* 🌞 Modifier la configuration de `systemd-resolved`
  * elle est dans `/etc/systemd/resolved.conf`
  * ajouter les serveurs de votre choix
  * vérifier la modification avec `resolvectl`
* 🌞 mise en place de DNS over TLS
  * renseignez-vous sur les avantages de DNS over TLS
  * effectuer une configuration globale (dans `/etc/systemd/resolved.conf`)
    * compléter la clause `DNS` pour ajouter un serveur qui supporte le DNS over TLS (on peut en trouver des listes sur internet)
    * utiliser la clause `DNSOverTLS` pour activer la fonctionnalité
      * valeur `opportunistic` pour tester les résolutions à travers TLS, et fallback sur une résolution DNS classique en cas d'erreur
      * valeur `yes` pour forcer les résolutions à travers TLS
  * prouver avec `tcpdump` que les résolutions sont bien à travers TLS (les serveurs DNS qui supportent le DNS over TLS écoutent sur le port 853)
* 🌞 activer l'utilisation de DNSSEC

## 5. Gestion de sessions `logind`

`logind` est le nom du démon qui gère les sessions.  

On peut le manipuler avec `loginctl`. Rien de fou ici (si on omet les détails techniques de gestion de session), je vous laisse explorer un peu la ligne de commande.

## 6. Gestion d'unité basique (services)

La principale entité que `systemd` gère sont les *unités systemd* ou *systemd units*.  
Les unités sont définies dans des fichiers texte et permettent de manipuler différents éléments du système : 
* services
* point de montage
* carte réseau
* autres. (later)

Les paths où sont définis les unités sont les suivants, du moins prioritaire, au plus prioritaires (non-exhaustif) :
* `/usr/lib/systemd/system` : utilisé par la plupart des installations par défaut
  * faites un tour et regardez un peu ce qui se balade là-bas
* `/run/systemd/system` : utilisé au runtime d'un service
* `/etc/systemd/system` : dossier dédié à la modification par l'administrateur

> Donc si on veut ajouter une nouvelle unité, c'est dans `/etc/systemd/system`.

---

Manipulation d'unité `systemd`

* lister les unités `systemd` actives de la machine
  * `systemctl list-units`
  * ou seulement `systemctl`
  * on peut ajouter des options pour filtrer par type
  * pour ne lister que les services : `systemctl list-units -t service`

> Beaucoup de commandes `systemd` qui écrivent des choses en sortie standard sont automatiquement munie d'un pager (pipé dans `less`). On peut ajouter l'option `--no-pager` pour se débarasser de ce comportement

* pour obtenir plus de détails sur une unitée donnée
  * `systemctl is-active <UNIT>`
    * détermine si l'unité est actuellement en cours de fonctionnement
  * `systemctl is-enabled <UNIT>`
    * détermine si l'unité est liée à un target (généralement, on s'en sert pour activer des unités au démarrage)
  * `systemctl status <UNIT>`
    * affiche l'état complet d'une unité donnée
    * comme le path où elle est définie, si elle a été enable, tous les processus liés, etc.
  * `systemctl cat <UNIT>`
    * affiche les fichiers qui définissent l'unité

Les outils de l'écosystème GNU/Linux ont été modifiés pour être utilisés avec `systemd`
* par exemple `ps`
* on peut utiliser `ps` pour trouver l'unité associée à un processus donné : `ps -e -o pid,cmd,unit`
* on peut aussi effectuer un `systemctl status <PID>` qui nous retournera l'état de l'unité qui est responsable de ce PID
  * les logs sont fournis par *journald*, les stats système par les mécanismes de *cgroups* (on y revient plus tard)
* 🌞 trouver l'unité associée au processus `chronyd`
