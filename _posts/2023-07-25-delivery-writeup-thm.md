---
title: Delivery Writeup - HackTheBox
date: 2023-07-25
categories: [Writeups, HTB]
tags: [Linux, HTB, CTF, osTicket, MatterMost, Hash Cracking]
img_path: /assets/img/commons/delivery/
image: delivery.png
---

¡Saludos!

En este writeup, nos sumergiremos en la máquina [**Delivery**](https://app.hackthebox.com/machines/308) de **HackTheBox**, la cual está categorizada con un nivel de dificultad fácil según la plataforma. Se trata de una máquina **Linux** en la que llevaremos a cabo una **enumeración HTTP** para identificar subdominios que nos permitirán acceder a un sistema de tickets de soporte que explotaremos con el fin de obtener acceso a **MatterMost**. Gracias a una **fuga de información**, conseguiremos las credenciales necesarias para acceder al sistema y también obtendremos una pista que nos guiará en la escalada de privilegios mediante **cracking de hashes**.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.10.222
PING 10.10.10.222 (10.10.10.222) 56(84) bytes of data.
64 bytes from 10.10.10.222: icmp_seq=1 ttl=63 time=125 ms

--- 10.10.10.222 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 125.038/125.038/125.038/0.000 ms
```

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.10.222 -oG allPorts.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-25 11:51 -05
Nmap scan report for 10.10.10.222
Host is up (0.12s latency).
Not shown: 58612 closed tcp ports (reset), 6920 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8065/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 19.51 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,80,8065 -sV -sC --min-rate 5000 10.10.10.222 -oN services.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-25 11:54 -05
Nmap scan report for 10.10.10.222
Host is up (0.11s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)
|_  256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)
80/tcp   open  http    nginx 1.14.2
|_http-title: Welcome
|_http-server-header: nginx/1.14.2
8065/tcp open  unknown
| fingerprint-strings: 
|   GenericLines, Help, RTSPRequest, SSLSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: no-cache, max-age=31556926, public
|     Content-Length: 3108
|     Content-Security-Policy: frame-ancestors 'self'; script-src 'self' cdn.rudderlabs.com
|     Content-Type: text/html; charset=utf-8
|     Last-Modified: Tue, 25 Jul 2023 06:43:13 GMT
|     X-Frame-Options: SAMEORIGIN
|     X-Request-Id: akrpu45bdj8a8cdngr7gckn41r
|     X-Version-Id: 5.30.0.5.30.1.57fb31b889bf81d99d8af8176d4bbaaa.false
|     Date: Tue, 25 Jul 2023 16:54:18 GMT
|     <!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=0"><meta name="robots" content="noindex, nofollow"><meta name="referrer" content="no-referrer"><title>Mattermost</title><meta name="mobile-web-app-capable" content="yes"><meta name="application-name" content="Mattermost"><meta name="format-detection" content="telephone=no"><link re
|   HTTPOptions: 
|     HTTP/1.0 405 Method Not Allowed
|     Date: Tue, 25 Jul 2023 16:54:18 GMT
|_    Content-Length: 0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8065-TCP:V=7.94%I=7%D=7/25%Time=64BFFE3A%P=x86_64-pc-linux-gnu%r(Ge
SF:nericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(GetRequest,DF3,"HTTP/1\.0\x20200\x20OK\r\nAccept-Ranges:\
SF:x20bytes\r\nCache-Control:\x20no-cache,\x20max-age=31556926,\x20public\
SF:r\nContent-Length:\x203108\r\nContent-Security-Policy:\x20frame-ancesto
SF:rs\x20'self';\x20script-src\x20'self'\x20cdn\.rudderlabs\.com\r\nConten
SF:t-Type:\x20text/html;\x20charset=utf-8\r\nLast-Modified:\x20Tue,\x2025\
SF:x20Jul\x202023\x2006:43:13\x20GMT\r\nX-Frame-Options:\x20SAMEORIGIN\r\n
SF:X-Request-Id:\x20akrpu45bdj8a8cdngr7gckn41r\r\nX-Version-Id:\x205\.30\.
SF:0\.5\.30\.1\.57fb31b889bf81d99d8af8176d4bbaaa\.false\r\nDate:\x20Tue,\x
SF:2025\x20Jul\x202023\x2016:54:18\x20GMT\r\n\r\n<!doctype\x20html><html\x
SF:20lang=\"en\"><head><meta\x20charset=\"utf-8\"><meta\x20name=\"viewport
SF:\"\x20content=\"width=device-width,initial-scale=1,maximum-scale=1,user
SF:-scalable=0\"><meta\x20name=\"robots\"\x20content=\"noindex,\x20nofollo
SF:w\"><meta\x20name=\"referrer\"\x20content=\"no-referrer\"><title>Matter
SF:most</title><meta\x20name=\"mobile-web-app-capable\"\x20content=\"yes\"
SF:><meta\x20name=\"application-name\"\x20content=\"Mattermost\"><meta\x20
SF:name=\"format-detection\"\x20content=\"telephone=no\"><link\x20re")%r(H
SF:TTPOptions,5B,"HTTP/1\.0\x20405\x20Method\x20Not\x20Allowed\r\nDate:\x2
SF:0Tue,\x2025\x20Jul\x202023\x2016:54:18\x20GMT\r\nContent-Length:\x200\r
SF:\n\r\n")%r(RTSPRequest,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConten
SF:t-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n
SF:400\x20Bad\x20Request")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r
SF:\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close
SF:\r\n\r\n400\x20Bad\x20Request")%r(SSLSessionReq,67,"HTTP/1\.1\x20400\x2
SF:0Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nCon
SF:nection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(TerminalServerCookie
SF:,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;
SF:\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request"
SF:);
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 103.47 seconds
```

El reporte de Nmap revela tres servicios en ejecución, **SSH** (`OpenSSH 7.9p1`) en el puerto `22`, un servicio HTTP (`nginx 1.14.2`) en el puerto `80` y un servicio desconocido en el `8065`. 

### HTTP - 80

Si navegamos a `http://10.10.10.222`, encontramos la siguiente página web.

![web](web.png)

Al presionar sobre el botón "**CONTACT US**", identificamos el dominio `delivery.htb` y el subdominio `helpdesk.delivery.htb`.

![contact](contact.png)

Procedimos a agregarlos al archivo `/etc/hosts` para resolver los nombres de dominio a la dirección ip `10.10.10.222`.

```bash
10.10.10.222    delivery.htb    helpdesk.delivery.htb
```

El dominio `delivery.htb` nos redirige a la misma página web, mientras que el subdominio `helpdesk.delivery.htb` nos redirige a la siguiente página web del sistema de tickets de soporte `osTicket`.

![osTicket](osTicket.png)

En la sección "**CONTACT US**", se indicaba que los usuarios no registrados deben comunicarse con el HelpDesk. Después, utilizando un correo electrónico válido con dominio `@delivery.htb`, podrán acceder al servidor **MatterMost**.

Por otro lado, al hacer clic en el enlace "**MatterMost server**", seremos redirigidos al panel de inicio de sesión de **MatterMost**, que se ejecuta en el puerto `8065`.

![MatterMost](MatterMost.png)

En este punto, intentemos crear un nuevo ticket en el sistema de tickets de soporte.

![ticket](ticket.png)

Después de pulsar el botón "`Create Ticket`", seremos redirigidos a una página que nos indica que la solicitud de ticket de soporte ha sido creada.

![support](support.png)

Además, se nos generó un ID de ticket y un correo electrónico temporal con dominio `@delivery.htb`. Para ver el estado del ticket, navegamos a la sección "**Check Ticket status**" e introducimos el correo electrónico con el cual creamos el ticket y el **ID** del ticket.

![status](status.png)

## Explotación

---

Perfecto, ahora que tenemos un correo electrónico válido con dominio `@delivery.htb`, lo utilizamos para intentar crear nuestra cuenta en **MatterMost**.

![MatterMost2](MatterMost2.png)

![MatterMost3](MatterMost3.png)

Si verificamos el estado del ticket, veremos que hemos recibido un correo con el enlace para activar la cuenta de **MatterMost**.

![status2](status2.png)

Accedemos al enlace proporcionado para verificar la cuenta y luego iniciamos sesión en **MatterMost**.

Dentro de **MatterMost**, encontramos el único chat público con algunos comentarios. En estos comentarios, mencionan las credenciales del servidor y sugieren desarrollar un programa para evitar la reutilización de contraseñas, especialmente aquellas que contengan la frase `"PleaseSubscribe!"`. También advierten que si alguien obtiene los hashes, es posible descifrar fácilmente los valores en texto plano utilizando reglas de `hashcat`.

![channels](channels.png)

Luego de haber obtenido las credenciales del servidor en **MatterMost**, ingresamos al panel de agentes de **osTicket** utilizando dichas credenciales.

![osTicket2](osTicket2.png)

Al acceder al panel de agentes de **osTicket**, podemos ver la lista de tickets creados. Sin embargo, parece que en este panel no encontramos muchas tareas por realizar o acciones adicionales que podamos llevar a cabo.

![osTicket3](osTicket3.png)

Al utilizar las credenciales encontradas en el servicio de SSH, hemos tenido éxito y logramos obtener una shell, lo que nos brinda acceso al sistema remoto.

```bash
❯ ssh maildeliverer@10.10.10.222
The authenticity of host '10.10.10.222 (10.10.10.222)' can't be established.
ED25519 key fingerprint is SHA256:AGdhHnQ749stJakbrtXVi48e6KTkaMj/+QNYMW+tyj8.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.222' (ED25519) to the list of known hosts.
maildeliverer@10.10.10.222's password: 
Linux Delivery 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jul 25 12:21:11 2023 from 10.10.14.138
maildeliverer@Delivery:~$ id
uid=1000(maildeliverer) gid=1000(maildeliverer) groups=1000(maildeliverer)
```

En este punto, podemos obtener la flag `user.txt`.

```bash
maildeliverer@Delivery:~$ cat user.txt 
6162567b03249584****************
```

## Escalación de privilegios

---

Después de una exhaustiva enumeración, encontramos las credenciales para la base de datos de osTicket en la ruta `/var/www/osticket/upload/include/ost-config.php`.

```bash
maildeliverer@Delivery:/var/www/osticket/upload/include$ cat ost-config.php
<?php

[...]

# Encrypt/Decrypt secret key - randomly generated during installation.
define('SECRET_SALT','nP8uygzdkzXRLJzYUmdmLDEqDSq5bGk3');

#Default admin email. Used only on db connection issues and related alerts.
define('ADMIN_EMAIL','maildeliverer@delivery.htb');

# Database Options
# ---------------------------------------------------
# Mysql Login info
define('DBTYPE','mysql');
define('DBHOST','localhost');
define('DBNAME','osticket');
define('DBUSER','ost_user');
define('DBPASS','!H3lpD3sk123!');

[...]
```

Listamos los procesos con el nombre "**mattermost**" e identificamos la ruta donde se encuentra montado el servidor (`/opt/mattermost/`).

```bash
maildeliverer@Delivery:~$ ps aux | grep -i mattermost
matterm+   707  0.0  3.7 1575864 150492 ?      Ssl  02:43   0:41 /opt/mattermost/bin/mattermost
matterm+  4291  0.0  0.5 1235572 21008 ?       Sl   12:03   0:00 plugins/com.mattermost.plugin-channel-export/server/dist/plugin-linux-amd64
matterm+  4298  0.0  0.6 1239060 27252 ?       Sl   12:03   0:02 plugins/com.mattermost.nps/server/dist/plugin-linux-amd64
maildel+ 28489  0.0  0.0   6208   880 pts/0    S+   20:20   0:00 grep -i mattermost
```

En esa ruta, encontramos la configuración y credenciales de la base de datos de **MatterMost**.

```bash
maildeliverer@Delivery:/opt/mattermost/config$ cat config.json

[...]

"SqlSettings": {
        "DriverName": "mysql",
        "DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8\u0026readTimeout=30s\u0026writeTimeout=30s",
        "DataSourceReplicas": [],
        "DataSourceSearchReplicas": [],
        "MaxIdleConns": 20,
        "ConnMaxLifetimeMilliseconds": 3600000,
        "MaxOpenConns": 300,
        "Trace": false,
        "AtRestEncryptKey": "n5uax3d4f919obtsp1pw1k5xetq1enez",
        "QueryTimeout": 30,
        "DisableDatabaseSearch": false

[...]
```

Utilizando las credenciales encontradas, nos conectamos a **MySQL** y procedemos a enumerar las bases de datos disponibles.

```bash
maildeliverer@Delivery:/opt/mattermost/config$ mysql -u mmuser -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 411
Server version: 10.3.27-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mattermost         |
+--------------------+
2 rows in set (0.000 sec)
```

Enumeramos las tablas de la base de datos `mattermost`.

```bash
MariaDB [(none)]> use mattermost
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [mattermost]> show tables;
+------------------------+
| Tables_in_mattermost   |
+------------------------+
| Audits                 |
| Bots                   |
| ChannelMemberHistory   |
| ChannelMembers         |
| Channels               |
| ClusterDiscovery       |
| CommandWebhooks        |
| Commands               |
| Compliances            |
| Emoji                  |
| FileInfo               |
| GroupChannels          |
| GroupMembers           |
| GroupTeams             |
| IncomingWebhooks       |
| Jobs                   |
| Licenses               |
| LinkMetadata           |
| OAuthAccessData        |
| OAuthApps              |
| OAuthAuthData          |
| OutgoingWebhooks       |
| PluginKeyValueStore    |
| Posts                  |
| Preferences            |
| ProductNoticeViewState |
| PublicChannels         |
| Reactions              |
| Roles                  |
| Schemes                |
| Sessions               |
| SidebarCategories      |
| SidebarChannels        |
| Status                 |
| Systems                |
| TeamMembers            |
| Teams                  |
| TermsOfService         |
| ThreadMemberships      |
| Threads                |
| Tokens                 |
| UploadSessions         |
| UserAccessTokens       |
| UserGroups             |
| UserTermsOfService     |
| Users                  |
+------------------------+
46 rows in set (0.001 sec)
```

Enumeramos las columnas de la tabla `Users`.

```bash
MariaDB [mattermost]> describe Users;
+--------------------+--------------+------+-----+---------+-------+
| Field              | Type         | Null | Key | Default | Extra |
+--------------------+--------------+------+-----+---------+-------+
| Id                 | varchar(26)  | NO   | PRI | NULL    |       |
| CreateAt           | bigint(20)   | YES  | MUL | NULL    |       |
| UpdateAt           | bigint(20)   | YES  | MUL | NULL    |       |
| DeleteAt           | bigint(20)   | YES  | MUL | NULL    |       |
| Username           | varchar(64)  | YES  | UNI | NULL    |       |
| Password           | varchar(128) | YES  |     | NULL    |       |
| AuthData           | varchar(128) | YES  | UNI | NULL    |       |
| AuthService        | varchar(32)  | YES  |     | NULL    |       |
| Email              | varchar(128) | YES  | UNI | NULL    |       |
| EmailVerified      | tinyint(1)   | YES  |     | NULL    |       |
| Nickname           | varchar(64)  | YES  |     | NULL    |       |
| FirstName          | varchar(64)  | YES  |     | NULL    |       |
| LastName           | varchar(64)  | YES  |     | NULL    |       |
| Position           | varchar(128) | YES  |     | NULL    |       |
| Roles              | text         | YES  |     | NULL    |       |
| AllowMarketing     | tinyint(1)   | YES  |     | NULL    |       |
| Props              | text         | YES  |     | NULL    |       |
| NotifyProps        | text         | YES  |     | NULL    |       |
| LastPasswordUpdate | bigint(20)   | YES  |     | NULL    |       |
| LastPictureUpdate  | bigint(20)   | YES  |     | NULL    |       |
| FailedAttempts     | int(11)      | YES  |     | NULL    |       |
| Locale             | varchar(5)   | YES  |     | NULL    |       |
| Timezone           | text         | YES  |     | NULL    |       |
| MfaActive          | tinyint(1)   | YES  |     | NULL    |       |
| MfaSecret          | varchar(128) | YES  |     | NULL    |       |
+--------------------+--------------+------+-----+---------+-------+
25 rows in set (0.001 sec)
```

Para obtener los datos de las columnas `Username` y `Password` de la tabla `Users`, procedemos a realizar un volcado de la información desde la base de datos.

```bash
MariaDB [mattermost]> select Username, Password from Users;
+----------------------------------+--------------------------------------------------------------+
| Username                         | Password                                                     |
+----------------------------------+--------------------------------------------------------------+
| surveybot                        |                                                              |
| c3ecacacc7b94f909d04dbfd308a9b93 | $2a$10$u5815SIBe2Fq1FZlv9S8I.VjU3zeSPBrIEg9wvpiLaS7ImuiItEiK |
| 5b785171bfb34762a933e127630c4860 | $2a$10$3m0quqyvCE8Z/R1gFcCOWO6tEj6FtqtBn8fRAXQXmaKmg.HDGpS/G |
| itzjp                            | $2a$10$x1r/xJPi0iXi9DDIV2Gn0OEBzjzYJF4VBtcVdXUtyb.UOPadGnqBe |
| root                             | $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO |
| ff0a21fc6fc2488195e16ea854c963ee | $2a$10$RnJsISTLc9W3iUcUggl1KOG9vqADED24CQcQ8zvUm1Ir9pxS.Pduq |
| channelexport                    |                                                              |
| 9ecfb4be145d47fda0724f697f35ffaf | $2a$10$s.cLPSjAVgawGOJwB7vrqenPg2lrDtOECRtjwWahOzHfq1CoFyFqm |
| test                             | $2a$10$vz4bSJdfOqdoP7EeBEoFmuv70GQ528USyMmuWfpSdS1ogNU/k2rBi |
| itzjp_sec                        | $2a$10$pQLrudMKuQtnTEzc/Oqwfu4Yo843mTBkCcacKOIubgvrfOe2U5TUO |
+----------------------------------+--------------------------------------------------------------+
10 rows in set (0.001 sec)
```

Obtenemos el hash del usuario `root`, y según los ejemplos de `hashcat`, parece que se trata de un hash en formato `bcrypt`.

![hash](hash.png)

El usuario `root` dio a entender que reutilizaban contraseñas y mencionó `PleaseSubscribe!`. Por lo tanto, guardamos esta información en un archivo para luego aplicar reglas de `hashcat`. También almacenamos el hash del usuario `root`.

```bash
❯ echo 'PleaseSubscribe!' > password.txt
❯ echo '$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO' > hash_root.txt
```

Ahora utilizamos `hashcat` junto con la regla `best64.rule` para crear un diccionario a partir de la palabra `PleaseSubscribe!`. De esta manera, intentaremos descifrar el hash del usuario `root`.

```bash
C:\Users\juanr\OneDrive\Documentos\Hashcat\hashcat-6.2.6>hashcat.exe -m 3200 -a 0 hash_root.txt password.txt -r rules\best64.rule
hashcat (v6.2.6) starting

Successfully initialized the NVIDIA main driver CUDA runtime library.

[...]

$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO:PleaseSubscribe!21

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
Hash.Target......: $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v...JwgjjO
Time.Started.....: Tue Jul 25 20:11:33 2023 (11 secs)
Time.Estimated...: Tue Jul 25 20:11:44 2023 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (password.txt)
Guess.Mod........: Rules (rules\best64.rule)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:        0 H/s (0.00ms) @ Accel:2 Loops:4 Thr:16 Vec:1
Speed.#2.........:        2 H/s (8.06ms) @ Accel:2 Loops:16 Thr:11 Vec:1
Speed.#*.........:        2 H/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 21/77 (27.27%)
Rejected.........: 0/21 (0.00%)
Restore.Point....: 0/1 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-0 Iteration:0-4
Restore.Sub.#2...: Salt:0 Amplifier:20-21 Iteration:1008-1024
Candidate.Engine.: Device Generator
Candidates.#1....: [Copying]
Candidates.#2....: PleaseSubscribe!21 -> PleaseSubscribe!21
Hardware.Mon.#1..: Util:  0% Core: 400MHz Mem:1600MHz Bus:16
Hardware.Mon.#2..: Temp: 51c Util: 99% Core:1957MHz Mem:5989MHz Bus:8

Started: Tue Jul 25 20:11:20 2023
Stopped: Tue Jul 25 20:11:46 2023
```

Una vez tenemos la contraseña en texto plano, procedemos a cambiarnos al usuario `root`.

```bash
maildeliverer@Delivery:/opt/mattermost/config$ su root
Password: 
root@Delivery:/opt/mattermost/config# whoami
root
```

Finalmente, conseguimos la flag `root.txt`.

```bash
root@Delivery:/opt/mattermost/config# cat /root/root.txt 
ee3a35962f043e70****************
```

!Happy Hacking¡
