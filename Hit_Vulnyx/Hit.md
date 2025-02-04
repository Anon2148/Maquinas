Write-up máquina Hit de plataforma [Vulnyx](https://vulnyx.com/#hit)

![VulNyx Offensive Security Playground](https://github.com/user-attachments/assets/884b7c2c-8883-484c-944b-67f2b12f220d)

## Antes de empezar...

Qué vamos a necesitar:

- Máquina atacante kali (virtualbox)
- Máquina victima Vulnyx (ova en virtualbox)
- Red NAT compartida entre ambas máquinas (aislamiento, configuración de virtualbox)

No estoy usando ninguna configuracion especial ni nada.
## Reconocimiento

Como en todo proceso de pentesting vamos a empezar con la fase de reconocimiento, donde intentaremos obtener la máxima información posible de la máquina para poder explotarla (vulnerarla). En este caso, la máquina víctima nos muestra ya su IP en la pantalla de inicio.

![HitBanner](https://github.com/user-attachments/assets/084f9e75-d89f-4e6c-9fd7-e08fde4246dd)

Si no fuera así ya sabemos que con `netdiscover` o `arp-scan` se puede obtener la ip de la siguiente forma:

```bash
sudo netdiscover -i <interfaz-red> -r <rango-IP>
```

Donde:
- La flag `-i` especificará la interfaz de red (en el caso que tengamos varias) por la que se realizará el escaneo
- La flag `-r`especificará el rango de IP sobre el que queremos hacer el escaneo, si esta flag se omite `netdiscover` usará una lista por defecto con los rangos de IPs más conocidos. En nuestro caso el rango debería ser `10.0.0.0/8` ya que estamos en la red NAT compartida con la máquina.

```bash
sudo arp-scan --localnet -g -D
```

Donde:
- `--localnet` se usará para que el escaneo se haga en la red local
- `-g` se usará para que no se muestren los duplicados
- `-D` se usará para que nos muestre el round-trip-time al enviar el paquete ARP

Con la IP ya podemos pasar a realizar un escaneo de puertos, ejecutando `nmap` con el siguiente comando:

```bash
nmap -p- -T 5 -vvv -sS -Pn 10.0.2.10 -oN targeted
```

Obtenemos lo siguiente:

```
...
Nmap scan report for 10.0.2.10
Host is up, received arp-response (0.00011s latency).
Scanned at 2025-02-04 04:58:16 EST for 1s
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:18:06:C1 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
...
```

Solo obtenemos el puerto 80 (servicio web) abierto. Vamos a realizar otro escaneo algo más exhaustivo sobre el puerto 80.

```bash
nmap -p80 -sCV 10.0.2.10 -oN targeted2
```

Y obtenemos lo siguiente:

```
...
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.22.1
| http-git: 
|   10.0.2.10:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: Commit #5 
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.22.1
MAC Address: 08:00:27:18:06:C1 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
...
```

El segundo escaneo nos reporta que existe un directorio `.git/` y que el servidor es un `nginx 1.22.1`. Si accedemos desde el navegador.

