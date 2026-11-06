# orange-iptv (version épurée)

TV Orange (VLAN 840) sur une passerelle UniFi OS : crée l'interface `iptv` sur le port WAN
et lance le proxy IGMP `improxy` (fourni avec UniFi OS) vers le réseau du décodeur.

Version réduite de [jbbodart/orange-iptv](https://github.com/jbbodart/orange-iptv) (GPLv2) :
plus de script shell, une unité systemd et la conf improxy.

## Prérequis

- Internet Orange (VLAN 832) déjà fonctionnel sur la passerelle.
- Contrôleur, réglages du WAN : **IGMP Proxy décoché**.
- IGMP snooping activé sur le réseau du décodeur.
- Décodeur servi en DHCP avec les DNS Orange (`80.10.246.136`, `81.253.149.6`) et l'option 125
  (format dans le [README d'origine](https://github.com/jbbodart/orange-iptv#prérequis)).

## Module GPON/XGS-PON sur le SFP+

Si l'ONT est un module GPON/XGS-PON (WAS-110, etc.) branché sur le port SFP+, **le flow control
(802.3x) doit être désactivé** dès qu'on utilise la TV. Les rafales du flux multicast vers des
appareils en 1G provoquent des trames de pause (`tx_pause`/`rx_pause`) qui remontent jusqu'au
port du module. Le module ne peut pas mettre l'OLT en pause : il jette le trafic, Internet
compris, à chaque pic (allumage de la TV, zapping).

Le port du module n'est pas géré par l'interface UniFi : `ethtool -A <port> rx off tx off`.

## Réglages

- `orange-iptv.service` : `WAN=` = port WAN de la passerelle.
- `improxy.conf` : `downstream` = bridge du réseau du décodeur.

## Installation

Copier ce dossier dans `/data/orange-iptv` sur la passerelle, puis :

```sh
cp /data/orange-iptv/orange-iptv.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now orange-iptv
```

Mises à jour du firmware : elles écrasent `/usr` (donc `/lib`), où l'original s'installait.
`/data` et `/etc/systemd/system` semblent conservés, donc le service devrait survivre, mais
Ubiquiti ne le garantit pas : vérifier `systemctl status orange-iptv` après chaque mise à jour,
et relancer les trois commandes ci-dessus si l'unité a disparu.

Après une modif de `improxy.conf` : `systemctl restart orange-iptv`.
Après une modif de l'unité : recopier, `systemctl daemon-reload`, puis restart.

## Vérification

```sh
ip -d link show iptv            # vlan id 840, egress-qos-map 0:5 ...
systemctl status orange-iptv    # improxy en running
journalctl -u orange-iptv -f
```

## Testé

UCG-Fiber, pare-feu par zones, 2026-10-30 : TV OK sans règle de pare-feu. `iptv` n'est dans
aucune zone UniFi, et le multicast passe par la politique ACCEPT de `FORWARD`. `rp_filter` vaut
déjà 2 sur `iptv`. Si une mise à jour bloque les interfaces hors zone, ajouter dans l'unité :
`ExecStartPost=/usr/sbin/iptables -A FORWARD -i iptv -o br10 -p udp -d 224.0.0.0/4 -j ACCEPT`.

## Désinstallation

```sh
systemctl disable --now orange-iptv
rm /etc/systemd/system/orange-iptv.service
systemctl daemon-reload
```
