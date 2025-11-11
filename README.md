# 🧩 LXD Deployment Guide — VM + Containers + NAT

Este repositorio documenta un despliegue completo de **infraestructura virtualizada con LXD**, incluyendo una **VM Windows Server 2022**, contenedores Linux (Kafka, NGINX), y configuración de **red NAT saliente y entrante**.  
El entorno se ejecuta sobre **Ubuntu en Azure**, con adaptaciones específicas para redes cloud y escenarios de desarrollo.

---

## 🧠 Arquitectura

### 🧩 [architecture.png](./architecture2.png)
Diagrama del entorno de red:
- Host Ubuntu con interfaces `eth0` y `lxdbr0`.
- VM Windows + Contenedores Linux en bridge interno.
- NAT saliente y DNAT entrante controlados por `nftables`.

---


## 📖 Documentación Principal

### 🪟 [lxd-win2022-setup.md](./lxd-win2022-setup.md)
Guía detallada para crear y configurar una VM **Windows Server 2022** bajo LXD, incluyendo:
- Creación de VM vacía y asignación de recursos.
- Adjuntar ISOs de instalación y controladores VirtIO.
- Desactivar Secure Boot y arrancar la instalación.
- Ajustes de red y proxy RDP.

---

### 🐳 [lxd_kafka_deployment.md](./lxd_kafka_deployment.md)
Pasos para desplegar un contenedor **Kafka** en LXD con configuración de red interna.  
Incluye validación de conectividad, pruebas de rendimiento y consideraciones sobre almacenamiento.

---

### 🌐 [LXD-NAT-Restore.md](./LXD-NAT-Restore.md)
Procedimiento para **restaurar la conectividad NAT saliente** cuando los contenedores pierden acceso a Internet.  
Cubre:
- Limpieza del ruleset `nftables`.
- Reinicio del demonio LXD.
- Verificación de `ipv4.nat` y diagnóstico de conectividad.
- Ajuste opcional de MTU (Azure-friendly: 1400).

---

### 🔁 [LXD-NAT-Incoming.md](./LXD-NAT-Incoming.md)
Configura las reglas de **NAT entrante (DNAT)** para exponer servicios:
- RDP (3389) hacia la VM Windows Server 2022.
- HTTP/HTTPS (80/443) hacia el contenedor NGINX.  
Incluye comandos `nftables`, configuración persistente y validación desde clientes externos.

---

### 🔍 [check-lxd-webui.md](./check-lxd-webui.md)
Guía rápida para verificar el acceso y estado del panel **LXD Web UI**, útil para validación remota y administración visual.

---

### 🌍 [access_windows_from_mac.md](./access_windows_from_mac.md)
Pasos para conectarse a la VM **Windows 2022 desde macOS** mediante Microsoft Remote Desktop, con configuración de certificado opcional y resolución de problemas comunes.

---

## Acceso (No en Github...)


### 🌍 [acceso](./access.md)

---
## Export to VMWare


### 🌍 [Export](./export-from-azure.md)




---

## ⚙️ Notas finales

- Todo el despliegue está optimizado para entornos **Ubuntu + LXD (snap)**.  
- Las reglas de `nftables` y redes `lxdbr0` se regeneran automáticamente tras reinicios del demonio.  
- Los ajustes están validados para **Azure VMs** con red dinámica (`eth0`).

> ✨ Autor: [Ivan Padilla](https://github.com/ivanlab) — Cisco Outshift / AI & Cloud Infra
>
“Con cariño y unas cuantas reglas de nftables,
— ChatGPT (coautor técnico invisible del repo 🧠💻)”

---