# 🚨 Ataque DTP — VLAN Hopping
### Laboratorio de Seguridad en Redes | ITLA
**Autor:** maitecruz23  
**Repositorio:** DTP-VLAN-Hopping-  
**Fecha:** Febrero 2026

---

VIDEO DE YOUTUBE

https://youtu.be/YhGLsdYvCZ0








## 📋 Tabla de Contenido

1. [Objetivo del Script](#1-objetivo-del-script)
2. [Topología de Red](#2-topología-de-red)
3. [Direccionamiento IP e Interfaces](#3-direccionamiento-ip-e-interfaces)
4. [VLANs Configuradas](#4-vlans-configuradas)
5. [Configuraciones de los Dispositivos](#5-configuraciones-de-los-dispositivos)
6. [Parámetros Usados](#6-parámetros-usados)
7. [Requisitos para Utilizar la Herramienta](#7-requisitos-para-utilizar-la-herramienta)
8. [Capturas de Pantalla](#8-capturas-de-pantalla)
9. [Medidas de Mitigación](#9-medidas-de-mitigación)

---

## 1. 🎯 Objetivo del Script

El script `ataque.py` tiene como objetivo demostrar un ataque de tipo **DTP VLAN Hopping**, una técnica de ataque a nivel de capa 2 del modelo OSI. En este ataque, un host atacante explota el protocolo **DTP (Dynamic Trunking Protocol)** de Cisco para negociar un enlace troncal con un switch vecino, logrando así convertir su puerto de acceso en un puerto trunk. Una vez establecido el trunk, el atacante obtiene visibilidad y acceso al tráfico de **todas las VLANs** permitidas en ese enlace, saltando las restricciones de segmentación por VLAN.

### ¿Qué hace el script paso a paso?

**Paso 1 — Sniffing con Scapy:**  
Scapy analiza la interfaz `eth0` del host atacante y espera capturar un paquete VTP/DTP enviado por el switch vecino. Utiliza un filtro basado en la dirección MAC multicast de Cisco `01:00:0c:cc:cc:cc`, que es la dirección de destino usada por los protocolos propietarios de Cisco como DTP, VTP y CDP.

**Paso 2 — Verificación de la condición de ataque:**  
Si Scapy intercepta el paquete, confirma que el puerto del switch está en modo `dynamic desirable` o `dynamic auto`, condición necesaria para que el ataque DTP sea posible. Si no detecta paquetes en el tiempo de espera (`timeout=10`), igualmente procede a lanzar Yersinia.

**Paso 3 — Lanzamiento de Yersinia:**  
Se lanza **Yersinia** en modo interactivo (`yersinia -I`), herramienta especializada en ataques a protocolos de capa 2. Yersinia envía paquetes DTP crafteados que negocian activamente un enlace troncal con el switch, logrando que el puerto cambie de modo `access` a modo `trunk/desirable`.

**Resultado del ataque:**  
El puerto del switch que antes solo pertenecía a la VLAN 10 (WINDOWS) aparece ahora como trunk activo (modo `desirable`, encapsulación `802.1q`, estado `trunking`), permitiendo al atacante ver y potencialmente inyectar tráfico en todas las VLANs de la red.

### Herramientas utilizadas

| Herramienta | Rol | Descripción |
|-------------|-----|-------------|
| **Scapy** | Framework principal | Sniffing e inspección de paquetes DTP/VTP en capa 2 |
| **Yersinia** | Herramienta auxiliar | Ejecución del ataque DTP, envío de paquetes maliciosos |
| **Python 3** | Lenguaje del script | Orquestación y automatización del ataque |

---

## 2. 🖧 Topología de Red

<img width="1176" height="1032" alt="image" src="https://github.com/user-attachments/assets/34defa03-f48e-4fb0-aed3-8943e4ec9aae" />


```

> ⚠️ **Escenario del ataque:** El host Windows (atacante) se conecta al SW-IZQ en modo acceso dentro de la VLAN 10. Mediante el ataque DTP, logra que su puerto Gi0/1 del SW-IZQ pase a modo trunk, obteniendo acceso al tráfico de la VLAN 20 donde reside el host Linux (víctima).

---

## 3. 🌐 Direccionamiento IP e Interfaces

| Dispositivo | Interfaz | Dirección IP | Máscara | Gateway |
|-------------|----------|--------------|---------|---------|
| vIOS (Router) | Gi0/0 | 20.24.11.1 | /24 (255.255.255.0) | — |
| SW-CENTRAL | VLAN 1 | 20.24.11.2 | /24 | 20.24.11.1 |
| SW-IZQ | VLAN 1 | 20.24.11.3 | /24 | 20.24.11.1 |
| SW-DER | VLAN 1 | 20.24.11.4 | /24 | 20.24.11.1 |
| Win (atacante) | e0 | 20.24.1.10 | /24 | — |
| Linux (víctima) | e0 | 20.24.2.10 | /24 | — |

---

## 4. 🗂️ VLANs Configuradas

| VLAN ID | Nombre | Propósito | Switches que la definen |
|---------|--------|-----------|-------------------------|
| 1 | default | Gestión y enlace troncal entre dispositivos | SW-CENTRAL, SW-IZQ, SW-DER |
| 10 | WINDOWS | Red del host atacante (Windows) | SW-CENTRAL, SW-IZQ |
| 20 | LINUX | Red del host víctima (Linux) | SW-CENTRAL, SW-DER |
| 99 | MANAGEMENT | VLAN de administración | SW-CENTRAL, SW-IZQ, SW-DER |

---

## 5. ⚙️ Configuraciones de los Dispositivos

### 🔹 Router (vIOS)

```cisco
enable
configure terminal
hostname vIOS
enable secret Cisco123!
service password-encryption
interface GigabitEthernet0/0
 description HACIA_SWITCH_CENTRAL
 ip address 20.24.11.1 255.255.255.0
 no shutdown
exit
line console 0
 login local
 logging synchronous
exit
line vty 0 4
 login authentication default
 transport input telnet ssh
exit
end
write memory
```

### 🔹 Switch Central — SW-CENTRAL (L3)

```cisco
enable
configure terminal
hostname SW-CENTRAL
enable secret Cisco123!
service password-encryption
vlan 10
 name WINDOWS
exit
vlan 20
 name LINUX
exit
vlan 99
 name MANAGEMENT
exit
interface vlan 1
 ip address 20.24.11.2 255.255.255.0
 no shutdown
exit
ip default-gateway 20.24.11.1
interface GigabitEthernet0/0
 description HACIA_ROUTER
 switchport mode trunk
 switchport nonegotiate
exit
interface GigabitEthernet0/1
 description HACIA_SW-IZQ
 switchport mode trunk
 switchport nonegotiate
exit
interface GigabitEthernet0/2
 description HACIA_SW-DER
 switchport mode trunk
 switchport nonegotiate
exit
end
write memory
```

### 🔹 Switch Izquierdo — SW-IZQ

```cisco
enable
configure terminal
hostname SW-IZQ
enable secret Cisco123!
service password-encryption
vlan 10
 name WINDOWS
exit
vlan 99
 name MANAGEMENT
exit
interface vlan 1
 ip address 20.24.11.3 255.255.255.0
 no shutdown
exit
ip default-gateway 20.24.11.1
interface GigabitEthernet0/0
 description HACIA_SW-CENTRAL
 switchport mode trunk
 switchport nonegotiate
exit
interface GigabitEthernet0/1
 description HACIA_WINDOWS
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
exit
end
write memory
```

### 🔹 Switch Derecho — SW-DER

```cisco
enable
configure terminal
hostname SW-DER
enable secret Cisco123!
service password-encryption
vlan 20
 name LINUX
exit
vlan 99
 name MANAGEMENT
exit
interface vlan 1
 ip address 20.24.11.4 255.255.255.0
 no shutdown
exit
ip default-gateway 20.24.11.1
interface GigabitEthernet0/0
 description HACIA_SW-CENTRAL
 switchport mode trunk
 switchport nonegotiate
exit
interface GigabitEthernet0/1
 description HACIA_KALI
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
exit
end
write memory
```

---

## 6. 🔧 Parámetros Usados

### Parámetros del Script `ataque.py`

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `INTERFAZ` | `"eth0"` | Interfaz de red del host atacante desde donde se escucha y lanza el ataque |
| `filter` | `"ether dst 01:00:0c:cc:cc:cc"` | Filtro BPF de Scapy para capturar solo paquetes DTP/VTP de Cisco |
| `count` | `1` | Número de paquetes a capturar para verificar la presencia de DTP |
| `timeout` | `10` | Segundos de espera para interceptar el paquete VTP antes de continuar |
| `iface` | `INTERFAZ` | Interfaz pasada al sniffer de Scapy |
| `os.system` | `"yersinia -I"` | Comando de sistema para lanzar Yersinia en modo interactivo |

### Parámetros de Red Relevantes

| Parámetro | Valor |
|-----------|-------|
| MAC multicast DTP/VTP Cisco | `01:00:0c:cc:cc:cc` |
| Encapsulación trunk | `802.1q` |
| VLAN nativa | `1` |
| Protocolo explotado | `DTP (Dynamic Trunking Protocol)` |
| Modo de puerto vulnerable | `dynamic desirable / dynamic auto` |

---

## 7. 📦 Requisitos para Utilizar la Herramienta

### Sistema Operativo

- **Kali Linux** (recomendado) o cualquier distribución Linux con soporte para Scapy y Yersinia
- El atacante debe estar **conectado físicamente o virtualmente** a un switch Cisco con DTP habilitado en el puerto de acceso

### Instalación de Dependencias

```bash
# Actualizar repositorios
sudo apt update

# Instalar Python 3
sudo apt install python3 python3-pip -y

# Instalar Scapy
pip3 install scapy

# Instalar Yersinia
sudo apt install yersinia -y
```

### Ejecución del Script

El script requiere **privilegios de root** para realizar sniffing y enviar paquetes en capa 2:

```bash
sudo python3 ataque.py
```

### Condiciones de Red Necesarias

Para que el ataque sea exitoso se deben cumplir las siguientes condiciones:

1. El puerto del switch al que está conectado el atacante debe tener DTP **habilitado** (modo `dynamic desirable` o `dynamic auto`)
2. El switch vecino debe estar enviando paquetes DTP/VTP (comportamiento por defecto en Cisco IOS)
3. La interfaz `eth0` del atacante debe poder realizar sniffing en capa 2
4. El puerto de acceso del atacante **NO** debe tener `switchport nonegotiate` configurado (eso bloquearía el ataque)

### Archivos del Repositorio

```
DTP-VLAN-Hopping/
├── ataque.py                          # Script principal del ataque DTP
├── README.md                          # Esta documentación técnica
├── Configuracion SW-DER               # Config del Switch Derecho
├── Configuracion SW-IZQ               # Config del Switch Izquierdo
├── Configuracion Switch Central-L3    # Config del Switch Central L3
└── Configuracion basica del Router    # Config del Router vIOS
```

---

## 8. 📸 Capturas de Pantalla

### Antes del Ataque — Estado inicial del SW-DER



**Troncales activas antes del ataque — Solo aparece Gi0/0:**

```
<img width="893" height="438" alt="image" src="https://github.com/user-attachments/assets/1063ae75-b9fc-4457-a4e3-8c02b5170a17" />

```

> ✅ En este punto el puerto `Gi0/1` (conectado al atacante) está en modo **access** y NO aparece como trunk.

---

### Durante el Ataque — Ejecución del script en Kali

```bash
<img width="798" height="274" alt="image" src="https://github.com/user-attachments/assets/242fb6b5-1217-4954-a9a6-54963b239389" />

```

**Yersinia detecta los switches vecinos en modo DTP:**

```

<img width="661" height="504" alt="image" src="https://github.com/user-attachments/assets/64b9379c-0e1e-4ff3-b697-39e3ff43937d" />


---

### Después del Ataque — Puerto Gi0/1 ahora en modo TRUNK

```
<img width="897" height="308" alt="image" src="https://github.com/user-attachments/assets/37fa22c2-3148-43dc-920d-cb4e3eef87d1" />

```

> 🔴 El puerto `Gi0/1` que antes era de **acceso** ahora está en modo **trunk/desirable**, permitiendo al atacante ver el tráfico de todas las VLANs, incluyendo la VLAN 20 donde está el host Linux (víctima).

---

## 9. 🛡️ Medidas de Mitigación

Para proteger la red contra ataques DTP VLAN Hopping se deben aplicar las siguientes medidas en todos los switches:

### ✅ Mitigación 1 — Deshabilitar DTP en puertos de acceso

Todos los puertos de acceso (host-facing) deben configurarse en modo estático y con `switchport nonegotiate` para que el switch nunca negocie un trunk con ese puerto:

```cisco
interface GigabitEthernet0/1
 switchport mode access
 switchport nonegotiate
```

### ✅ Mitigación 2 — Configurar explícitamente los puertos troncales

Los puertos trunk deben configurarse de forma estática y también con `nonegotiate`:

```cisco
interface GigabitEthernet0/0
 switchport mode trunk
 switchport nonegotiate
```

> 🔒 `switchport nonegotiate` deshabilita el envío y recepción de paquetes DTP, eliminando completamente la posibilidad de negociación de trunk.

### ✅ Mitigación 3 — Cambiar la VLAN nativa

La VLAN nativa (por defecto VLAN 1) es un vector de ataque. Se debe cambiar a una VLAN dedicada que no tenga hosts:

```cisco
interface GigabitEthernet0/0
 switchport trunk native vlan 99
```

### ✅ Mitigación 4 — Deshabilitar puertos no utilizados

Todos los puertos de switch que no estén en uso deben apagarse y asignarse a una VLAN de cuarentena:

```cisco
interface range GigabitEthernet0/3 - 7
 switchport mode access
 switchport access vlan 999
 shutdown
```

### ✅ Mitigación 5 — Restringir las VLANs permitidas en trunks

Los enlaces trunk solo deben permitir las VLANs estrictamente necesarias:

```cisco
interface GigabitEthernet0/0
 switchport trunk allowed vlan 10,20,99
```

### ✅ Mitigación 6 — Implementar Port Security

Limitar el número de MACs por puerto para prevenir MAC flooding y ataques combinados:

```cisco
interface GigabitEthernet0/1
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security
```

### ✅ Mitigación 7 — Habilitar BPDU Guard en puertos de acceso

Evitar que dispositivos no autorizados envíen BPDUs de Spanning Tree:

```cisco
interface GigabitEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### Resumen de Mitigaciones

| # | Mitigación | Comando Clave | Protección |
|---|------------|---------------|------------|
| 1 | Deshabilitar DTP | `switchport nonegotiate` | Elimina negociación de trunk en puertos de acceso |
| 2 | Trunk estático | `switchport mode trunk` | Sin modo dinámico en uplinks |
| 3 | Cambiar VLAN nativa | `switchport trunk native vlan 99` | Evita VLAN hopping por VLAN nativa |
| 4 | Apagar puertos sin uso | `shutdown` | Reduce superficie de ataque |
| 5 | Restringir VLANs en trunk | `switchport trunk allowed vlan` | Limita propagación de VLANs |
| 6 | Port Security | `switchport port-security` | Limita dispositivos conectados por puerto |
| 7 | BPDU Guard | `spanning-tree bpduguard enable` | Protege el árbol de expansión (STP) |

---



10. 🔐 Configuración RADIUS (AAA)
# Configuración RADIUS - Windows Server NPS

## Servidor RADIUS
- Sistema Operativo: Windows Server 2019
- IP: 20.24.11.100
- Máscara: 255.255.255.0
- Gateway: 20.24.11.1

## Configuración NPS
- Cliente RADIUS: Router_Cisco
- IP del cliente: 20.24.11.1
- Shared Secret: cisco123
- Puerto autenticación: 1812
- Puerto contabilidad: 1813

## Usuario RADIUS creado
- Username: admin_radius
- Password: Admin123!
- Grupo: Administrators
- Dial-in: Allow Access

## Configuración Router (AAA)
radius server Servidor_Windows
 address ipv4 20.24.11.100 auth-port 1812 acct-port 1813
 key cisco123

aaa new-model
aaa authentication login default group radius local
aaa authorization exec default group radius local if-authenticated

## Verificación
test aaa group radius admin_radius Admin123! legacy
show aaa servers
show aaa sessions

aaa new-model
aaa authentication login default group radius local
aaa authorization exec default group radius local if-authenticated
Verificación
ciscotest aaa group radius admin_radius Admin123! legacy
show aaa servers
show aaa sessions

> ⚠️ **Aviso Legal:** Este laboratorio fue realizado en un entorno controlado y simulado con fines exclusivamente educativos en el marco del curso de Seguridad en Redes del ITLA. La ejecución de estos ataques en redes reales sin autorización expresa es ilegal y está penada por la ley.
