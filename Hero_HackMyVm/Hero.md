Write-up de la máquina Hero de la plataforma [HackMyVm](https://hackmyvm.eu/)

![Screenshot 2025-02-11 at 10-40-19 HackMyVM](https://github.com/user-attachments/assets/42bd604b-07f0-4fb7-a533-a412c6c15bb4)

## Antes de empezar...

Qué vamos a necesitar:

- Máquina atacante AthenaOS (virtualbox)
- Máquina victima HackMyVm (ova en virtualbox)
- Red NAT compartida entre ambas máquinas (aislamiento, configuración de virtualbox)

No estoy usando ninguna configuracion especial ni nada.
## Reconocimiento

Vamos a iniciar el proceso de reconocimiento, en este caso la pantalla de boot de la máquina no muestra la ip, así que vamos a intentar obtener su ip con `netdiscover` o `arp-scan`. En este caso voy a utilizar `netdiscover` con el siguiente comando:

```bash
sudo netdiscover -r 10.0.2.0/16
```

Y con esto obtenemos la Ip de la máquina dentro de nuestra red NAT. Bien, con la Ip podemos proceder a realizar un escaneo de puertos con nmap, vamos a usar el siguiente comando:

```bash
sudo nmap -p- -sS -T 5 -Pn -n -vvv 10.0.2.4 -oN targeted
```

Y el escaneo nos muestra lo siguiente:

![nmapScan](https://github.com/user-attachments/assets/4f79f3cb-bb59-4bab-acdb-03404e37dfc9)


