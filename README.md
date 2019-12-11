# I. systemd-basics

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

`nmcli con show`
Détailler une interface : `nmcli con show enp0s8`
Modification à faire pour accèder en SSH à la VM : `nano /etc/sysconfig/network-scripts/ifcfg-enp0s8`
Infos DHCP récupérée par le Network Manager : `nmcli con show enp0s3 | grep DHCP`

### `systemd-networkd`

Désactivation NetworkManager : `sudo systemctl stop NetworkManager` 
Contrôler : `sudo systemctl status NetworkManager`

Fichier enp0s8.network :

`cat enp0s8.network`

Retour :

`[Match]
Key=enp0s8

[Network]
Address=192.168.2.2/24
DNS=8.8.8.8`

### `systemd-resolved`

Activer la résolution de noms par systemd-resolved en démarrant le service (maintenant et au boot) :

`sudo systemctl start systemd-resolved`
`sudo systemctl enable systemd-resolved`


## 5. Gestion de sessions `logind`

`loginctl`

SESSION  UID USER SEAT TTY  
      1 1000 alex      pts/0

1 sessions listed.

## 6. Gestion d'unité basique (services)


Lancer chronyd `systemcl start chronyd`
Puis pour l'unité associée à chronyd : `ps -e -o pid,cmd,unit | grep chronyd`

# II. Boot et Logs

Pour générer le graphique, c'est la commande suivante :

`systemd-analyze plot > graphe.svg`
