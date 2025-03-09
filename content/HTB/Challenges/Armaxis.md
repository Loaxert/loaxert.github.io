---
title: "Armaxis"
date: "2025-02-21T17:00:00-05:00"
type: "challenges"
draft: false
tags: ["Web", "Insecure Direct Object Reference (IDOR)", "Markdown Injection"]
---

Tenemos una url con dos puertos por lo que vamos a visitar los dos a ver que contienen.

<a href="https://imgur.com/INILlbn"><img src="https://i.imgur.com/INILlbn.png" title="source: imgur.com" /></a>
<a href="https://imgur.com/0DFM6yL"><img src="https://i.imgur.com/0DFM6yL.png" title="source: imgur.com" /></a>

Vemos como tenemos acceso a una bandeja de un correo y en el otro puerto tenemos la opcion de registrar usuarios ingresar y recuperar la contraseña.

Revisando el codigo fuente nos damos cuenta que la recuperacion de contraseña no esta bien pensada y es vulnerable a un IDOR porque si bien pide un token y verifica que el token sea valido no verifica a quien le pertenece el token, con este en conocimiento le podemos cambiar la contraseña a cualquier usuario.

<a href="https://imgur.com/NgmrWjW"><img src="https://i.imgur.com/NgmrWjW.png" title="source: imgur.com" /></a>

Vamos a tratar de hallar algun correo de administrador en el codigo fuente usando grep:

```bash
grep -r admin
```

<a href="https://imgur.com/Pz55VjH"><img src="https://i.imgur.com/Pz55VjH.png" title="source: imgur.com" /></a>

Solicitamos el cambio de contraseña despues de crear un usuario con el correo que nos dieron:

<a href="https://imgur.com/FpSi3nd"><img src="https://i.imgur.com/FpSi3nd.png" title="source: imgur.com" /></a>

Y hacemos el cambio de contraseña para el usuario administrador:

<a href="https://imgur.com/qw2z1SF"><img src="https://i.imgur.com/qw2z1SF.png" title="source: imgur.com" /></a>

Ahora con eso podemos iniciar sesion como administradores y veremos que tenemos acceso a un funcion para mostrar “weapons”

<a href="https://imgur.com/lAJ1HvJ"><img src="https://i.imgur.com/lAJ1HvJ.png" title="source: imgur.com" /></a>

Tambien vemos que en la nota acepta markdown por lo que podemos tratar de hacer una inyección para obtener la flag.

```markdown
img[!](file:///flag.txt)
```

<a href="https://imgur.com/qav1aoI"><img src="https://i.imgur.com/qav1aoI.png" title="source: imgur.com" /></a>
<a href="https://imgur.com/hRyzZEA"><img src="https://i.imgur.com/hRyzZEA.png" title="source: imgur.com" /></a>

Si pasamos lo que nos muestra el codigo fuente a un decoder de base64 nos dara la flag.
