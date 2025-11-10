# 🔄 Restaurar NAT y Conectividad Saliente en LXD

Este documento detalla cómo **restaurar la conectividad saliente (NAT)** en un entorno **Ubuntu + LXD** cuando los contenedores o VMs pierden acceso a internet debido a conflictos entre `iptables` y `nftables`.

---

## 🚨 Síntoma

- Los contenedores/VMs pueden hacer `ping 8.8.8.8`, pero **no pueden conectar vía HTTP/HTTPS** (`curl` falla).  
- En `tcpdump` se observan paquetes **SYN sin respuesta**.  
- El host Ubuntu sí tiene acceso a internet.  

---

## 🧹 1️⃣ Limpiar Reglas Existentes

Eliminar todas las reglas activas en `nftables` (sin afectar configuración persistente):

```bash
sudo nft flush ruleset
```

---

## ⚙️ 2️⃣ Reiniciar el Demonio de LXD

Esto fuerza a LXD a **regenerar su configuración de red y NAT interna**:

```bash
sudo systemctl restart snap.lxd.daemon
```

> 💡 Este paso recrea las cadenas `inet lxd` con NAT automático para todos los bridges gestionados (`lxdbr0`, `lxdbr1`, etc.).

---

## 🔍 3️⃣ Verificar la Red del Bridge

Comprueba que la red principal (`lxdbr0`) tenga **NAT habilitado**:

```bash
lxc network show lxdbr0
```

Debe mostrar:

```yaml
config:
  ipv4.address: 10.191.80.1/24
  ipv4.nat: "true"
  ipv6.address: none
```

Si `ipv4.nat` está en `false`, corrígelo:

```bash
lxc network set lxdbr0 ipv4.nat true
```

---

## 🧱 4️⃣ Revisar las Reglas NAT Activas

Verifica que LXD haya generado correctamente la tabla NAT:

```bash
sudo nft list table inet lxd
```

Debes ver algo como:

```
chain pstrt.lxdbr0 {
    type nat hook postrouting priority srcnat; policy accept;
    ip saddr 10.191.80.0/24 ip daddr != 10.191.80.0/24 oifname != "lxdbr0" masquerade
}
```

---

## 🌐 5️⃣ Validar Conectividad

Dentro de un contenedor o VM:

```bash
lxc exec nginx -- ping -c3 8.8.8.8
lxc exec nginx -- curl -I http://archive.ubuntu.com/ubuntu/
```

Desde el host:

```bash
curl -I http://archive.ubuntu.com/ubuntu/
```

Ambos deben funcionar correctamente.

---

## 🧩 6️⃣ (Opcional) Ajustar MTU para Entornos Cloud

En Azure, a veces el MTU por defecto (1500) puede causar fragmentación. Ajusta a 1400:

```bash
lxc network set lxdbr0 bridge.mtu 1400
sudo ip link set dev lxdbr0 mtu 1400
```

Verifica dentro del contenedor:

```bash
ip link show eth0
```

---

## ✅ Resultado Esperado

- Los contenedores y VMs pueden acceder a internet (HTTP/HTTPS).  
- Las reglas NAT internas son gestionadas automáticamente por LXD.  
- No hay duplicidad entre `iptables` y `nftables`.  

---

## 💾 (Opcional) Persistencia de Configuración

Guarda las reglas regeneradas por LXD para inspección o respaldo:

```bash
sudo sh -c 'nft list ruleset > /etc/nftables.conf'
sudo systemctl enable nftables
```

---

## 🧠 Nota Final

> **LXD controla su propio NAT**. Evita mezclar reglas manuales (`iptables` o `nftables`) a menos que necesites DNAT (port forwarding).  
> Si NAT vuelve a romperse, simplemente ejecuta:

```bash
sudo nft flush ruleset
sudo systemctl restart snap.lxd.daemon
```
