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


  
