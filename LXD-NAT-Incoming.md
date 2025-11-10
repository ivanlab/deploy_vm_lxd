# 🔁 NAT Entrante para LXD (RDP y Web)

Este documento explica cómo exponer servicios de **máquinas virtuales** y **contenedores** de LXD al exterior mediante reglas de **DNAT** y **forwarding** con `nftables`.

---

## 🧭 Objetivo

| Servicio | IP interna | Puerto interno | Puerto externo |
|-----------|-------------|----------------|----------------|
| RDP (Win2022) | 10.191.80.17 | 3389 | 3389 |
| HTTP (NGINX) | 10.191.80.50 | 80 | 80 |
| HTTPS (NGINX) | 10.191.80.50 | 443 | 443 |

---

## ⚙️ 1️⃣ Crear las reglas DNAT con `nftables`

Ejecutar en el **host Ubuntu**:

```bash
sudo nft add table ip nat 2>/dev/null || true
sudo nft add chain ip nat PREROUTING '{ type nat hook prerouting priority -100; }' 2>/dev/null || true

# RDP hacia Windows
sudo nft add rule ip nat PREROUTING tcp dport 3389 counter dnat to 10.191.80.17:3389

# HTTP/HTTPS hacia NGINX
sudo nft add rule ip nat PREROUTING tcp dport 80  counter dnat to 10.191.80.50:80
sudo nft add rule ip nat PREROUTING tcp dport 443 counter dnat to 10.191.80.50:443
```

---

## ⚙️ 2️⃣ Permitir el Forward

Permite que los paquetes fluyan entre `eth0` (host) y `lxdbr0`:

```bash
sudo nft add table inet filter 2>/dev/null || true
sudo nft add chain inet filter forward '{ type filter hook forward priority 0; policy accept; }' 2>/dev/null || true

sudo nft add rule inet filter forward iifname "eth0" oifname "lxdbr0" ct state established,related counter accept
sudo nft add rule inet filter forward iifname "lxdbr0" oifname "eth0" ct state new,established,related counter accept
```

---

## ⚙️ 3️⃣ Verificar las Reglas

Comprobar el estado de las reglas DNAT y forward:

```bash
sudo nft list table ip nat
sudo nft list table inet filter | grep forward -A5
```

Deberías ver algo similar a:

```
chain PREROUTING {
    type nat hook prerouting priority dstnat; policy accept;
    tcp dport 3389 counter dnat to 10.191.80.17:3389
    tcp dport 80   counter dnat to 10.191.80.50:80
    tcp dport 443  counter dnat to 10.191.80.50:443
}
```

---

## 🌐 4️⃣ Probar desde el Exterior

Desde tu Mac o un cliente remoto:

```bash
# RDP (Microsoft Remote Desktop)
open "rdp://full%20address=s:<IP_PUBLICA_UBUNTU>:3389"

# HTTP/HTTPS
curl -I http://<IP_PUBLICA_UBUNTU>/
curl -I https://<IP_PUBLICA_UBUNTU>/
```

> ⚠️ Si la VM Ubuntu está en **Azure**, asegúrate de abrir los puertos 80, 443 y 3389 en los **NSG / Security Rules**.

---

## 💾 5️⃣ Hacer las reglas persistentes

Guarda las reglas para que sobrevivan a reinicios:

```bash
sudo sh -c 'nft list ruleset > /etc/nftables.conf'
sudo systemctl enable nftables
```

---

## ✅ Resultado Esperado

- El contenedor NGINX es accesible desde fuera en los puertos 80/443.  
- La VM Win2022 acepta conexiones RDP en el puerto 3389.  
- El NAT saliente y entrante está gestionado por el demonio de LXD más las reglas DNAT manuales.  
