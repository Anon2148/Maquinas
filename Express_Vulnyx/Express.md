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

Las dos primeras funciones solamente nos muestran las canciones y los géneros de canciones, pero las dos últimas si parecen tener relevancia. En primer lugar, tenemos una función que nos devolvería los usuarios que tuvieran una `key` especifica como parámetro. Y en segundo lugar tenemos una función que nos devuelve el status de una página web a través de una petición POST a lo que parece una dirección URL de admin `/api/admin/availability`. Como tenemos que empezar por algún sitio, vamos a empezar a enviar diferentes peticiones al primer endpoint para ver si obtenemos información.

