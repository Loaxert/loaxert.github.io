---
title: "SpookyPass"
date: "2025-03-29T17:00:00-05:00"
type: "challenges"
draft: false
tags: ["Reversing", "ltrace", "ghidra", "radar2"]
---

En este caso vamos a resolver un reto de reversing bastante sencillo que se puede resolver de diferentes formas.

Después de descargar el archivo ZIP y descomprimirlo encontraremos un ejecutable de Linux que si tratamos de ejecutar nos pedirá aun contraseña por lo que ya podemos pensar que nuestra tarea va a ser encontrar esta contraseña a continuación mostraremos tres formas diferentes de hacerlo:

# Ltrace

Si el ejecutable nos esta pidiendo una contraseña lo primero que tenemos que analizar es que esta contraseña esta en el ejecutable y además lo esta comparando con el input del usuario por lo que usando ltrace vamos a interceptar llamadas a funciones de bibliotecas compartidas depurando así la ejecución de la aplicación:

<a href="https://imgur.com/4QaDUDK"><img src="https://i.imgur.com/4QaDUDK.png" title="source: imgur.com" /></a>

Vemos que esta comparando nuestro input con la cadena `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul` pero si tratamos de ingresar esto como contraseña nos dirá que es errónea lo que podemos hacer es verificar las cadenas en texto claro del ejecutable usando `strings` de la siguiente forma:

<a href="https://imgur.com/uBYMeSX"><img src="https://i.imgur.com/uBYMeSX.png" title="source: imgur.com" /></a>

Por lo que nos estaba faltando un ultimo carácter que es el 5 ya si ingresamos esto podremos ver la flag para entregar en Hack The Box.

# Radare2 y Ghidra

Si bien son herramientas diferentes consisten en lo mismo en este caso, vamos a analizar la función `main` del archivo y encontrar la comparación que hacen con nuestro input. Primero lo vamos a hacer con `radare2` y ya después mostraremos `ghidra`.

<a href="https://imgur.com/ojc49ki"><img src="https://i.imgur.com/ojc49ki.png" title="source: imgur.com" /></a>

Acá lo que estamos haciendo es mirar las funciones existentes en el ejecutable donde podemos observar esta el `main` incluido por lo que lo vamos a revisar mas a fondo de la siguiente manera:

<a href="https://imgur.com/YRxs38E"><img src="https://i.imgur.com/YRxs38E.png" title="source: imgur.com" /></a>

Lo que hace esto es mostrarnos que esta haciendo el ejecutable a mas bajo nivel y si miramos un poco encontraremos tanto la flag como la contraseña entre la salida que nos mostro el comando `pdf @main` .

Para `ghidra` toca hacer unos cuantos pasos mas pero podemos ver con mayor claridad esto, vamos a abrir la aplicación y vamos a crear un nuevo proyecto después de esto vamos a importar el ejecutable que descargamos de la pagina de Hack The Box, luego de esto vamos a arrastrar ese ejecutable al símbolo de dragón que hay arriba a la izquierda:

<a href="https://imgur.com/Su1Hx4q"><img src="https://i.imgur.com/Su1Hx4q.png" title="source: imgur.com" /></a>

Ya con esto a la izquierda veremos un árbol de funciones por lo que si ingresamos a main veremos la contraseña de esta forma:

<a href="https://imgur.com/j3lhPEs"><img src="https://i.imgur.com/j3lhPEs.png" title="source: imgur.com" /></a>