# Ataque MAC Flooding

**Autor:** Roger Rodriguez  
**Matrícula:** 20250757  
**Fecha:** Junio 2026
**Link:** https://youtu.be/p0JDGb3RVeY

---

## Objetivo del laboratorio

Demostrar cómo un atacante puede desbordar la tabla CAM (Content Addressable Memory)
de un switch enviando miles de tramas Ethernet con MACs de origen falsas y aleatorias,
forzando al switch a comportarse como un hub y transmitir el tráfico por todos sus
puertos, permitiendo al atacante capturar tráfico de toda la red.

---

## Objetivo del script

El script `mac_flooding.py` realiza las siguientes acciones:

1. Genera 2000 tramas Ethernet con MACs de origen y destino aleatorias
2. Las envía a alta velocidad por la interfaz `eth0`
3. El switch aprende cada MAC falsa y llena su tabla CAM
4. Cuando la tabla CAM se desborda el switch hace flooding de todo el tráfico
5. El atacante puede capturar tráfico de todos los dispositivos

---

## Parámetros usados

| Parámetro | Valor | Descripción |
|---|---|---|
| `INTERFAZ` | `eth0` | Interfaz de red del atacante |
| `PAQUETES` | `2000` | Cantidad de tramas con MACs falsas |
| `DELAY` | `0.001` | Tiempo entre paquetes (1ms) |
| MAC origen | Aleatoria | Nueva MAC por cada paquete |
| MAC destino | Aleatoria | Destino inexistente |

---

## Requisitos para utilizar la herramienta

### Software
- Kali Linux
- Python 3.x
- Librería Scapy

### Instalación de dependencias
```bash
sudo apt update && sudo apt install python3-scapy -y
```

### Permisos
```bash
sudo python3 mac_flooding.py
```

---

## Documentación del funcionamiento del script

### ¿Cómo funciona MAC Flooding?

Un switch mantiene una tabla CAM que mapea MACs a puertos físicos.
Esta tabla tiene un tamaño limitado (generalmente miles de entradas).
Cuando la tabla se llena el switch no puede aprender nuevas MACs y
comienza a enviar todo el tráfico desconocido por todos sus puertos,
comportándose como un hub. Esto permite al atacante capturar tráfico
que no le estaba destinado.

### Flujo del ataque

```
NORMAL (switch funciona correctamente):
Tabla CAM:
  Puerto 1 → MAC_PC1
  Puerto 2 → MAC_PC2
  Puerto 3 → MAC_GW
Tráfico PC1→PC2 solo sale por Puerto 2

CON ATAQUE (tabla CAM desbordada):
Tabla CAM:
  Puerto 1 → MAC_falsa_1
  Puerto 1 → MAC_falsa_2
  ...2000 entradas falsas...
  (MACs reales expulsadas por falta de espacio)
Tráfico PC1→PC2 sale por TODOS los puertos (flooding)
Atacante captura TODO el tráfico
```

### Diagrama del ataque

```
ATTACKER (Kali)         SW-ACCESS-1              PC1 / PC2
20.25.7.100                                    
      |                      |                      |
      |--trama MAC-1-------->| tabla CAM: +MAC-1    |
      |--trama MAC-2-------->| tabla CAM: +MAC-2    |
      |--trama MAC-3-------->| tabla CAM: +MAC-3    |
      |       ...            |       ...            |
      |--trama MAC-2000----->| tabla CAM llena      |
      |                      |                      |
      |                 TABLA CAM LLENA             |
      |                      |                      |
PC1   |--datos a PC2-------->|---flood a TODOS----->| todos ven el trafico
      |<---------------------| incluyendo Kali      |
```

### Pasos del script

1. **Generación MAC:** Crea MACs aleatorias para origen y destino
2. **Frame Ethernet:** Construye trama con IP y ICMP para darle contenido
3. **Envío masivo:** Manda 2000 tramas a 1ms de intervalo
4. **Reporte:** Muestra progreso cada 200 paquetes

---

## Documentación de la red

### Topología

```
                    Router-GW
                   20.25.7.1/24
                        |
                    SW-CORE
                   /         \
            SW-ACCESS-1    SW-ACCESS-2
            /       \            \
        ATTACKER    PC1          PC2
      20.25.7.100  20.25.7.10  20.25.7.20
```

### Interfaces y direccionamiento

| Dispositivo | Interfaz | IP | VLAN |
|---|---|---|---|
| Router-GW | e0/0 | 20.25.7.1/24 | 1 |
| ATTACKER | eth0 | 20.25.7.100/24 | 1 |
| PC1 | eth0 | 20.25.7.10/24 | 1 |
| PC2 | eth0 | 20.25.7.20/24 | 1 |

---

## Ejecución del ataque

```bash
# Clonar el repositorio
git clone https://github.com/tu-usuario/mac-flooding-attack

# Entrar al directorio
cd mac-flooding-attack

# Ejecutar el script
sudo python3 mac_flooding.py
```

### Resultado esperado
```
==================================================
 MAC Flooding Attack
 Autor    : Roger Rodriguez
 Matricula: 20250757
 Interfaz : eth0
 Paquetes : 2000
==================================================
[*] Iniciando ataque...
[+] Paquetes enviados: 200/2000
[+] Paquetes enviados: 400/2000
[+] Paquetes enviados: 600/2000
[+] Paquetes enviados: 800/2000
...
==================================================
[*] Ataque finalizado
[+] Enviados : 2000
[-] Errores  : 0
==================================================
```

### Verificar impacto en SW-ACCESS-1
```
SW-ACCESS-1# show mac address-table count
Mac Entries for Vlan 1:
Dynamic Address Count : 277
```

---

## Contra-medida

### Descripción
Configurar **Port Security** en los puertos de acceso para limitar
la cantidad máxima de MACs que puede aprender cada puerto. Si se supera
el límite el puerto aplica la acción configurada (restrict, shutdown, protect).

### Implementación en SW-ACCESS-1
```
enable
configure terminal
interface e0/1
 switchport port-security
 switchport port-security maximum 5
 switchport port-security violation restrict
 switchport port-security aging time 2
interface e0/2
 switchport port-security
 switchport port-security maximum 5
 switchport port-security violation restrict
 switchport port-security aging time 2
end
write memory
```

### Modos de violación

| Modo | Descripción |
|---|---|
| `restrict` | Descarta tramas y genera log |
| `shutdown` | Deshabilita el puerto completamente |
| `protect` | Descarta tramas sin generar log |

### Verificación
```
SW-ACCESS-1# show port-security
Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
-----------  -------------  -----------  -----------------  ---------------
      Et0/1              5            0                  0         Restrict
      Et0/2              5            0                  0         Restrict
```

### ¿Por qué funciona?
Port Security limita a 5 MACs por puerto. El atacante puede intentar
llenar la tabla pero el switch solo aprenderá 5 MACs desde ese puerto,
ignorando el resto y protegiendo la tabla CAM de desbordarse.

---

## Capturas de pantalla

| Archivo | Descripción |
|---|---|
|<img width="898" height="537" alt="image" src="https://github.com/user-attachments/assets/f70eb133-0c86-41ac-b3df-713d42496a61" /> | Topología en EVE-NG |
| <img width="784" height="764" alt="image" src="https://github.com/user-attachments/assets/e0e8ceea-ba44-4f3f-aad3-add043959f01" />| Script corriendo con 2000 paquetes |
| <img width="753" height="631" alt="image" src="https://github.com/user-attachments/assets/8b76c672-1749-4b74-a975-3be87e3ea1e3" />| Tabla CAM con 277 MACs falsas |
|<img width="879" height="207" alt="image" src="https://github.com/user-attachments/assets/afad0ae1-bc0b-484d-82cd-85a426c0a587" /> | Port Security activo en SW-ACCESS-1 |

---

## Referencias

- [MAC Flooding Attack](https://www.geeksforgeeks.org/mac-flooding-attack/)
- [Cisco Port Security](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/ios/12-2SX/configuration/guide/book/port_sec.html)
- [Scapy Documentation](https://scapy.readthedocs.io/)
