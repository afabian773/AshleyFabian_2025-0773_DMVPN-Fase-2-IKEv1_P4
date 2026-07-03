# DMVPN Fase 2 con IKEv1 y OSPF

**Nombre:** Ashley Fabian Ortiz  
**Matrícula:** 2025-0773  
**Práctica:** P4  

---

## Objetivo

Configurar una VPN hub and spoke punto a multipunto **DMVPN (Dynamic Multipoint VPN) Fase 2** con IKEv1 y enrutamiento dinámico OSPF entre un Hub y dos Spokes (Cisco CSR1000v), permitiendo comunicación cifrada tanto entre Hub↔Spoke como directamente entre Spoke↔Spoke sin pasar por el Hub.

DMVPN combina tres tecnologías:
- **mGRE (Multipoint GRE):** Una sola interfaz Tunnel en el Hub acepta conexiones de múltiples Spokes simultáneamente.
- **NHRP (Next Hop Resolution Protocol):** Los Spokes se registran con el Hub como servidor NHS. En Fase 2, el Hub resuelve la IP NBMA (pública) del Spoke destino para que los Spokes puedan crear túneles directos entre sí.
- **IPSec:** Cifra todo el tráfico GRE que viaja por los túneles.

### Diferencia entre Fases DMVPN

| Característica | Fase 1 | Fase 2 | Fase 3 |
|----------------|--------|--------|--------|
| Tráfico Spoke↔Spoke | Pasa por Hub | **Directo** | Directo + summarization |
| NHRP redirect | No | Sí (Hub) | Sí (Hub) |
| NHRP shortcut | No | Sí (Spokes) | Sí (Spokes) |
| Routing en Hub | No split-horizon | No split-horizon | ip nhrp redirect |
| Escalabilidad | Baja | Media | Alta |

---

## Topología

```
                    [PC-Hub]
                       |
                    [SW-Hub]
                       |
[PC-S1]--[SW-S1]--[Spoke1]--[ISP]--[Hub]
                              |
                          [Spoke2]--[SW-S2]--[PC-S2]
                          
Túneles DMVPN (mGRE sobre IPSec):
Hub (10.0.0.1) ←→ Spoke1 (10.0.0.2)
Hub (10.0.0.1) ←→ Spoke2 (10.0.0.3)
Spoke1 (10.0.0.2) ←→ Spoke2 (10.0.0.3) [túnel directo Fase 2]
```

---

## Direccionamiento IP

| Dispositivo | Interfaz         | Dirección IP  | Máscara         | Descripción           |
|-------------|------------------|---------------|-----------------|-----------------------|
| Hub         | GigabitEthernet1 | 25.7.73.193   | 255.255.255.252 | WAN Hub → ISP         |
| ISP         | GigabitEthernet1 | 25.7.73.194   | 255.255.255.252 | WAN ISP → Hub         |
| ISP         | GigabitEthernet2 | 25.7.73.197   | 255.255.255.252 | WAN ISP → Spoke1      |
| ISP         | GigabitEthernet3 | 25.7.73.201   | 255.255.255.252 | WAN ISP → Spoke2      |
| Spoke1      | GigabitEthernet2 | 25.7.73.198   | 255.255.255.252 | WAN Spoke1 → ISP      |
| Spoke2      | GigabitEthernet2 | 25.7.73.202   | 255.255.255.252 | WAN Spoke2 → ISP      |
| Hub         | GigabitEthernet3 | 25.7.73.205   | 255.255.255.248 | Gateway LAN Hub       |
| Spoke1      | GigabitEthernet3 | 25.7.73.213   | 255.255.255.248 | Gateway LAN Spoke1    |
| Spoke2      | GigabitEthernet3 | 25.7.73.221   | 255.255.255.248 | Gateway LAN Spoke2    |
| Hub         | Tunnel0 (mGRE)   | 10.0.0.1      | 255.255.255.0   | NHS (Hub DMVPN)       |
| Spoke1      | Tunnel0 (mGRE)   | 10.0.0.2      | 255.255.255.0   | NHC Spoke1            |
| Spoke2      | Tunnel0 (mGRE)   | 10.0.0.3      | 255.255.255.0   | NHC Spoke2            |
| PC-Hub      | eth0             | 25.7.73.206   | 255.255.255.248 | Host LAN Hub          |
| PC-S1       | eth0             | 25.7.73.214   | 255.255.255.248 | Host LAN Spoke1       |
| PC-S2       | eth0             | 25.7.73.222   | 255.255.255.248 | Host LAN Spoke2       |

### Subredes utilizadas (bloque 25.7.73.192/27)

| Subred           | Rango utilizable            | Uso                  |
|-------------------|-----------------------------|----------------------|
| 25.7.73.192/30    | 25.7.73.193 – 25.7.73.194   | WAN Hub – ISP        |
| 25.7.73.196/30    | 25.7.73.197 – 25.7.73.198   | WAN ISP – Spoke1     |
| 25.7.73.200/30    | 25.7.73.201 – 25.7.73.202   | WAN ISP – Spoke2     |
| 25.7.73.204/29    | 25.7.73.205 – 25.7.73.210   | LAN Hub              |
| 25.7.73.208/29    | 25.7.73.209 – 25.7.73.214   | LAN Spoke1           |
| 25.7.73.216/29    | 25.7.73.217 – 25.7.73.222   | LAN Spoke2           |
| 10.0.0.0/24       | 10.0.0.1 – 10.0.0.254       | Red DMVPN (Tunnel0)  |

> La red del túnel DMVPN usa el espacio privado `10.0.0.0/24` para evitar solapamiento con el bloque `25.7.73.0/24` usado en las interfaces físicas.

---

## Parámetros de configuración

### IKEv1 ISAKMP Policy

| Parámetro      | Valor               |
|----------------|---------------------|
| Cifrado        | AES 256             |
| Hash           | SHA-256             |
| Autenticación  | Pre-shared key      |
| Grupo DH       | Group 14 (2048-bit) |
| Lifetime       | 86400 segundos      |
| Pre-shared key | Cisco123!           |
| Peer address   | 0.0.0.0 (cualquier) |

### IPSec Transform Set

| Parámetro      | Valor         |
|----------------|---------------|
| Cifrado ESP    | AES 256       |
| Integridad ESP | SHA-256 HMAC  |
| Modo           | Transport     |

### Tunnel mGRE

| Parámetro              | Hub          | Spoke1        | Spoke2        |
|------------------------|--------------|---------------|---------------|
| IP Tunnel              | 10.0.0.1/24  | 10.0.0.2/24   | 10.0.0.3/24   |
| Tunnel source          | Gi1          | Gi2           | Gi2           |
| Tunnel mode            | gre multipoint | gre multipoint | gre multipoint |
| NHRP network-id        | 100          | 100           | 100           |
| NHRP NHS               | —            | 10.0.0.1      | 10.0.0.1      |
| NHRP map al Hub        | —            | 10.0.0.1→25.7.73.193 | 10.0.0.1→25.7.73.193 |
| ip nhrp redirect       | Sí           | No            | No            |
| ip nhrp shortcut       | No           | Sí            | Sí            |
| ip ospf network        | broadcast    | broadcast     | broadcast     |
| ip ospf priority       | 2 (DR)       | 0 (never DR)  | 0 (never DR)  |

### OSPF

| Parámetro   | Hub                              | Spoke1                            | Spoke2                            |
|-------------|----------------------------------|-----------------------------------|-----------------------------------|
| Process ID  | 1                                | 1                                 | 1                                 |
| Area        | 0                                | 0                                 | 0                                 |
| Networks    | 10.0.0.0/24, 25.7.73.204/29     | 10.0.0.0/24, 25.7.73.208/29      | 10.0.0.0/24, 25.7.73.216/29      |

---

## Scripts de configuración

### ISP

```
enable
configure terminal
hostname ISP

interface GigabitEthernet1
 ip address 25.7.73.194 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2
 ip address 25.7.73.197 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.201 255.255.255.252
 no shutdown
 exit

end
write memory
```

### Hub

```
enable
configure terminal
hostname Hub

interface GigabitEthernet1
 ip address 25.7.73.193 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.205 255.255.255.248
 no shutdown
 exit

! --- Ruta default y rutas hacia WANs de Spokes ---
ip route 0.0.0.0 0.0.0.0 25.7.73.194
ip route 25.7.73.196 255.255.255.252 25.7.73.194
ip route 25.7.73.200 255.255.255.252 25.7.73.194

! --- IKEv1 ---
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
 exit

crypto isakmp key Cisco123! address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
 exit

crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
 exit

! --- Tunnel mGRE (Hub) ---
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 ip nhrp authentication Cisco123
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip nhrp redirect
 ip ospf network broadcast
 ip ospf priority 2
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
 exit

! --- OSPF ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 25.7.73.204 0.0.0.7 area 0
 exit

end
write memory
```

### Spoke1

```
enable
configure terminal
hostname Spoke1

interface GigabitEthernet2
 ip address 25.7.73.198 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.213 255.255.255.248
 no shutdown
 exit

! --- Ruta default y rutas explícitas hacia WAN de Spoke2 ---
ip route 0.0.0.0 0.0.0.0 25.7.73.197
ip route 25.7.73.200 255.255.255.252 25.7.73.197
ip route 25.7.73.202 255.255.255.255 25.7.73.197

! --- IKEv1 ---
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
 exit

crypto isakmp key Cisco123! address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
 exit

crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
 exit

! --- Tunnel mGRE (Spoke) ---
interface Tunnel0
 ip address 10.0.0.2 255.255.255.0
 ip nhrp authentication Cisco123
 ip nhrp map 10.0.0.1 25.7.73.193
 ip nhrp map multicast 25.7.73.193
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip nhrp nhs 10.0.0.1
 ip nhrp shortcut
 ip ospf network broadcast
 ip ospf priority 0
 tunnel source GigabitEthernet2
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
 exit

! --- OSPF ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 25.7.73.208 0.0.0.7 area 0
 exit

end
write memory
```

### Spoke2

```
enable
configure terminal
hostname Spoke2

interface GigabitEthernet2
 ip address 25.7.73.202 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.221 255.255.255.248
 no shutdown
 exit

! --- Ruta default y rutas explícitas hacia WAN de Spoke1 ---
ip route 0.0.0.0 0.0.0.0 25.7.73.201
ip route 25.7.73.196 255.255.255.252 25.7.73.201
ip route 25.7.73.198 255.255.255.255 25.7.73.201

! --- IKEv1 ---
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
 exit

crypto isakmp key Cisco123! address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
 exit

crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN
 exit

! --- Tunnel mGRE (Spoke) ---
interface Tunnel0
 ip address 10.0.0.3 255.255.255.0
 ip nhrp authentication Cisco123
 ip nhrp map 10.0.0.1 25.7.73.193
 ip nhrp map multicast 25.7.73.193
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip nhrp nhs 10.0.0.1
 ip nhrp shortcut
 ip ospf network broadcast
 ip ospf priority 0
 tunnel source GigabitEthernet2
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown
 exit

! --- OSPF ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 25.7.73.216 0.0.0.7 area 0
 exit

end
write memory
```

### SW-Hub, SW-S1, SW-S2 (IOSvL2)

```
enable
configure terminal

interface GigabitEthernet0/0
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

end
write memory
```

### PC-Hub, PC-S1, PC-S2 (VPCS)

```
! PC-Hub
ip 25.7.73.206 255.255.255.248 25.7.73.205
save

! PC-S1
ip 25.7.73.214 255.255.255.248 25.7.73.213
save

! PC-S2
ip 25.7.73.222 255.255.255.248 25.7.73.221
save
```

---

## Verificación y pruebas

> Comandos de verificación ejecutados en Hub, Spoke1 y Spoke2.

### Comandos de verificación

```
! En Hub
show dmvpn
show ip ospf neighbor
show crypto isakmp sa
show ip route ospf

! En Spoke1 y Spoke2
show dmvpn
show crypto isakmp sa
```

### Resultados obtenidos — Hub

**show dmvpn:**
```
Type:Hub, NHRP Peers:2
25.7.73.198   10.0.0.2   UP   Dynamic  ← Spoke1 registrado
25.7.73.202   10.0.0.3   UP   Dynamic  ← Spoke2 registrado
```

**show ip ospf neighbor:**
```
25.7.73.213   0   FULL/DROTHER   Tunnel0  ← Spoke1
25.7.73.221   0   FULL/DROTHER   Tunnel0  ← Spoke2
```

**show ip route ospf:**
```
O   25.7.73.208/29 [110/1001] via 10.0.0.2, Tunnel0  ← LAN Spoke1
O   25.7.73.216/29 [110/1001] via 10.0.0.3, Tunnel0  ← LAN Spoke2
```

### Resultados obtenidos — Spoke1

**show dmvpn:**
```
Type:Spoke, NHRP Peers:2
25.7.73.193   10.0.0.1   UP   Static   ← Hub (entrada estática)
25.7.73.202   10.0.0.3   UP   Dynamic  ← Spoke2 (túnel directo Fase 2)
```

**show crypto isakmp sa:**
```
25.7.73.198   25.7.73.202   QM_IDLE   ACTIVE  ← SA directa Spoke1↔Spoke2
25.7.73.193   25.7.73.198   QM_IDLE   ACTIVE  ← SA Spoke1↔Hub
```

### Pruebas de conectividad

**Hub → Spokes:**
```
Hub# ping 25.7.73.213
Success rate is 100 percent (5/5)

Hub# ping 25.7.73.221
Success rate is 100 percent (5/5)
```

**Spoke1 → Spoke2 (túnel directo):**
```
Spoke1# ping 25.7.73.221 source GigabitEthernet3
! Primer intento: 0% (negociación NHRP + IPSec)
! Segundo intento: 100% (túnel directo establecido)
Success rate is 100 percent (5/5)
```

**Spoke2 → Spoke1 (túnel directo):**
```
Spoke2# ping 25.7.73.213 source GigabitEthernet3
Success rate is 100 percent (5/5)
```

> El primer ping entre Spokes falla mientras NHRP resuelve la IP NBMA del Spoke destino y se negocia la SA IPSec directa. Este es el comportamiento esperado y definitorio de DMVPN Fase 2.

---

## Capturas de pantalla

> Insertar aquí las capturas de:
> 1. Topología completa en GNS3 con nombre y matrícula visible
![](./Topologia.png)

> 2. `show dmvpn` en Hub (2 peers UP Dynamic)
![](./CH1.png)

> 3. `show ip ospf neighbor` en Hub (2 vecinos FULL)
![](./CH2.png)

> 4. `show crypto isakmp sa` en Hub
![](./CH3.png)

> 5. `show ip route ospf` en Hub (rutas hacia LANs de Spokes)
![](./CH4.png)

> 6. `show dmvpn` en Spoke1 (peer dinámico hacia Spoke2)
![](./C1S1.png)

> 7. `show crypto isakmp sa` en Spoke1 (SA directa con Spoke2)
![](./C1S2.png)

> 8. Ping Spoke1 → Spoke2 (primer fallo + segundo exitoso)
![](./C1S3.png)

> 9. Ping Spoke2 → Spoke1 exitoso
![](./C2S3.png)
---

## Conclusión

Se configuró exitosamente DMVPN Fase 2 con IKEv1 y OSPF entre un Hub y dos Spokes. Los Spokes se registraron dinámicamente con el Hub mediante NHRP, y OSPF distribuyó las rutas de las LANs a través del túnel mGRE. La característica definitoria de la Fase 2 — el túnel directo Spoke↔Spoke sin pasar por el Hub — fue verificada mediante el comportamiento del primer ping fallido (durante la resolución NHRP y negociación IPSec) seguido de pings exitosos al establecerse la SA directa entre Spoke1 (25.7.73.198) y Spoke2 (25.7.73.202). Las dos SAs IKEv1 en cada Spoke (una hacia el Hub y otra hacia el Spoke remoto) confirmaron el funcionamiento correcto de la Fase 2.
