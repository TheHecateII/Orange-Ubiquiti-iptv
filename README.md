# Orange-Ubiquiti-iptv

La TV d'Orange (France) sur une passerelle UniFi OS (Cloud Gateway, UXG, Dream Machine…),
sans Livebox.

Orange transporte Internet sur le VLAN 832 et la TV en direct sur le VLAN 840, en multicast.
L'interface UniFi ne permet de configurer qu'un seul VLAN sur le port WAN : la TV ne peut donc
pas être reçue sans intervention. Ce projet ajoute l'interface VLAN 840 sur le port WAN et
relaie le flux TV vers le réseau du décodeur avec `improxy`, le proxy IGMP déjà présent dans
UniFi OS.

Il ne contient aucun script : une unité systemd et un fichier de configuration suffisent.

## Fonctionnement

```
ONT ── port WAN ─┬─ VLAN 832 : Internet (géré par UniFi)
                 └─ VLAN 840 : interface iptv ──► improxy ──► réseau du décodeur (brXX)
```

Au démarrage, le service `orange-iptv` :

1. crée l'interface `iptv` (VLAN 840) sur le port WAN, avec une priorité 802.1p de 5 ;
2. lui attribue une adresse factice (`192.168.255.254/24`) : il n'y a pas de DHCP sur le
   VLAN 840, mais `improxy` a besoin d'une adresse pour démarrer ;
3. lance `improxy`, qui transmet les demandes IGMP du décodeur à Orange et relaie le flux
   multicast vers le réseau du décodeur.

À l'arrêt, l'interface `iptv` est supprimée.

## Compatibilité

| Matériel | Résultat |
|---|---|
| UniFi Cloud Gateway Fiber (UCG-Fiber), pare-feu par zones, module XGS-PON en SFP+ | Fonctionne |
| UniFi Gateway Max, UniFi OS 4.1.13, Network 9.4.19, décodeurs UHD et TV6 | Fonctionne avec le [projet d'origine](https://github.com/jbbodart/orange-iptv) |

Les autres passerelles UniFi OS devraient fonctionner : seuls les noms du port WAN et du
bridge changent.

## Prérequis

- Une connexion Internet Orange déjà fonctionnelle sur la passerelle (VLAN 832 configuré dans
  UniFi).
- Un accès SSH à la passerelle (paramètres de la console UniFi OS).
- Dans les réglages du WAN : **IGMP Proxy décoché**. Sinon, le proxy intégré d'UniFi entre en
  conflit avec celui du service.
- Sur le réseau du décodeur : **IGMP Snooping activé**, pour que le flux TV n'inonde pas tout
  le réseau, Wi-Fi compris.
- Le décodeur doit recevoir en DHCP :
  - uniquement les DNS d'Orange : `80.10.246.136` et `81.253.149.6` ;
  - l'option 125, construite à partir de l'adresse MAC, du numéro de série et de la version de
    la Livebox. Le format est décrit dans le
    [README du projet d'origine](https://github.com/jbbodart/orange-iptv#prérequis), et un
    [générateur](https://lafibre.info/remplacer-livebox/le-guide-complet-pour-usgusg-pro-internet-tv-livebox-ipv6/msg608516/#msg608516)
    est disponible sur lafibre.info.

UniFi applique les options DHCP à tout un réseau. Deux possibilités :

- placer le décodeur dans un réseau dédié ;
- ou confier le DHCP à un serveur externe (dnsmasq par exemple), qui peut cibler le décodeur
  par son adresse MAC et le laisser sur le réseau principal.

## Module GPON/XGS-PON en SFP+

Si l'ONT est un module GPON/XGS-PON (WAS-110 ou équivalent) branché sur le port SFP+, **le flow
control (802.3x) doit être désactivé** dès que la TV est utilisée.

Les rafales du flux multicast vers des appareils en 1 Gbit/s provoquent des trames de pause
(`tx_pause`/`rx_pause`) qui remontent jusqu'au port du module. Le module ne peut pas mettre le
réseau d'Orange en pause : il jette le trafic, Internet compris, à chaque pic (allumage de la
TV, zapping).

Le port du module n'est pas géré par l'interface UniFi. Il se règle sur la passerelle avec
`ethtool -A <port> rx off tx off`.

## Installation

### 1. Télécharger le projet sur la passerelle

Le dossier doit s'appeler exactement `/data/orange-iptv` : ce chemin est écrit dans l'unité.

```sh
cd /data
curl -sL https://github.com/TheHecateII/Orange-Ubiquiti-iptv/archive/refs/heads/main.tar.gz | tar -xz
mv Orange-Ubiquiti-iptv-main orange-iptv
```

### 2. Adapter les deux réglages

**Port WAN**, dans `orange-iptv.service` (ligne `Environment=WAN=`, `eth6` par défaut). C'est le
port physique qui porte déjà le VLAN 832 d'Internet. Son nom apparaît après le `@` :

```sh
ip -d link | grep -B2 'vlan protocol 802.1Q id 832'
# 26: eth6.832@eth6: ...   →   WAN=eth6
```

**Bridge du réseau du décodeur**, dans `improxy.conf` (ligne `downstream`, `br10` par défaut).
UniFi nomme ses bridges `br<id du VLAN>` (`br0` pour le réseau par défaut). Le bon est celui
qui porte l'adresse de la passerelle sur ce réseau :

```sh
ip -br addr | grep br
```

Les deux fichiers se modifient avec `vi` directement sur la passerelle.

### 3. Activer le service

```sh
cp /data/orange-iptv/orange-iptv.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now orange-iptv
```

Après une modification de `improxy.conf` : `systemctl restart orange-iptv`.
Après une modification de l'unité : recopier le fichier, `systemctl daemon-reload`, puis
`systemctl restart orange-iptv`.

## Vérification

```sh
systemctl status orange-iptv    # « active (running) », improxy en processus principal
ip -d link show iptv            # iptv@<port WAN>, vlan id 840, egress-qos-map { 0:5 ... }
```

La ligne `Cannot find device "iptv"` au démarrage est normale : le service supprime d'abord une
éventuelle interface restante.

Une fois le décodeur allumé sur une chaîne, une route multicast doit apparaître, avec des
compteurs qui augmentent :

```sh
ip -s mroute
# (81.253.x.x,232.0.x.x)   Iif: iptv   Oifs: br10   State: resolved
```

## Dépannage

| Constat | Piste |
|---|---|
| Le service ne démarre pas | `journalctl -u orange-iptv` : vérifier le nom du port WAN et du bridge, et que l'IGMP Proxy du WAN est bien décoché. |
| Le décodeur affiche une erreur au démarrage | Côté DHCP : DNS d'Orange ou option 125. |
| Aucun flux n'arrive sur `iptv` | `tcpdump -ni iptv igmp` : les demandes IGMP (`igmp v2 report`) doivent partir au changement de chaîne. |
| Le flux arrive sur `iptv` mais aucune route dans `ip mroute` | Filtrage de la source : `sysctl -w net.ipv4.conf.iptv.rp_filter=2`. |
| Route présente, compteurs en hausse, rien sur le bridge | Pare-feu : voir ci-dessous. |
| Coupures d'Internet au zapping | Voir [Module GPON/XGS-PON en SFP+](#module-gponxgs-pon-en-sfp). |

**Pare-feu.** L'interface `iptv` n'appartient à aucune zone UniFi et ne peut donc pas être
visée depuis le contrôleur. Sur UCG-Fiber, en pare-feu par zones, aucune règle n'est
nécessaire : le multicast passe par la politique par défaut. Si le flux est bloqué chez vous,
ajoutez dans l'unité (en adaptant le bridge) :

```ini
ExecStartPost=/usr/sbin/iptables -A FORWARD -i iptv -o br10 -p udp -d 224.0.0.0/4 -j ACCEPT
```

## Mises à jour du firmware

Une mise à jour d'UniFi OS remplace le système. `/data` est conservé, et `/etc/systemd/system`
semble l'être aussi, mais Ubiquiti ne le garantit pas. Après chaque mise à jour, vérifiez :

```sh
systemctl status orange-iptv
```

Si le service a disparu, relancez les trois commandes de l'étape 3 : la configuration dans
`/data/orange-iptv` est intacte.

## Désinstallation

```sh
systemctl disable --now orange-iptv
rm /etc/systemd/system/orange-iptv.service
systemctl daemon-reload
rm -r /data/orange-iptv
```

## Crédits et licence

Version simplifiée de [jbbodart/orange-iptv](https://github.com/jbbodart/orange-iptv), lui-même
basé sur [fabianishere/udm-iptv](https://github.com/fabianishere/udm-iptv).

Distribué sous licence GPLv2 : voir [COPYING.txt](COPYING.txt).
