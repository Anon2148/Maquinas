Write-up máquina Twitx de plataforma [Vulnyx](https://vulnyx.com/#twitx)

## Antes de empezar...

Qué vamos a necesitar:

- Máquina atacante kali (virtualbox)
- Máquina victima Vulnyx (ova en virtualbox)
- Red NAT compartida entre ambas máquinas (aislamiento, configuración de virtualbox)

No estoy usando ninguna configuracion especial ni nada.
## Reconocimiento

Como en todo proceso de pentesting vamos a empezar con la fase de reconocimiento, donde intentaremos obtener la máxima información posible de la máquina para poder explotarla (vulnerarla). En este caso, la máquina víctima nos muestra ya su IP en la pantalla de inicio.

![IP_Victima](https://github.com/user-attachments/assets/34d0bb60-eca9-4744-b908-932de8d6ec7c)

Si ese no fuera el caso podriamos usar el siguiente comando:

```bash
sudo arp-scan --localnet | grep 08
```

Donde:

- `arp-scan`: Es un comando que viene instalado en kali para mandar paquetes arp a las máquinas de nuestra red
- `grep 08`: Vamos a usar el comando `grep` para filtrar por la cadena '08' ya que la MAC de las máquinas virtuales suelen empezar por defecto por 08:...

Teniendo la IP podemos pasar a realizar un reconocimiento de puertos y servicios abiertos en la máquina víctima, existen muchos comandos para llevar a cabo este proceso pero el comando por excelencia es `nmap`, este comando tiene muchas flags y modos distintos, dejaré que los consulten vosotros para ver que hacen. El comando que usaremos es el siguiente:

```bash
sudo nmap -p- -vvv -Pn -sS <IP-Victima> -oN targeted
```

La salida de este comando es la siguiente:

![nmapPrimerScan](https://github.com/user-attachments/assets/ec3fc8d9-cb17-4e5a-b88a-37dbd8dafd25)

Como podemos comprobar solo tenemos dos puertos abiertos:

- Puerto 80 -> servicio web
- Puerto 22 -> servicio ssh (SecureShell)

Vamos a realizar otro escaneo un poco más exhaustivo para ver si obtenemos más información de interes.

```bash
sudo nmap -sCV -p22,80 <IP-victima> -oN targeted2
```

Y el resultado del comando es el siguiente:

![nmapSegundoScan](https://github.com/user-attachments/assets/fac1b508-7d4e-408b-a4cb-1bb06a31d88a)

Sin embargo no nos muestra mucha más información de la que ya teniamos. Por lo tanto vamos a proceder a revisar que podemos obtener de los dos puertos encontrados. Empezamos visitando la página web alojada en el puerto 80. Al acceder nos muestran la página de inicio de un servidor Apache, hasta aquí nada interesante, lo mismo pasa con el puerto 22 que está protegido con contraseña por lo que vamos a dejar esta via a un lado y volveremos más tarde cuando encontremos credenciales.

## Escaneo

En esta fase vamos a escanear la web por si encontramos vulnerabilidades, vamos a empezar con un escaneo de directorios por si encontramos algún directorio abierto. Vamos a usar `gobuster` como herramienta para ello, con el siguiente comando:

```bash
gobuster dir -w <wordlist> -u <url> -b 403,404
```
La salida de este comando es la siguiente:

![gobusterScan](https://github.com/user-attachments/assets/81f1e141-58a2-4a18-9dd3-fb101fd6eba2)

Las direcciones `/info.php` y `/index.html` no contienen nada relevante, sin embargo `/javascript` tiene código de estado 301 (moved permanently) lo cual nos redirige a una página sin permisos. Además la dirección `/note` nos muestra una nota con la siguiente frase:

- Recuerda contratar certificado del dominio twitx.nyx para el lanzamiento.

Bien, con esto tenemos una ip y un dominio, por lo que podemos pensar que se está dando un port-forwarding a un servicio que no podemos "ver" con el escaneo anterior, por lo que vamos a añadir la siguiente línea al archivo `/etc/hosts`:

- `<IP-victima>    twitx.nyx http://twitx.nyx/`

Con esto ya podemos acceder desde el navegador a la página que estaba alojada en el servidor apache que habiamos enumerado anteriormente.

![paginaWebTwitx](https://github.com/user-attachments/assets/7a36ac28-8cb2-47cc-9547-6b618774ecdf)

Si recorremos la página podemos observar que en el último apartado hay un formulario de registro donde nos podemos registrar pero no hay ningún boton de login después de registrarnos. Aunque si miramos el código fuente si existe un elemento en la barra de navegación que es un botón de login, solo que estaba oculto.

![menuLogin](https://github.com/user-attachments/assets/bf1d251c-e3ae-45f8-9d2a-7970f73b6c14)

Vamos a realizar otro escaneo de directorios con este nuevo dominio por si encontramos algo que nos sirva.

![gobusterSecondScan](https://github.com/user-attachments/assets/910ec990-8eb6-4144-bd07-a6db611fc93d)

Si accedemos nos muestra un panel de login donde podemos acceder con el usuario previamente creado. Bien, si accedemos a la seccion de nuestro perfil nos fijaremos que la url tiene algunos parámetros.

![urlParams](https://github.com/user-attachments/assets/8fd5889f-1443-4fcb-8dba-48212ab2f9af)

Si probamos a cambiar estos parámetros con los que hemos obtenido por las dirección obtenidas en la segunda captura de gobuster vemos que no muestra nada pero tampoco falla la petición por lo que en realidad si hace una petición solo que no encuentra el recurso. En este punto yo probaría un tipo de ataque (Unrestricted file upload) dónde intento subir un archivo malicioso como foto de perfil y a continuación intento visualizar ese archivo haciendo esa llamada a través de los parámetros de la url.

Una primera prueba podría ser con el siguiente archivo:

![archivoMalicioso1](https://github.com/user-attachments/assets/17ddd481-6bf5-4217-97d0-f9fea4f880cd)

Lo intentamos subir como foto de perfil...

![loginMalicioso](https://github.com/user-attachments/assets/55b1cf8b-afc9-48fa-80c7-d90a95263cb0)

![loginMaliciosoExitoso1](https://github.com/user-attachments/assets/dc621da9-b3a2-4503-9a94-9823962a6fd5)

Y funciona!!! Bien, ahora vamos a inciar sesión de la forma explicada anteriormente y vamos a probar en la url.

![pruebaUrl1](https://github.com/user-attachments/assets/3e07f53b-32cd-442a-b5b0-b031f95cd53e)

Vemos que no muestra nada, además estuve probando en el resto de directorios pero sin éxito. Entonces decidí mirar en el código html como se estaba mostrando la foto, ya que no se veia en la ventana de perfil de usuario, y descubrí que el nombre de la foto estaba encodeada por eso no encontrabamos la dirección exacta.

![fotoEncodeada](https://github.com/user-attachments/assets/7d74c6a0-eb26-4d94-87c9-f6ab8ac45f7c)

Si probamos con la dirección: `http://twitx.nyx/private.php?folder=upload&file=<foto-encodeada-html>`

![fotoPayloadExito](https://github.com/user-attachments/assets/8074b700-9f28-45f1-8a0b-cf71261ea159)

La foto se ha renderizado como código html!!! Eso significa que podemos ejecutar comandos y probablemente obtener una reverse-shell. Voy a realizar el mismo procedimiento pero esta vez el archivo será una reverse shell muy conocida, el repositorio es el sigueinte:

- https://github.com/pentestmonkey/php-reverse-shell
- También podeis usar recursos online como [RevShells](https://www.revshells.com/)

Realizando los mismos pasos obtenemos una reverse-shell como el usuario www-data.

![revershell1](https://github.com/user-attachments/assets/d876ca86-61e3-4292-95da-affeb0a85009)

Ahora pasaremos a la fase de escalada de privilegios.

## Escalada de privilegios

Perfecto, estando dentro de la máquina víctima vamos a proceder a buscar alguna forma de escalar privilegios ya sea consiguiendo ejecutar comandos como superusuario o incluso consiguiendo acceso a la máquina como el propio superusuario. Para empezar vamos a inspeccionar las carpetas del servicio web por si encontramos algo de información útil como credenciales.

En efecto, dentro de la ruta `/var/www/twitx.nyx/includes` hay varios archivos del backend de la aplicación, dentro del archivo `config.php` podemos encontrar credenciales de la base de datos de la máquina.

![dbcredentials](https://github.com/user-attachments/assets/97e4093a-f5d9-4d74-ab75-3b413389dd78)

Si utilizamos estas credenciales para acceder a la base de datos encontraremos una tabla con los usuarios, y sus contraseñas en forma hash, creados en la aplicación web, uno de ellos tiene el rol de administrador.

![dbaccess](https://github.com/user-attachments/assets/55c7cfc6-af0d-4fb2-b062-b1031c3ae67e)

![adminHash](https://github.com/user-attachments/assets/3d89bf3f-d283-4f1b-b97c-33591dfdcaf4)

Como tenemos un hash vamos a usar la herramienta [john](https://www.openwall.com/john/) que ya viene instalada por defecto en kali.

![john](https://github.com/user-attachments/assets/e8bd9871-b64a-4c86-9fde-9f6a004bc838)

### Escalada al usuario timer

Por ahora esta contraseña va a estar asociada al usuario lenam pero no la vamos a poder usar aún. Si seguimos revisando el directorio anteriormente mencionado encontramos otro archivo llamado `taak.php` el cual ejecuta cierto comando como si fuera el usuario timer (otro usuario del sistema, podemos ver los usuarios del sistema si ejecutamos `cat /etc/passwd`). Como podemos ejecutar comandos como el usuario timer a través de este archivo vamos a insertar otra reverse shell (esta vez usando la herramienta online [RevShells](https://www.revshells.com/) que mencionamos antes. Precisamente la reverse shell de PHP de Ivan Sincek.

![timeruser](https://github.com/user-attachments/assets/12dcb34b-bb75-420d-a58f-19ee81dffc73)

Al añadirla al archivo `taak.php` y esperar a que se ejecute obtenemos otra reverse shell como el usuario timer. Si ejecutamos el comando `sudo -l` descubrimos que el usuario timer puede ejecutar el comando `ascii85` con sudo. Para saber como explotar esta vulnerabilidad podemos usar la página de [GTFOBins](https://gtfobins.github.io/), una página famosa con ejemplos de como explotar diferentes vulnerabilidades en binarios Unix.

- [ascii85](https://gtfobins.github.io/gtfobins/ascii85/)

### Escalada al usuario lenam

Segun GTFOBins con `ascii85` podemos leer archivos del sistema aunque no tengamos permisos para ello. Por lo tanto, como sabemos que el puerto 22 (ssh) esta abierto podemos intentar leer la clave privada del usuario lenam a través del comando `ascii85` de la siguiente forma:

```bash
sudo /usr/bin/ascii85 "/home/lenam/.ssh/id_rsa" | ascii85 --decode
```

![sshprivatekey](https://github.com/user-attachments/assets/2d953f94-22e8-41b9-aabb-297317e3a348)

Nos copiamos la clave en un archivo dentro de nuestra máquina atacante e intentamos acceder por ssh con las credenciales `lenam:patricia`.

![lenamssh](https://github.com/user-attachments/assets/e2ebb848-f594-4a07-9b06-e0d15073e438)

### Escalada a root

Genial, ya hemos conseguido acceder como el usuario lenam. Volvemos a realizar el mismo procedimiento y revisar los archivos a los que tiene acceso el usuario lenam por si podemos escalar al usuario root. Sin irnos muy lejos encontramos en el propio directorio del usuario una ruta `/look/inside`, dentro hay un binario que tiene el [bit SUID](https://www.redhat.com/en/blog/suid-sgid-sticky-bit) activo.

![unshare](https://github.com/user-attachments/assets/1d96e12e-a223-4ac0-95ff-64d66ebe529f)

Volvemos a la página [GTFOBins](https://gtfobins.github.io/) y nos muestra que el binario `unshare` se utiliza para salir de entornos restringidos generando una intérprete de comandos/terminal y podemos explotarlo obteniendo una shell interactiva con privilegios de root de la siguiente forma:

- [Unshare](https://gtfobins.github.io/gtfobins/unshare/)

```bash
./unshare -r /bin/sh
```
![root](https://github.com/user-attachments/assets/fb677aae-3e9d-4e33-8a02-e0cd22c774fa)

Hemos conseguido obtener acceso a la máquina como usuario root explotando algunas vulnerabilidades tanto web como de binarios. Y con esto quedaría resuelta la máquina Twitx de la plataforma Vulnyx. Un saludo y que el hacking este con vosotros!

