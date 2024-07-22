---
layout: default
title: JABITA
parent: Hack My VM
---
# JABITA
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
{:.lead}
<p style="text-align: center;"> JABITA - HackMyVM</p>
![](/assets/img/HackMyVM_JABITA/logo.png)

## Fase de Reconocimiento:

Bueno, esta maquina viene de [HackMyVM](https://hackmyvm.eu/), por lo que tendremos que emularla e introducirla dentro de nuesta red local.

> Para ver los dispositivos en nuestro segmento de red, podremos usar multiples utilidades que vienen por defecto en nuestro sistema.

En este caso usare **arp-scan**:

```bash
arp-scan -l
```
>>> Nota rapida, no olvidemos ejecutar este comando como sudo.

![](/assets/img/HackMyVM_JABITA/J1.png)

Una vez descubierta la direccion IP, procederemos al **analisis de los puertos abiertos**.

> No olvidar que para descubrir el sistema operativo mucho antes de usar herramientas como **nmap**, es basarnos en el TTL obtenido al lanzar un **ping**; el **TTL comunmente de Linux** es <= 64

![](/assets/img/HackMyVM_JABITA/J2.png)

Como podemos ver, tenemos el puerto 80 y el 22 abiertos:

- El **Puerto 80**: Usado para el servicio HTTP.
- El **Puerto 22**: Usado para el servicio SSH.

Acto seguido, vamos a ver que se encuentra tas el servicio HTTP:

![](/assets/img/HackMyVM_JABITA/J3.png)

Podemos apreciar que se trata de una web en "construccion", pero no venimos a ver eso, venimos a **~~hackear~~** divertirnos, por lo que aplicaremos un **directory fuzzing.**

![](/assets/img/HackMyVM_JABITA/J4.png)

Como podemos apreciar, hemos descubierto un directorio: **building**

Si vamos a este directorio podemos ver algo interesante al momento de pasar el raton sobre cualquier widget presente en la barra superior:

![](/assets/img/HackMyVM_JABITA/J5.png)

Podemos ver un posible **LFI**:

![](/assets/img/HackMyVM_JABITA/J6.png)

Por lo que apuntaremos a rutas que nos filtren informacion necesaria:

![](/assets/img/HackMyVM_JABITA/J7.png)

Como podemos apreciar, vemos que efectivamente existe un **LFI**, entonces apuntaremos a rutas claves del sistema:

> **Algunas rutas claves son:** /etc/shadow, /etc/hosts, /etc/passwd... Esto debido a que en el la ruta **/etc** de los sistemas **Linux** es dond podremos encontrar mayoria de configuraciones, tanto del sistema, como dealgunas herramientas presentes.

![](/assets/img/HackMyVM_JABITA/J8.png)

Como podemos observar, al momento de listar el /etc/shadow, podemos encontrar las passwords de los usuarios, solo que encodeadas, por lo que tendremos que romperlas.

Usando la herramienta John the Ripper, y valiendonos del diccionario **"rockyou"** obtendremos la password del usuario jack:

![](/assets/img/HackMyVM_JABITA/J9.png)

Por lo que ahora nos quedaria entrar por SSH.

![](/assets/img/HackMyVM_JABITA/J10.png)

Ahora nos queda listar binarios o permisos que nos permitan escalar nuestros privilegios para llegar al super user.

Si buscamos permisos a nivel de sudoers, podemos toparnos con que podemos usar el comando **'awk'** como el ususario **jaba**, por lo que pivotaremos a su shell de la siguiente forma:

```bash
sudo -u jaba /usr/bin/awk 'BEGIN {system("/bin/sh")}'
```
![](/assets/img/HackMyVM_JABITA/J11.png)

Si buscamos permisos para este usuario, nos encontraremos con que podemos correr un comando en **Python**, asi que vamos por partes:

![](/assets/img/HackMyVM_JABITA/J12.png)

- Vemos que tenemos permisos de ejecucion para un binario en lenguaje **Python**
- El binario llama a una funcion: "wild.py"
- Procederemos a buscar la funcion y vemos que es un permiso: **646**, esto es interesante, puesto que nos va a permitir escribir sobre el perteneciendo al grupo de **OTHERS**

![](/assets/img/HackMyVM_JABITA/J13.png)

Yo en este caso, dare permisos de ejecucion al binario de **bash**, tu puedes invocar una shell si deseas:

```python
import os;
os.system("chmod u+s /bin/bash")
```

![](/assets/img/HackMyVM_JABITA/J14.png)

## Entendiendo la vulnerabilidad

Como vimos, aprovechamos una vulnerabilidad interesante, dentro de esta maquina, al igual que en otros **posts** hemos tocado, esta vulnerabilidad nos permite listar ficheros interesantes del servidor:

![](/assets/img/HackMyVM_JABITA/J15.png)

La vulnerabilidad yace en el codigo **php**:

```php
<?php
  include($_GET['page'])
?>
```

Eso es porque al no estar sanitizado, ya sea con una regex o similares, la funcion buscara el archivo en concreto, ya sea dentro de su directorio, o de otros... Y, por ende, es posible esta vulnerabilidad.

# FIN
