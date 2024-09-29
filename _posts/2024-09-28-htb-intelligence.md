---
layout: post
toc: true
title: "Intelligence - HackTheBox Writeup"
categories: HackTheBox
tags: [CTF, Active Directory, Windows]
author:
  - H4RRIZN
---

![logo](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/logo.png)

## Reconocimiento Inicial

Como siempre comenzamos con un scan de nmap hacia el host victima para identificar puertos abiertos:

![00](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/00.png)

Tenemos diversos puertos abiertos típicos de un AD. Lanzamos un nuevo scan para obtener más información de los servicios y versiones en estos puertos:

![0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/0.png)

Podemos determinar en base a la captura el nombre del dominio (dc.intelligence.htb) fuera de eso no vemos información más relevante. Añadiremos el dominio encontrado al fichero /etc/hosts:

![1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/1.png)

## Reconocimiento Web

Ingresamos al servidor web:

![2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/2.png)

Al investigar por la web no notamos nada particularmente interesante. Podemos visualizar 2 documentos disponibles `2020-01-01-upload.pdf` y `2020-12-15-upload.pdf` ubicados en `http://intelligence.htb/documents`:

![3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/3.png)

Los documentos no tienen nada interesante en si, solo texto genérico:

![4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/4.png)

Al descargarlos y analizar los metadatos con exiftool podemos identificar que tenemos 2 usuarios; William.Lee y Jose.Williams
```bash
exiftool documents.pdf
```

![5](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/5.png)

Vemos que al validar estos con crackmapexec no vemos si el usuario es valido o no, podemos utilizar Kerbrute para enumerar estos usuarios:

![6](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/6.png)

Como vemos son usuarios válidos, sin embargo no tenemos ninguna contraseña. Luego de agotar mis posibilidades buscando recursos en SMB con sesiones nulas decidí seguir buscando en el servidor web. Como pude obtener 2 usuarios validos de los PDF que descargue decidí revisar el directorio **`documents`:**

![7](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/7.png)

Como no es posible ingresar decidí construir un pequeño script en python el cual realice fuzzing sobre las fechas de los documentos con el fin de encontrar algún documento extra. (Es posible realizar esto con el intruder de Burp. Pero para fines prácticos decidí construir el script).

### Fuzzing PDFs

Sabemos que los documentos que tenemos `2020-01-01-upload.pdf` y `2020-12-15-upload.pdf` comparten el mismo año (2020). Entonces solo recorreré los meses y días:

```bash
import requests
import itertools
from concurrent.futures import ThreadPoolExecutor

base_url = "http://intelligence.htb/documents/"
year = "2020"

meses = ["{:02d}".format(m) for m in range(1, 13)]
dias = ["{:02d}".format(d) for d in range(1, 32)]
fechas = itertools.product(meses, dias)

total_documentos = 0

def buscar_documento(fecha):
    global total_documentos
    url = f"{base_url}{fecha}-upload.pdf"
    response = requests.get(url)
    if response.status_code == 200:
        total_documentos += 1
        print(f"Documento encontrado: {url}")

with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(buscar_documento, [f"{year}-{mes}-{dia}" for mes, dia in fechas])

print(f"Total de documentos encontrados: {total_documentos}")
```

Al ejecutar el script y esperar a que finalice vemos como resultado un total de 84 documentos:

![8](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/8.png)

Hemos encontrado 84 documentos disponibles, si bien el script nos muestra todas las URL disponibles seria mucho trabajo entrar una por una y revisar su contenido además de tener que descargarlos para luego extraer la metadata y crear una lista de usuario. Además es posible que estos documentos contengan información relevante como detalles de cuentas o incluso contraseñas. Pero nada que un poco de scripting no solucione. El [script final se encontrará en este repositorio](https://github.com/H4RRIZN/CTFRepo/tree/main/Hack%20The%20Box%20-%20Intelligence) para su uso libre:

![9](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/9.png)

Como se aprecia el resultado final del script muestra que en el documento **`2020-06-04-upload.pdf`** se ha encontrado un texto relacionado a nuevas cuentas con la contraseña **NewIntelligenceCorpUser9876.** En el documento **`2020-12-30-upload.pdf`** se habla de un usuario Ted que está almacenando un script en algún lugar.

### Creando una lista de usuarios

Tenemos una contraseña valida y previamente hemos encontrado usuarios validos en los metadatos de los documentos. Ahora que contamos con 84 documentos vale la pena buscar más usuarios. El script almacena los documentos en una carpeta llamada “documentos”:

![10](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/10.png)

podemos utilizar exiftool en combinación con expresiones regulares para crear una lista de usuarios:

```bash
exiftool *.pdf | grep "Creator" | awk '{print $3}' | sort -u | tee users
```

![11](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/11.png)

### Validación de usuarios con kerbrute

Observamos que al utilizar kerbrute para validar los usuarios todos los usuarios son validos en el sistema:

![12](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/12.png)

Con estos y una contraseña disponible podemos probar un ataque de Password Spraying para ver si la contraseña funciona para alguno de estos usuarios:

## Password Spraying

Para realizar el ataque utilizare kerbrute de la siguiente forma:

![13](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/13.png)

Y el usuario “Tiffany.Molina@intelligence.htb” es válido. El mensaje de (Clock slow is too great) es porque nuestra hora no esta sincronizada con la del host victima. Podemos actualizar está con ntpdate:

![14](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/14.png)

## SMB - Tiffany

Ya que tenemos credenciales realizare la enumeración del servicio smb:

![15](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/15.png)

Como vemos tenemos acceso de lectura a varias carpetas en las que destacan IT y Users. comenzare por observar el contenido de la carpeta Users. Al ingresar podemos ver que tenemos las carpetas de los usuarios en la que se encuentra la carpeta del usuario **Ted.Graves**. Recordemos que este está almacenando un script en algún lado.

```bash
smbclient //10.10.10.248/Users -U "Tiffany.Molina"
```

![16](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/16.png)

Vemos que no es posible listar el contenido de la carpeta del usuario Ted.Graves:

![17](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/17.png)

Al enumerar el recurso IT observamos un script en powershell llamado **downdetector.ps1**. Lo descargamos con get:

![18](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/18.png)

A continuación observamos el contenido del script:

![19](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/19.png)

Vemos que este script en PowerShell verifica el estado de los servidores web cuyos nombres comienzan con "web" en la zona DNS intelligence.htb. Finalmente Ted.Graves envía un correo electrónico si alguno de los servidores está caído. Teniendo todo esto en cuenta podemos utilizar la herramienta [dnstool.py](https://github.com/dirkjanm/krbrelayx/blob/master/dnstool.py) para realizar una consulta dns agregando un servidor web el cual no existe, con el fin de capturar la petición de la red con responder.

## Hash - Ted.Graves

Iniciaremos responder con la configuración por defecto:

```bash
responder -I tun0
```

![20](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/20.png)

A continuación realizaremos la consulta con dnstool de la siguiente forma:

![21](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/21.png)

Al esperar los 5 minutos como indica el script downdetector.ps1 podemos ver que responder capturó un hash Net-NTLMv2 del usuario Ted.Graves:

![22](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/22.png)

### Crackeando el hash

Procedemos a crackear el hash, utilizare hashcat y el diccionario de contraseñas rockyou:

```bash
hashcat -m 5600 ted_hash rockyou.txt
```

![23](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/23.png)

Ahora tenemos la contraseña **Mr.<REDACTED>** del usuario **Ted.Graves** 

## Enum - Ted Graves

Luego de buscar con smb en el directorio del usuario algo interesante no pude dar con nada. Asi que utilizare bloodhound-python para enumerar el dominio:

![24](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/24.png)

Obtendremos algunos ficheros .json los cuales deberemos comprimir para poder subir a BloodHound. En mi caso utilice zip:

![25](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/25.png)

Cargamos el zip a BloodHound y seleccionamos en el Nodo de inicio al usuario Ted.Graves y presionamos click derecho sobre este y seleccionamos la opción **Mark User as Owned:**

![26](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/26.png)

Ahora seleccionamos Shortest Path from Owned Principals en la pestaña de **Analysis**

![27](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/27.png)

A continuación observamos que pertenecemos al grupo ITSUPPORT.

![28](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/28.png)

Siguiendo el flujo podemos ver que SVC_INT$@INTELLIGENCE.HTB es una cuenta de servicio gestionada por grupo. El grupo ITSUPPORT@INTELLIGENCE.HTB puede recuperar la contraseña de la GMSA del usuario SVC_INT$@INTELLIGENCE.HTB.

![29](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/29.png)

Finalmente vemos que el usuario SVC_INT$@INTELLIGENCE.HTB tiene el privilegio de delegación restringida a DC.INTELLIGENCE.HTB.

![30](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/30.png)

## Obtención de hash NT de SVC_INT

Como sabemos gracias a bloodhound podemos leer la contraseña GMSA del usuario svc_int. Utilizare la herramienta [gMSADumper](https://github.com/micahvandeusen/gMSADumper) para este proposito:

```bash
python3 gMSADumper.py -u 'ted.graves' -p 'Mr.<REDACTED>' -d 'intelligence.htb'
```

![31](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/31.png)

## Shell as Admin

Siguiendo la ruta de BlooHound ahora podriamos solicitar un TGT para el usuario Administrador utilizando la herramienta getST de impacket:

```bash
impacket-getST -dc-ip 10.10.10.248 -spn www/dc.intelligence.htb -hashes :486b1ed222932<REDACTED> -impersonate administrator intelligence.htb/svc_int
```

![32](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/32.png)

Como vemos se ha almacenado el ticket en el fichero `administrator.ccache` podemos utilizar wmiexec de impacket para conectarnos realizando un “Pass The Hash” con el parametro -k. Lo primero que debemos hacer es exportar la variable `KRB5CCNAME` con el valor del ticket:

![33](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/33.png)

A continuación podemos realizar el procedimiento mencionado y leer la flag del usuario Administrator

![34](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CTFIMG/Intelligence/34.png)

Pwned! 🏴‍☠️