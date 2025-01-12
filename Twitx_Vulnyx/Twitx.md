Write-up máquina Twitx de plataforma [Vulnyx](https://vulnyx.com/#twitx)

## Antes de empezar...

Qué vamos a necesitar:

- Máquina atacante kali (virtualbox)
- Máquina victima Vulnyx (ova en virtualbox)
- Red NAT compartida entre ambas máquinas (aislamiento, configuración de virtualbox)

No estoy usando ninguna configuracion especial ni nada.
## Reconocimiento

Como en todo proceso de pentesting vamos a empezar con la fase de reconocimiento, donde intentaremos obtener la máxima información posible de la máquina para poder explotarla (vulnerarla). En este caso, la máquina víctima nos muestra ya su IP en la pantalla de inicio