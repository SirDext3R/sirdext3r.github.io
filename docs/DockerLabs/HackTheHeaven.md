---
layout: default
title: HackTheHeaven
parent: DockerLabs
---
# HackTheHeaven
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
# Fase de reconocimiento
Usamos nuestra herramienta de confianza para descubrir **puertos abiertos**{:.text-green-100}.
![](/assets/img/DockerLabs/HackTheHeaven/H0.png)

Veremos que solo disponemos un puerto abierto **HTTP**{:.text-red-000}, donde si vamos, veremos la presentacion de bienvenida de Apache. Por lo que nos quedaria aplicar un **WEBFUZZING**{:.text-yellow-200}.
![](/assets/img/DockerLabs/HackTheHeaven/H1.png)

Podremos encontrar cosas interesantes, como:
- Un **info.php**{:.text-green-100}.
- Un directorio llamado **idol.html**{:.text-green-100}.

## Encontrando cosas interesantes

Si vamos a la pagina de **idol.html**{:.text-red-000}, podremos visualizar un boton que nos redirige a un archivo php que parece que lee archivos.
![](/assets/img/DockerLabs/HackTheHeaven/H2.png)
![](/assets/img/DockerLabs/HackTheHeaven/H3.png)

Por lo que ahora nos queda descubrir que **parametro** se usa para leer archivos, basicamente intentaremos encontrar un **LFI**.
![](/assets/img/DockerLabs/HackTheHeaven/H4.png)

Logramos dar con el parametro que se usa para leer archivos, por lo que intentaremos dar con un **LFI**{:.text-yellow-200}.

Si bien existe una regla que evita que leamos archivos directamente, podemos saltarla concatenando unos caracteres adicionales.
![](/assets/img/DockerLabs/HackTheHeaven/H5.png)

## Descubriendo el LFI2RCE

Si bien, buscamos por nuestra [**BIBLIA HACKER**](https://book.hacktricks.xyz/pentesting-web/file-inclusion/lfi2rce-via-phpinfo){:.text-red-300}, encontraremos que existe una forma para para ejecutar comandos aprovechando el LFI encontrado.
Este nos pide algunas condiciones que podremos observar dentro del **info.php**
![](/assets/img/DockerLabs/HackTheHeaven/H6.png)

Nos toparemos que si se cumplen las condiciones para lograr esto, por lo que leyendo la teoria se nos explica que cada que subimos un archivo, este se aloja en el directorio temporal hasta que se termine de procesar la peticion, luego este se elimina. Por lo que trataremos de subir un archivo con un script malicioso que nos permita la ejecucion de comandos.

Usare este archivo brindado en hacktricks: [lfito_rce](https://www.insomniasec.com/downloads/publications/phpinfolfi.py){:.bg-green-100}.

![](/assets/img/DockerLabs/HackTheHeaven/H7.png)

# Usuario Xerosec

Si buscamos dentro de las carpetas, encontraremos una nota destinada para el usuario **MARIO**{:.text-green-100}, por parte del usuario **XEROSEC**{:.text-red-100}, dicho fichero contiene cosas interesantes.
![](/assets/img/DockerLabs/HackTheHeaven/H8.png)

Si probamos el curioso texto que tiene como la passwd del usuario **xero**{:.text-red-100}, podremos ganar acceso, por lo que ahora tendremos que buscar una forma de logar pivotar a otros usuarios.

Si comprobamos permisos del usuario, podremos ver que tenemos la capacidad de ejecutar un comando como el usuario **MARIO**.
![](/assets/img/DockerLabs/HackTheHeaven/H9.png)

Si vemos el script, podremos ver que importa la libreria hashlib.
```python
import hashlib

if __name__ == '__main__':
    cadena = input("Introduce la cadena: ")
    hash_md5 = hashlib.md5(cadena.encode()).hexdigest()
    print("El hash MD5 de la cadena es:", hash_md5)
```

# Usuario Mario
Por lo que podemos crear un archivo que se importe en vez de la libreria, esto lo hacemos creando un archivo py que se llame igual
```python
#Archivo hashlib.py 
import os 
os.system("/bin/bash")
```
Por lo que si ejecutamos el permiso de sudoers, obtendemos acceso al usuario **mario**. Una vez dentro, podremos ver que existe una nota dentro del home, esta nos habla que existe un usuario con un servicio web detras, el usuario **s4vitar**{:.text-yellow-200}.
![](/assets/img/DockerLabs/HackTheHeaven/H10.png)

Si usamos el comando **ps**, podremos ver que efectivamente el usuario **s4vitar** posee un servicio php corriendo en local.
![](/assets/img/DockerLabs/HackTheHeaven/H11.png)
Por lo que podemos usar el comando **CURL** para poder mandar comandos.
# Usuario s4vitar
Como podemos observar en la imagen anterior, podremos darnos cuenta que existe una shell que podemos usar en localhost, por lo que podemos tirarnos una shell usando **URLENCONDE**:
![](/assets/img/DockerLabs/HackTheHeaven/H12.png)
Una vez ganada la shell, podremos observar como se hizo esta utilidad:
```php
<?php
// Verifica si la solicitud proviene de la máquina local
if ($_SERVER['REMOTE_ADDR'] == '127.0.0.1') {
    // Verifica si se ha pasado un comando como parámetro
    if (isset($_GET['cmds4vi'])) {
        $comando = $_GET['cmds4vi'];
        // Ejecuta el comando y obtén la salida
        $salida = shell_exec($comando);
        // Imprime la salida del comando
        echo "$salida";
    } else {
        echo "No se ha pasado ningun comando.";
    }
} else {
    // Si la solicitud no proviene de la máquina local, devuelve un error
    header("HTTP/1.1 403 Forbidden");
    echo "Acceso denegado.";
}
?>
```
Ahora bien, si vemos los permisos sudoers, podremos apreciar que podemos ejecutar el comando **xargs**{:.text-yellow-000}
![](/assets/img/DockerLabs/HackTheHeaven/H13.png)
## Usuario ROOT
Aprovechando este privilegio de **SUDOERS**, podremos encontrar que existe una forma de irnos al usuario **root**, podremos encontrar el comando aqui [**GTFOBins**](https://gtfobins.github.io/){:.text-blue-100}.
```bash
sudo xargs -a /dev/null sh
```
Una vez hecho eso, habremos completado exitosamente la maquina.
![](/assets/img/DockerLabs/HackTheHeaven/H14.png)
# FIN
