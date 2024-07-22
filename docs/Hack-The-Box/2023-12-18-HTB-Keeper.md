---
layout: post
title: Keeper
parent: HackTheBox
---
# Keeper
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
{:.lead}
![](/assets/img/HTB_Keeper/K0.png)

## Fase de Reconocimiento

Comenzaremos utilizando la herramienta **nmap**, con la que descubriremos los puertos:
```bash
nmap -sCV -T5 --open -Pn 10.10.11.227
```
- 22: SSH
- 80: HTTP

Si nos vamos al servicio web, veremos lo siguiente:
![](/assets/img/HTB_Keeper/K1.png)

Si vamos a esta pagina, nos saldra un error 404, esto debido a que necesitamos hacer un virtual hosting a la direccion que nos marca, esto lo haremos con:

```bash
#Estando como usuario root
echo "10.10.11.227 tickets.keeper.htb | tee -a /etc/hosts"
```
Una vez hecho esto, podremos ver un panel de autenticacion:
![](/assets/img/HTB_Keeper/K2.png)
Si buscamos esta heramienta en la web, podremos encontrar credenciales por defecto:
![](/assets/img/HTB_Keeper/K3.png)
Si las ingresamos podremos ver el panel de administracion:
![](/assets/img/HTB_Keeper/K4.png)
Si vemos en la parte del navbar del header, podremos apreciar un apartado "Admin", donde encontraremos 2 **usuarios**.
![](/assets/img/HTB_Keeper/K5.png)
Si vemos al usuario **Inorgaard**, podremos encontrar credenciales:
![](/assets/img/HTB_Keeper/K6.png)
Si recordamos, estaba el servicio SSH activo, por lo que trataremos de ingresar.
![](/assets/img/HTB_Keeper/K7.png)
Una vez dentro, podremos apreciar un comprimido, por lo que lo pasaremos a nuestra maquina de atacante:
![](/assets/img/HTB_Keeper/K8.png)
![](/assets/img/HTB_Keeper/K9.png)
Podemos apreciar que es un dump y una base de datos para **Keepass**

## Rompiendo el Keepass

Si buscamos por la web, encontraremos el siguiente [CVE](https://github.com/vdohney/keepass-password-dumper), el cual aplica a la version superior 2, este nos va a dar una password aproximada de la base de datos.
Una vez usado el exploit, encontraremos un aproximado de la passwd:
![](/assets/img/HTB_Keeper/K10.png)
Si damos una busqueda rapida, encontraremos un poste con un nombre similar:
![](/assets/img/HTB_Keeper/K11.png)
Ahora bien, trataremos de entrar al archivo ""**passcodes.kdbx**""
Una vez entramos, podemos ver que el usuario **ROOT** tiene dentro de notas una clave PuTTY, por la que la usaremos para generar una clave **OpenSSH** y poder ingresar por el puerto SSH de la maquina
![](/assets/img/HTB_Keeper/K12.png)
Ahora, empleando **PuttyGen**, lograremos descubrir la id_rsa para entrar sin necesidad de proporcionar una password de autenticacion.
Usaremos el siguiente comando:
```bash
puttygen ticket.putty -O private-openssh -o id_rsa
```
![](/assets/img/HTB_Keeper/K13.png)
Por lo que ahora nos quedaria darle permiso de lectura y escritura (600) por parte del owner:
```bash
chmod 600 id_rsa
```
![](/assets/img/HTB_Keeper/K14.png)

# FIN
