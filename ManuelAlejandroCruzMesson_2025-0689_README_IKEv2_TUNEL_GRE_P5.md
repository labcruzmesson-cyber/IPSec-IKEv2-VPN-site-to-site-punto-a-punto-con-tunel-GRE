# VPN Site-to-Site GRE sobre IPsec (IKEv2)

**Asignatura:** Seguridad de Redes
**Estudiante:** Manuel Cruz
**Docente:** Jonathan Rondón
**Fecha:** 2 de julio de 2026

> **Nota:** el direccionamiento no se realizó en base a la matrícula del estudiante; este requerimiento se recordó demasiado tarde y no fue posible rehacer las topologías y videos.

---

## Tabla de Contenidos

- [1. Resumen y Objetivos](#1-resumen-y-objetivos)
- [2. Topología de Red y Direccionamiento](#2-topología-de-red-y-direccionamiento)
- [3. Especificación de Políticas y Parámetros de Seguridad (IKEv2)](#3-especificación-de-políticas-y-parámetros-de-seguridad-ikev2)
- [4. Configuración de la Interfaz de Túnel GRE Protegido](#4-configuración-de-la-interfaz-de-túnel-gre-protegido)
- [5. Puntos Críticos Analizados](#5-puntos-críticos-analizados)
- [6. Protocolo de Verificación y Diagnóstico Técnico](#6-protocolo-de-verificación-y-diagnóstico-técnico)
- [7. Conclusiones](#7-conclusiones)

---

## 1. Resumen y Objetivos

Este documento describe una nueva variante del laboratorio de VPN Site-to-Site, en la cual se combina el encapsulamiento **GRE (Generic Routing Encapsulation)** con el modelo modular de **IKEv2** para el establecimiento de la Fase 1, protegiendo el túnel mediante un transform-set IPsec configurado en **modo transporte**. A diferencia de la variante basada en ISAKMP (IKEv1), aquí se recuperan las ventajas de la suite modular de IKEv2 (proposal, policy, keyring y profile independientes) aplicadas sobre un túnel GRE en lugar de una interfaz VTI nativa. Se mantiene intacto el direccionamiento IP de la infraestructura física del proyecto original.

### Objetivos del Proyecto

- **Convergencia GRE + IKEv2:** combinar el transporte flexible de GRE con la robustez y modularidad del protocolo IKEv2 para el establecimiento de la Fase 1 (proposal, policy, keyring y profile independientes).
- **Confidencialidad e Integridad Avanzada:** garantizar la protección estricta del flujo de datos entre la LAN de PEER A (`172.16.1.0/24`) y la LAN de PEER B (`172.16.2.0/24`) mediante algoritmos de cifrado simétrico robustos (AES-256 / SHA-256) aplicados en modo transporte sobre el payload GRE.
- **Enrutamiento Simplificado hacia Tunnel0:** establecer la interfaz `Tunnel0` utilizando como origen y destino las direcciones IP públicas de cada peer (`tunnel source` como dirección IP explícita), asegurando el enrutamiento estático hacia la LAN remota a través de dicha interfaz.

---

## 2. Topología de Red y Direccionamiento

La topología lógica mantiene el diseño de dos sedes principales interconectadas a través de un gateway WAN que emula un proveedor de servicios de Internet (R-ISP). La interfaz lógica `Tunnel0` en cada extremo actúa como cabecera de encapsulamiento GRE, protegida en su totalidad por el perfil IPsec en modo transporte, negociado mediante IKEv2.

| Dispositivo / Sede | Interfaz | Dirección IP | Máscara de Subred | Propósito / Rol |
|---|---|---|---|---|
| PEER A | Gi0/0 | 10.0.0.60 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER A | Gi0/1 | 172.16.1.1 | 255.255.255.0 | Gateway LAN A / NAT Inside |
| PEER A | Tunnel0 | 192.168.100.1 | 255.255.255.252 | Interfaz GRE protegida por IPsec (transporte) |
| PEER B | Gi0/0 | 10.0.0.70 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER B | Gi0/1 | 172.16.2.1 | 255.255.255.0 | Gateway LAN B / NAT Inside |
| PEER B | Tunnel0 | 192.168.100.2 | 255.255.255.252 | Interfaz GRE protegida por IPsec (transporte) |
| R-ISP (Gateway) | N/A | 10.0.0.1 | 255.255.255.0 | Puerta de enlace predeterminada WAN |

---

## 3. Especificación de Políticas y Parámetros de Seguridad (IKEv2)

Esta implementación retoma la suite modular de IKEv2 (igual que en el esquema VTI original) pero, a diferencia de aquel, protege un túnel GRE mediante un transform-set en modo transporte en lugar de operar la interfaz `Tunnel0` directamente en modo IPsec.

### Fase 1: IKEv2 Proposal, Policy, Keyring y Profile — PEER A

| Parámetro | Valor |
|---|---|
| Algoritmo de Cifrado | AES-CBC-256 (Advanced Encryption Standard con encadenamiento de bloques de cifrado de 256 bits) |
| Integridad y Hash | SHA-256 (Secure Hash Algorithm de 256 bits para procesos de autenticación e intercambio seguro) |
| Grupo Diffie-Hellman | Grupo 14 (Exponenciación modular de 2048 bits para la generación segura de claves efímeras) |
| Autenticación | Llaves precompartidas asimétricas declaradas de forma independiente para el sentido local y remoto (`pre-shared-key local` / `pre-shared-key remote`) dentro del keyring `KEY_IKEV2` |
| Pre-Shared Key | `cisco123` |

### Fase 2: IPsec Transform Set en Modo Transporte

| Parámetro | Valor |
|---|---|
| Nombre del Set | `TS_IPSEC` |
| Encapsulación Criptográfica | ESP con AES de 256 bits (esp-aes 256) |
| Autenticación/Hashing de Datos | ESP bajo código de autenticación de mensajes cifrados (esp-sha256-hmac) |
| Modo de Operación | Modo Transporte (mode transport). IPsec cifra únicamente el payload GRE ya encapsulado, sin agregar una nueva cabecera IP completa como ocurre en modo túnel |
| Perfil IPsec | `IPSEC_PROF`, el cual vincula el transform-set `TS_IPSEC` con el perfil IKEv2 `PROF_IKEV2` y se aplica sobre la interfaz `Tunnel0` mediante `tunnel protection` |

---

## 4. Configuración de la Interfaz de Túnel GRE Protegido

La interfaz `Tunnel0` se configura como túnel GRE estándar, utilizando como `tunnel source` la dirección IP física de la interfaz WAN (en lugar de la referencia a la interfaz GigabitEthernet0/0) y aplicando la protección IPsec en modo transporte negociada mediante el perfil IKEv2. El enrutamiento hacia la LAN remota se realiza mediante una ruta estática directa hacia `Tunnel0`.

- **PEER A** — Interfaz Tunnel0 y Enrutamiento
- **PEER B** — Interfaz Tunnel0 y Enrutamiento

---

## 5. Puntos Críticos Analizados

### A. Tunnel Source como Dirección IP vs. Interfaz Física

A diferencia de las variantes anteriores, donde `tunnel source` referenciaba directamente la interfaz GigabitEthernet0/0, en este esquema se declara la dirección IP explícita (`10.0.0.60` / `10.0.0.70`) como origen del túnel. Funcionalmente el comportamiento es equivalente, ya que el router resuelve dicha IP contra la interfaz física que la posee; sin embargo, esta forma es menos flexible ante cambios de direccionamiento, dado que requeriría editar manualmente el comando si la IP de la interfaz cambia.

### B. GRE Protegido en Modo Transporte bajo IKEv2

Se combina la flexibilidad de encapsulamiento de GRE (permitiendo el transporte de tráfico multicast y protocolos de enrutamiento dinámico) con la robustez y modularidad de IKEv2 para la Fase 1. IPsec, operando en modo transporte, cifra exclusivamente el payload GRE ya encapsulado, delegando el encabezado de enrutamiento IP externo al propio túnel GRE.

### C. Ausencia de Ajuste Explícito en la ACL de NAT

Este script no incluye una modificación explícita de la lista `ACL_NAT_INTERNET`. Se recomienda validar que dicha lista de exclusión de NAT (`deny ip LAN_local LAN_remota` / `permit ip LAN_local any`) permanezca vigente desde configuraciones previas del laboratorio, ya que sigue siendo indispensable para que el tráfico inter-LAN no sea traducido erróneamente hacia Internet antes de alcanzar la interfaz `Tunnel0`.

---

## 6. Protocolo de Verificación y Diagnóstico Técnico

### Paso 1: Generación de Tráfico Interesante para Levantamiento de Túnel

Por definición, los mecanismos IPsec levantan las asociaciones de seguridad bajo demanda en presencia del primer paquete de datos legítimo que atraviese la interfaz `Tunnel0`. Para activar la infraestructura, se emite una solicitud de eco ICMP desde un host terminal interno con destino a la LAN remota.

### Paso 2: Validación del Canal de Control en Fase 1 (IKEv2 SA)

Para examinar el estado del intercambio de llaves y la correcta concordancia criptográfica de los peers remotos, se emplea el comando de diagnóstico avanzado para IKEv2:

```
Router# show crypto ikev2 sa
```

El resultado en consola debe declarar de forma explícita el parámetro **READY** en la columna de estado. Esto certifica que la autenticación mutua mediante las claves precompartidas y los perfiles asignados ha culminado con éxito.

### Paso 3: Validación del Canal de Datos en Fase 2 (IPsec SA sobre GRE, Modo Transporte)

Una vez establecido el enlace de control, es indispensable auditar que los flujos de datos GRE sean procesados por los algoritmos de encriptación simétrica ESP en modo transporte:

```
Router# show crypto ipsec sa | include pkts
```

Los registros y contadores del sistema para `#pkts encaps` (paquetes cifrados salientes) y `#pkts decaps` (paquetes descifrados entrantes) deben mostrar valores enteros mayores a cero e incrementar en tiempo real conforme persista el envío de ráfagas de datos a través del túnel GRE.

---

## 7. Conclusiones

- **La suite modular de IKEv2** — el uso de perfiles modulares (proposal, policy, keyring y profile) facilita la escalabilidad ante múltiples peers, incluso cuando IKEv2 se utiliza para proteger un túnel GRE en lugar de una interfaz VTI nativa.
- **El modo transporte de IPsec** — reduce el overhead de doble cabecera IP frente al modo túnel, siendo especialmente adecuado cuando el encapsulamiento IP externo ya es provisto por GRE.
- **El ajuste (o verificación) de la lista de exclusión de NAT** — continúa siendo indispensable para preservar las direcciones IP originales del tráfico inter-LAN y evitar su traducción errónea hacia Internet antes de ser encapsulado y cifrado.
