---
layout: post
toc: true
title: "Celestial - HackTheBox Writeup"
categories: HackTheBox
tags: [CTF, Web]
author:
  - H4RRIZN
---
![altered](https://i.imgur.com/yv65Ax1.png)

## Reconocimiento inicial
Como siempre comenzamos con un scan para detectar los puertos abiertos en el host victima:
```bash
nmap -sS --min-rate 1500 -p- --open -n -Pn 10.10.10.85
```
![nmap1](https://i.imgur.com/EyTBkW7.png)

Como se aprecia solo se encuentra el puerto 3000 abierto. Realzamos otro scan para obtener los servicios y versiones que se están utilizando en ese puerto:    
```bash
nmap -sCV -p3000 10.10.10.85
```
![nmap2](https://i.imgur.com/KfYp1jD.png)

Y observamos que corresponde a un servicio web en el cual se esta utilizando **`Node.js Express framework`.** Con esto presente ingresamos a la web para visualizar el contenido de esta.
## Reconocimiento web
Vemos que al ingresar se muestra el mensaje **`Hey Dummy 2+2 is 22`**:
![web1](https://i.imgur.com/LgWHvxt.png)
Luego de enumerar un poco notamos que la web no tiene nada más que esto. Observando las cookies de sesión encontramos un valor parecido a un **`JWT`** pero en realidad es un valor en base64 ya que no contiene los puntos de la firma de un **`JWT`** normal:    
![fakejwt](https://i.imgur.com/Knw19a1.png)
Al decodificar el valor en base64 notamos que es un objeto que esta serializado:     
![modifiedparam](https://i.imgur.com/1xXTIhm.png)
Si tomamos estos valores y los modificamos como por ejemplo el valor de **`username`** y lo enviamos en la cookie de sesión podemos ver que efectivamente cambian:     
![changeok](https://i.imgur.com/bX1FGtA.png)
![changed](https://i.imgur.com/Z0tQV3Q.png)
### Node JS Deserialization Attack
Buscando damos con el siguiente post:     
[Exploiting Node.js deserialization bug for Remote Code Execution](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/)     

El cual explica un método en el cual es posible realizar una ejecución remota de comandos mediante la explotación de deserialización insegura en node.js. En el cual se nos explica que:      
> Los datos no confiables pasados a la función **`unserialize()`** en el módulo **`node-serialize`** pueden ser explotados para lograr la ejecución arbitraria de código pasando un objeto JavaScript serializado con una expresión de función invocada inmediatamente (**`IIFE`**) **Immediately invoked function expression**.
>

### → Construcción del Payload
Para construir el payload es necesario contar con la librería de node.js **`node-serialize`.** Está la podemos instalar con **`npm`**:    
![nodeserialize](https://i.imgur.com/3JZMwX5.png)
Al ejecutarlo observamos que nos crea el objeto:     
![obcreated](https://i.imgur.com/TQAvIaM.png)

Como menciona el post, Ahora tenemos una cadena serializada que puede ser deserializada con la función **`unserialize()`**. Pero el problema es que la ejecución del código no ocurrirá hasta que disparemos la función correspondiente a la propiedad rce del objeto.
> Más tarde descubrí que podemos usar la expresión de función invocada inmediatamente (IIFE) de JavaScript para llamar a la función. Si usamos el doble paréntesis de IIFE **`()`** después del cuerpo de la función, la función será invocada cuando el objeto sea creado. Funciona de forma similar a un constructor de clase en C++.
>

Ahora se llama a la función **`serialize()`** con el código del objeto modificado:     
![iife](https://i.imgur.com/xFJQu1B.png)
La vulnerabilidad en la aplicación web es que lee la cookie de la petición HTTP, realiza la decodificación base64 del valor de la cookie y lo pasa a la función **`unserialize()`**. Como la cookie es una entrada no confiable, un atacante puede crear un valor de cookie malicioso para explotar esta vulnerabilidad.

## Shell as Sun
Para agilizar la explotación podemos utilizar la herramienta **[nodejsshell.py](https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py)** para obtener una reverse shell:     
![shellcommand](https://i.imgur.com/AmB3Dmk.png)
Copiamos el payload entregado y lo reemplazamos en el primer script de la siguiente forma:     
![shellreplaced](https://i.imgur.com/qD8KDZG.png)
Desde ya nos ponemos en escucha:     
![nclisten](https://i.imgur.com/g42VcSr.png)
Y codificamos el payload en base64 para enviarlo en la cookie:      
![codedpayload](https://i.imgur.com/7ssQEVv.png)
![cookiereplace](https://i.imgur.com/7jda20N.png)
Observamos que ya tenemos acceso:    
![shellrec](https://i.imgur.com/9KgAmt8.png)
## Privesc
Al recibir la conexión notamos que estamos en el directorio del usuario sun. Dentro de este notamos el fichero **`output.txt`**:    
![sunfolder](https://i.imgur.com/Vz6Drjx.png) 
El cual si observamos podemos notar el mensaje de: **`Script is running...`** Pero no sabemos que script. Además notamos que el output es generado por el usuario **`root`**:    
![rootfile](https://i.imgur.com/ptm8Pn6.png)
### Detectando procesos con Pspy
En base al fichero detectado podemos pensar que se esta ejecutando un script que no podemos detectar. Para esto utilizaremos la herramienta **[pspy](https://github.com/DominicBreuker/pspy)** la cual debemos transferir a la maquina victima:     
![psytransfered](https://i.imgur.com/4JJm9SW.png)
Le damos permisos de ejecución y la ejecutamos. Y luego de esperar por un rato podemos observar que el UID 0 correspondiente al usuario root, está ejecutando el siguiente comando:     
![pspydetect](https://i.imgur.com/o4HET9z.png)
Vemos que root ejecuta el script **`script.py`** en el directorio **`Documents`** del usuario **`sun`** para luego depositar el fichero **`output.txt`.**  Al observar el contenido del script podemos ver que simplemente ejecuta un print de “**`Script is running…`**” por lo que realmente no hace nada. Pero lo curioso de esto es que el propietario del script es el usuario que ya tenemos comprometido:     
![script](https://i.imgur.com/ilICJzG.png)
En base a esto podemos modificar el script para que se cambie el permiso de la **`/bin/bash`** a **`SUID`**:      
![change](https://i.imgur.com/qGoLPlE.png)
Ahora debemos esperar a que el usuario root ejecute el script para poder cambiar el permiso de la **`/bin/bash`** y escalar privilegios al usuario root:   
![suid](https://i.imgur.com/pdiaorD.png)

## Shell as root
Ya con permiso SUID en la bash podemos utilizar el parámetro -p para ejecutarla con permisos de root:     
![pwned](https://i.imgur.com/kRLRlSa.png)
Con esto ya podemos leer la flag del usuario root.

Pwned! 🏴‍☠️