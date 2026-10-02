# Práctica 2 - Infraestructura 1: VPN IPsec Site-to-Site (FortiGate <---> FortiGate)

* **Estudiante:** Cristopher Navarro  
* **Matrícula:** 2025-0720  
* **Asignatura:** Seguridad de Redes  
* **Docente:** Jonathan Esteban Rondón Corniel  
* **Institución:** Instituto Tecnológico de Las Américas (ITLA)  
* **Repositorio GitHub:** https://github.com/CristopherNavarro/Tarea3_SeguridadRedes_P1_CristopherNavarro_20250720  
* **Fecha de Realización:** Octubre 2026  

---

## 1. Enlace al Video Demostrativo en YouTube
* **URL del Video:** https://youtu.be/PzuzZlKLeTs  
*(Video explicativo con cámara y micrófono activos, duración ≤ 10 minutos, mostrando en pantalla fecha/hora, topología GNS3, interfaces web de FortiOS y pruebas de consola en vivo).*

---

## 2. Propósito y Objetivos del Laboratorio

El objetivo primordial de esta práctica consistió en diseñar, implementar, auditar y certificar una arquitectura de comunicaciones seguras mediante un túnel **VPN IPsec Site-to-Site** entre dos firewalls de próxima generación FortiGate (FortiOS 7.0.9 KVM), interconectando dos sedes corporativas distantes (Sitio A y Sitio B) a través de un proveedor de servicios de Internet público simulado (ISP).

### Objetivos Específicos Alcanzados:
1. Diseñar e implementar el direccionamiento IP personalizado derivado estrictamente de mi número de matrícula institucional (`2025-0720`).
2. Configurar la segmentación mediante VLANs 802.1Q en el Sitio A y servicio dinámico de direccionamiento IP mediante DHCP implementado directamente en la subinterfaz de FortiGate-1.
3. Establecer las asociaciones de seguridad (SA) IKE Fase 1 e IPsec Fase 2 compatibles y optimizadas para el entorno virtualizado, garantizando confidencialidad e integridad del tráfico.
4. Aplicar políticas de firewall simétricas de inspección de estado para controlar estrictamente el tráfico admitido a través del túnel VPN.
5. Desplegar un servicio de centro de datos seguro (Servidor Web HTTPS con certificado SSL TLS en el puerto 443) y validar su consumo exitoso desde la estación de trabajo remota.
6. Ejecutar pruebas de enforzamiento de seguridad y aislamiento mediante caída controlada de políticas para comprobar que no existan fugas de tráfico hacia la red pública.

---

## 3. Topología de Red y Diagrama de Conectividad

La topología fue construida y cableada en GNS3 respaldado por VMware Workstation:

```text
[PC-Usuario] (10.25.7.10/25 - VLAN 10)
      | (eth0)
      | (e1 - Access VLAN 10)
[SW-SiteA]
      | (e0 - 802.1Q Trunk)
      | (port2.10)
[FortiGate-1] (Sitio A)
      | (port1: 203.25.7.2/29)
      |
      | (f0/0: 203.25.7.1/29)
    [ISP]
      | (f0/1: 203.25.7.9/29)
      |
      | (port1: 203.25.7.10/29)
[FortiGate-2] (Sitio B)
      | (port2: 10.25.7.129/28)
      | (eth0)
[Servidor-Web] (10.25.7.130/28 - HTTPS 443)
```

---

## 4. Esquema de Direccionamiento IP (Matrícula: 2025-0720)

Aplicando los parámetros exigidos en la guía académica con base en mi matrícula:

| Dispositivo | Interfaz Física / Lógica | Rol / Segmento | Dirección IP | Máscara de Red | Puerta de Enlace |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PC-Usuario** | `eth0` | LAN Usuarios (VLAN 10) | `10.25.7.10` (DHCP) | `255.255.255.128` (/25) | `10.25.7.1` |
| **SW-SiteA** | `e0` (Trunk) / `e1` (Acceso) | Conmutación Local | N/A | N/A | N/A |
| **FortiGate-1** | `port2.10` (VLAN 10) | Gateway LAN Sitio A | `10.25.7.1` | `255.255.255.128` (/25) | N/A |
| **FortiGate-1** | `port1` | WAN Sitio A | `203.25.7.2` | `255.255.255.248` (/29) | `203.25.7.1` |
| **FortiGate-1** | `port3` | Gestión Web GUI | `192.168.6.201` | `255.255.255.0` (/24) | N/A |
| **ISP** | `FastEthernet0/0` | Enlace WAN Sitio A | `203.25.7.1` | `255.255.255.248` (/29) | N/A |
| **ISP** | `FastEthernet0/1` | Enlace WAN Sitio B | `203.25.7.9` | `255.255.255.248` (/29) | N/A |
| **ISP** | `Loopback0` | Simulación DNS/Internet | `8.8.8.8` | `255.255.255.255` (/32) | N/A |
| **FortiGate-2** | `port1` | WAN Sitio B | `203.25.7.10` | `255.255.255.248` (/29) | `203.25.7.9` |
| **FortiGate-2** | `port2` | Gateway LAN Sitio B | `10.25.7.129` | `255.255.255.240` (/28) | N/A |
| **FortiGate-2** | `port3` | Gestión Web GUI | `192.168.6.202` | `255.255.255.0` (/24) | N/A |
| **Servidor-Web** | `eth0` | Servidor Web Seguro | `10.25.7.130` | `255.255.255.240` (/28) | `10.25.7.129` |

---

## 5. Procedimiento de Configuración Implementado

### 5.1. Configuración del Router ISP (Cisco 7200 IOS 15.2)
Configuré el enrutamiento base del proveedor público para interconectar los dos extremos WAN y proporcionar resolución simulada:
```cisco
hostname ISP
no ip domain-lookup
interface Loopback0
 ip address 8.8.8.8 255.255.255.255
 no shutdown
interface FastEthernet0/0
 description Enlace WAN hacia FortiGate-1 (Sitio A)
 ip address 203.25.7.1 255.255.255.248
 no shutdown
interface FastEthernet0/1
 description Enlace WAN hacia FortiGate-2 (Sitio B)
 ip address 203.25.7.9 255.255.255.248
 no shutdown
```

### 5.2. Configuración de FortiGate-1 (Sitio A)
1. **Interfaces y Subinterfaz VLAN 10 con Servidor DHCP:**
```fortios
config system interface
    edit "port1"
        set mode static
        set ip 203.25.7.2 255.255.255.248
        set allowaccess ping
    next
    edit "port2.10"
        set vdom "root"
        set ip 10.25.7.1 255.255.255.128
        set allowaccess ping
        set interface "port2"
        set vlanid 10
    next
    edit "port3"
        set mode static
        set ip 192.168.6.201 255.255.255.0
        set allowaccess ping https ssh http
    next
end

config system dhcp server
    edit 1
        set default-gateway 10.25.7.1
        set netmask 255.255.255.128
        set interface "port2.10"
        set vci-match disable
        config ip-range
            edit 1
                set start-ip 10.25.7.10
                set end-ip 10.25.7.50
            next
        end
        set dns-service default
    next
end
```

2. **Ruta Estática por Defecto hacia el ISP:**
```fortios
config router static
    edit 1
        set gateway 203.25.7.1
        set device "port1"
    next
end
```

3. **Túnel VPN IPsec Site-to-Site (Fase 1 y Fase 2):**
```fortios
config vpn ipsec phase1-interface
    edit "VPN-S2S-FGT"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dpd on-idle
        set dhgrp 14
        set remote-gw 203.25.7.10
        set psksecret Fortinet123!
    next
end

config vpn ipsec phase2-interface
    edit "VPN-S2S-FGT"
        set phase1name "VPN-S2S-FGT"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 10.25.7.0 255.255.255.128
        set dst-subnet 10.25.7.128 255.255.255.240
        set auto-negotiate enable
    next
end

config router static
    edit 2
        set dst 10.25.7.128 255.255.255.240
        set device "VPN-S2S-FGT"
    next
end
```

4. **Políticas de Seguridad en el Firewall:**
```fortios
config firewall address
    edit "LAN_SITE_A"
        set subnet 10.25.7.0 255.255.255.128
    next
    edit "LAN_SITE_B"
        set subnet 10.25.7.128 255.255.255.240
    next
end

config firewall policy
    edit 1
        set name "LAN_to_VPN"
        set srcintf "port2.10"
        set dstintf "VPN-S2S-FGT"
        set action accept
        set srcaddr "LAN_SITE_A"
        set dstaddr "LAN_SITE_B"
        set schedule "always"
        set service "ALL"
    next
    edit 2
        set name "VPN_to_LAN"
        set srcintf "VPN-S2S-FGT"
        set dstintf "port2.10"
        set action accept
        set srcaddr "LAN_SITE_B"
        set dstaddr "LAN_SITE_A"
        set schedule "always"
        set service "ALL"
    next
end
```

### 5.3. Configuración de FortiGate-2 (Sitio B)
De forma simétrica, configuré la WAN en `port1` (`203.25.7.10/29`), la LAN en `port2` (`10.25.7.129/28`), el túnel IPsec con puerta de enlace remota hacia `203.25.7.2`, ruta estática hacia `10.25.7.0/25` por el túnel y políticas de firewall permitiendo el tráfico corporativo.

---

## 6. Evidencias de Verificación y Auditoría en Vivo

### 6.1. Comprobación del Direccionamiento DHCP en PC-Usuario
```text
/ # ip addr show dev eth0
7: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN qlen 1000
    link/ether 02:42:1f:97:a9:00 brd ff:ff:ff:ff:ff:ff
    inet 10.25.7.10/25 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:1fff:fe97:a900/64 scope link 
       valid_lft forever preferred_lft forever

/ # ip route show
default via 10.25.7.1 dev eth0  metric 207 
10.25.7.0/25 dev eth0 scope link  src 10.25.7.10 
```

### 6.2. Conectividad Extremo a Extremo a través del Túnel IPsec (Ping)
```text
/ # ping -c 4 10.25.7.130
PING 10.25.7.130 (10.25.7.130): 56 data bytes
64 bytes from 10.25.7.130: seq=0 ttl=62 time=13.100 ms
64 bytes from 10.25.7.130: seq=1 ttl=62 time=18.398 ms
64 bytes from 10.25.7.130: seq=2 ttl=62 time=20.392 ms
64 bytes from 10.25.7.130: seq=3 ttl=62 time=18.597 ms

--- 10.25.7.130 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 13.100/17.621/20.392 ms
```

### 6.3. Traza de Saltos de Red (Traceroute)
```text
/ # traceroute -n 10.25.7.130
traceroute to 10.25.7.130 (10.25.7.130), 30 hops max, 46 byte packets
 1  10.25.7.1  2.967 ms  0.608 ms  0.334 ms
 2  203.25.7.10  14.916 ms  22.792 ms  21.704 ms
 3  10.25.7.130  21.713 ms  23.226 ms  20.659 ms
```
*Interpretación técnica:* El primer salto es la puerta de enlace local `10.25.7.1` (FortiGate-1); el segundo salto es la IP pública del extremo remoto `203.25.7.10` (FortiGate-2), demostrando que el tráfico no expone nodos internos al ISP y viaja encapsulado; el tercer salto es el servidor de destino `10.25.7.130`.

### 6.4. Consumo del Servicio Web Seguro (HTTPS 443)
```text
/ # curl -k -i https://10.25.7.130/
HTTP/1.1 200 OK
Server: SimpleHTTP/0.6 Python/3.14.8
Date: Thu, 01 Oct 2026 23:12:37 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1006
Connection: close

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Servidor Web Seguro - Practica 2</title>
</head>
<body style="font-family: Arial, sans-serif; background-color: #f4f6f9; color: #333; padding: 30px;">
    <div style="background: white; border: 1px solid #ddd; padding: 25px; border-radius: 8px; max-width: 700px; margin: 0 auto; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
        <h1 style="color: #0b5394;">Servidor Web Seguro (HTTPS)</h1>
        <hr style="border: 0; border-top: 1px solid #ccc;">
        <p><strong>Estudiante:</strong> Cristopher Navarro</p>
        <p><strong>Matricula:</strong> 2025-0720</p>
        <p><strong>Asignatura:</strong> Seguridad de Redes</p>
        <p><strong>Docente:</strong> Jonathan Esteban Rondon Corniel</p>
        <p><strong>Segmento Servidor:</strong> 10.25.7.128/28 (IP: 10.25.7.130)</p>
        <p style="color: #274e13; font-weight: bold;">Acceso exitoso a traves del tunel VPN IPsec Site-to-Site!</p>
    </div>
</body>
</html>
```

### 6.5. Prueba de Enforzamiento de Seguridad y Aislamiento (Drop Test)
Para certificar que el firewall no permite el tránsito de tráfico fuera del túnel:
1. **Deshabilitación de la política `LAN_to_VPN` en FortiGate-1:**
   ```text
   / # ping -c 3 -W 1 10.25.7.130
   PING 10.25.7.130 (10.25.7.130): 56 data bytes

   --- 10.25.7.130 ping statistics ---
   3 packets transmitted, 0 packets received, 100% packet loss
   ```
   *Resultado:* El tráfico cae al **100 % de pérdida**, impidiendo cualquier fuga hacia la WAN no cifrada.

2. **Reactivación de la política `LAN_to_VPN` en FortiGate-1:**
   ```text
   / # ping -c 3 10.25.7.130
   PING 10.25.7.130 (10.25.7.130): 56 data bytes
   64 bytes from 10.25.7.130: seq=0 ttl=62 time=21.410 ms
   64 bytes from 10.25.7.130: seq=1 ttl=62 time=13.675 ms
   64 bytes from 10.25.7.130: seq=2 ttl=62 time=18.583 ms

   --- 10.25.7.130 ping statistics ---
   3 packets transmitted, 3 packets received, 0% packet loss
   round-trip min/avg/max = 13.675/17.889/21.410 ms
   ```
   *Resultado:* La conectividad se restaura de forma instantánea al **0 % de pérdida**.

---

## 7. Conclusiones Técnicas

1. **Eficiencia de la Solución IPsec:** La implementación del túnel IPsec Site-to-Site en modo túnel garantizó una comunicación confidencial, íntegra y transparente entre ambas sucursales, cifrando todo el contenido de la capa de transporte y aplicación.
2. **Control Estricto por Políticas:** La integración nativa de políticas de firewall orientadas a interfaces e IPsec en FortiOS permite un control granular del tráfico admitido, asegurando que ante cualquier alteración de la política el tráfico se descarte inmediatamente sin degradar la postura de seguridad.
3. **Validación de Servicios Reales:** El consumo satisfactorio de un servicio web HTTPS demostró que la arquitectura soporta tráfico corporativo real con negociación TLS sobre el túnel sin problemas de fragmentación ni MTU.
