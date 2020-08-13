---
title: Remplacer sa box SFR par un routeur pfSense
date: 2020-08-13 16:32:23
tags:
---

Ce document à pour but d'expliquer la procédure permettant le remplacement d'une box SFR NVB6AC (fibre FTTH) par un routeur pfSense. Ce document est à but éducatif uniquement et ne vous incite aucunement à rendre votre matériel à SFR. 

Si vous effectuez ces manipulations sur l'installation de votre domicile, le service client SFR sera certainement dans l'impossibilité de vous dépanner. Je décline l'entière responsabilité des possibles dommages causés à votre ligne, installation, et matériel. À vos risques et périls !


---

## Préambule

Il est nécessaire de posséder au moins deux interfaces réseau sur la machine où sera installée le pfSense. Il est à noter qu'il est aussi possible d'utiliser d'autres OS que pfSense, si vous êtes à l'aise, n'hésitez pas à porter la config. N'hésitez pas à me [contacter par email](mailto:jessy.sobreiro@epitech.eu) pour l'intégrer à ce tutoriel. 

Si la TV ne vous intéresse pas (pas de support IGMP Proxy ni de Multicast Routing sur le kernel), un tutoriel est disponible pour l'[Ubiquiti UniFi Dream Machine Pro (UDM-PRO)](https://blog.jess-sys.fr/articles/udmp-sfr).

Tout au long de ce tutoriel, il sera fait référence à plusieurs variables (modifiez ces variables pour refléter votre réseau, sans cela, rien ne fonctionnera). J'utiliserai donc:

- `192.168.1.0/24` comme subnet côté LAN pfSense (avec 192.168.1.1 comme gateway)
- `igb0` sera l'interface WAN (adaptateur fibre) côté pfSense
- `igb1` sera l'interface LAN (réseau local) côté pfSense

<u>Attention :</u> Pour accéder aux services TV, il est nécessaire de récupérer des fichiers de configuration sur la box (voir *TV - Étape 1* ci-dessous). Il est nécessaire de compléter cette opération avant de continuer.



---

## Internet (IP, DHCP)

C'est plutôt simple, vous verrez.

### Étape 1 : On débranche, et on rebranche!

* Déconnectez la box SFR, débranchez l'ONT éléctriquement.
* Branchez le port ethernet de l'ONT (vérifiez aussi que la fibre y est bien branchée...) sur le port WAN de votre pfSense.
* Reconnectez l'alimentation éléctrique de l'ONT



### Étape 2 : Configurer le DHCP côté pfSense

Pour obtenir une adresse IP, il est nécessaire de spécifier l'Option 60 (vendor-class-identifier) lors du `DHCPREQUEST`.

Pour cela, dirigez-vous vers l'interface de configuration de pfSense, puis allez dans "**Interfaces > WAN**". Renseignez ensuite le paramètre `Send options` avec la valeur suivante :

```
dhcp-class-identifier "neufbox6"
```

C'est tout. Sauvegardez et appliquez la conf.



---

## Téléphone (VoIP, SIP, Asterisk)

C'est là que tout se corse. Au boulot!

### Étape 1 : Récupérer vos identifiants VoIP

Après avoir connecté internet (avec la box SFR ça ne fonctionnera pas), suivez l'excellent guide de Florent Daignière (merci à lui d'avoir développé ce magnifique petit outil) : https://florent.daigniere.com/posts/2019/04/extracting-voip-credentials-from-my-broadband-router/

### Étape 2 : Configurer un serveur VoIP Asterisk

Je pars du principe que vous avez trouvé un moyen de compiler et démarrer un serveur **Asterisk** (version 16.6.0, problèmes de compatibilité avec des versions antérieures, mauvais support de chan_sip dans des versions plus récents). Il peut tourner sur une Raspberry (~40€) sur votre LAN.

Voici les fichiers de configuration à utiliser (copiez-collez, modifiez les variables) :

Configuration générale, enregistrement trunk SFR : `sip.conf` 

```
test
```

Configuration de vos téléphones physiques : `users.conf`

```
test
```

Configuration des routes : `extensions.conf`

```
test
```

Redémarrez Asterisk :

```
sudo systemctl restart asterisk
```

### Étape 3 : Configurez votre téléphone physique

Plusieurs choix s'offrent à vous, la configuration suit les mêmes principes peu importe l'option choisie.

* Si vous voulez gardez votre combiné filaire/sans fil, vous pouvez acheter un **Cisco SPA112**, il fera le lien entre votre téléphone et le serveur **Asterisk**.
* Acheter un téléphone VoIP (de bonnes occasions peu chères (20€ ~) sont disponibles sur eBay).

Pour l'exemple, j'utilise un téléphone VoIP **Cisco SPA525G**. L'interface de configuration est quasi-identique à celle du SPA112.



---

## TV (IPTV, Multicast, IGMP)

Pour les puristes qui veulent profiter du décodeur TV et des rares chaines du câble disponibles chez SFR, il y a un peu plus de travail.

> **Attention :**  Si vous souhaitez brancher plusieurs appareils sur votre réseau, il faudra un switch manageable capable d'**IGMP Snooping**, sinon dites aurevoir à votre débit. Si vous avez plusieurs interfaces réseau disponibles, vous pouvez créer un bridge entre deux interfaces et utiliser la fonction d'IGMP Snooping de pfSense. Vous pourrez ensuite brancher un dumb switch sur l'une des deux interfaces.

### Étape 1 : Récupérer les fichiers de configuration de la box

Pour que le décodeur TV fonctionne correctement, il est nécessaire de récupérer des fichiers de configuration sur votre box. 

```
http://ip-neufbox/api/1.0/?method=system.getInfo
http://ip-neufbox/api/1.0/?method=wan.getInfo
http://ip-neufbox/api/1.0/?method=ftth.getInfo
http://ip-neufbox/api/1.0/?method=tv.getInfo
http://ip-neufbox/api/1.0/?method=usb.getInfo
http://ip-neufbox/api/1.0/?method=lan.getHostsList
http://ip-neufbox/api/1.0/?method=lan.getInfo
```

> **Note :** Remplacez `ip-neufbox` par l'adresse IP locale de votre box SFR. Le plus souvent, ce sera 192.168.1.1 ou 192.168.0.1, sauf si vous l'avez changé volontairement (en modifiant l'HTML de la page avant la validation du formulaire, c'est possible sur certains modèles).
>
> Le décodeur TV ira chercher ces fichiers de configuration sur l'IP de la passerelle réseau, si vous souhaitez utiliser une plage d'IP différente, veillez à ce qu'un serveur HTTP répondant à ces requêtes (ou un reverse proxy) soit configuré sur cette adresse IP.

Pour référence, vous trouverez des fichiers de configuration d'exemple ci-dessous (remplacez vos adresses MAC et plages d'IP à votre convenance):

[system.getInfo](https://blog.jess-sys.fr/files/system.info.xml), [wan.getInfo](https://blog.jess-sys.fr/files/wan.info.xml), [ftth.getInfo](https://blog.jess-sys.fr/files/ftth.info.xml), [tv.getInfo](https://blog.jess-sys.fr/files/tv.info.xml), [usb.getInfo](https://blog.jess-sys.fr/files/usb.info.xml), [lan.getHostsList](https://blog.jess-sys.fr/files/lan.hosts.xml), [lan.getInfo](https://blog.jess-sys.fr/files/lan.info.xml)

### Étape 2 : Servir les fichiers de configuration

##### Étape 2.1 : Servir les requêtes

Utilisez la même Raspberry Pi que vous avez utilisé pour faire tourner **Asterisk**. Installez le serveur web **Nginx**. 

Éditez `/etc/nginx/sites-available/default`, pour refléter cette conf :

```nginx
server {
    
}
```

Redémarrez **Nginx** et activez son démarrage au boot :

```bash
sudo systemctl enable nginx && sudo systemctl restart nginx
```

##### Étape 2.2 : Rediriger les requêtes

Comme dit précédemment, le décodeur TV ira chercher les fichiers de configuration sur l'IP de la gateway sur le LAN. Il faut donc configurer pfSense pour qu'il redirige ces requêtes vers la Raspberry Pi.

###### Étape 2.2.1 : Changement de port HTTP par défaut

pfSense écoute le port 80 par défaut, il faut changer son port d'écoute par défaut dans "**System > Advanced**", cocher *HTTP*, puis spécifier un autre port (8088 par exemple). Appliquer la conf, puis connectez vous sur l'interface avec ce nouveau port.

###### Étape 2.2.1 : Configuration de HAProxy

Dans "**System > Packages > Available**", cherchez `HAProxy` et installez-le.

### Étape 3 : Activer le proxy IGMP (IGMPProxy)

Pour que les trames Multicast soient correctement forwardées entre le WAN et le décodeur TV, il est nécessaire de configurer un proxy IGMP sur le pfSense. Sans cela, pas de flux vidéo.

##### Etape 3.1 : Configurer le service IGMPProxy

Dans l'interface de configuration pfSense, aller dans **Diagnostics > Edit file**. Puis éditez `/etc/inc/services.inc`.

Dans la fonction `services_igmpproxy_configure`, cherchez la ligne spécifiant le chemin du fichier de configuration chargé au démarrage d'IGMP Proxy dans mon cas, lignes 1838 et 1840.

Modifiez la ligne 1838 contenant:

```php
		mwexec_bg("/usr/local/sbin/igmpproxy -v {$g['varetc_path']}/igmpproxy.conf");
```

Par :

```php
		mwexec_bg("/usr/local/sbin/igmpproxy -v /etc/igmpproxy.conf");
```

Modifiez la ligne 1840 contenant :

```php
		mwexec_bg("/usr/local/sbin/igmpproxy {$g['varetc_path']}/igmpproxy.conf");
```

Par :

```php
		mwexec_bg("/usr/local/sbin/igmpproxy /etc/igmpproxy.conf");
```

Cela va nous permettre de charger un fichier de configuration personnalisé. Il est nécessaire de charger un fichier généré manuellement, car la génération via l'interface ne détecte pas les bonnes interfaces (dans mon cas).

##### Etape 3.2 : Configurer IGMPProxy

Toujours dans l'interface de configuration pfSense, aller dans **Diagnostics > Edit file**. Puis éditez `/etc/igmpproxy.conf`. Copiez-collez le code suivant, puis sauvegardez.

```
# 
# igb0 est l'interface réseau connectée à l'adaptateur fibre côté WAN
# igb1 est l'interface connectée au switch côté LAN
# 

# Commenter cette ligne si vous avez l'option Multi-TV
quickleave

phyint igb0 upstream ratelimit 0 threshold 1
altnet 192.168.1.0/24

phyint igb1 downstream ratelimit 0 threshold 1

# Si vous avez d'autres interfaces réseau, pensez à les désactiver
phyint igb2 disabled
```

##### Étape 3.3 :  Activer IGMPProxy

Toujours dans l'interface de configuration pfSense, aller dans **Services > IGMP Proxy**, cochez `Enabled` puis sauvegardez. 

### Étape 4 : Configurer le firewall pour accepter le Multicast



---

Jessy SOBREIRO <jessy.sobreiro@epitech.eu) aka. jess-sys
10 août 2020
