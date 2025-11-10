# 🪟 Habilitar RDP en Windows Server 2022 y Acceder desde tu Mac (via LXD)

## 🎯 Objetivo
Permitir el acceso remoto por **RDP (Remote Desktop Protocol)** a tu máquina virtual **Windows Server 2022** ejecutándose dentro de **LXD**, y hacerlo accesible desde tu **Mac**, incluso si está aislada por NAT dentro del bridge `lxdbr0`.

---

## 🧩 1. Activar el acceso RDP en Windows Server

Abre **PowerShell como Administrador** dentro de la VM Windows y ejecuta:

```powershell
# Habilitar acceso RDP
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" -Value 0

# Permitir tráfico RDP en el firewall
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Iniciar el servicio de RDP si no lo está
Start-Service TermService
```

Verifica que esté activo:
```powershell
Get-Service TermService
```

Debe mostrar:
```
Status   Name               DisplayName
------   ----               -----------
Running  TermService        Remote Desktop Services
```

---

## 🌐 2. Verificar IP interna

```powershell
ipconfig
```
Ejemplo de salida:
```
Ethernet adapter Ethernet0:
   IPv4 Address. . . . . . . . . . . : 10.191.69.218
```

Esta IP pertenece a la red interna `lxdbr0`.

---

## 🚫 Problema habitual: NAT de LXD

Tu red `lxdbr0` está **NATeada**, lo que significa:
- Windows **puede salir a Internet**.
- Pero tu **Mac no puede acceder directamente** a `10.191.69.218`, ya que pertenece a una red privada del host Ubuntu.

---

## ✅ 3. Soluciones posibles

### 🅰️ Opción 1 — Redirigir el puerto RDP con un *proxy device* (recomendada)

En el **host Ubuntu**, ejecuta:

```bash
lxc config device add win2022 rdp proxy   listen=tcp:0.0.0.0:3389   connect=tcp:10.191.69.218:3389
```

Esto crea un túnel entre el puerto **3389 del host** y el **3389 interno de Windows**.

> Ahora desde tu **Mac** (en la misma red que Ubuntu) puedes conectar con:
> ```
> mstsc /v:IP_DEL_HOST_UBUNTU
> ```
> o en **Microsoft Remote Desktop**:
> ```
> IP_DEL_HOST_UBUNTU:3389
> ```

💡 Ejemplo: Si Ubuntu tiene IP `10.0.0.4`, conéctate a `10.0.0.4:3389`.

---

### 🅱️ Opción 2 — Crear una red bridge compartida (sin NAT)

Si prefieres que Windows tenga una IP directa en tu red LAN:

```bash
lxc network create br0 parent=eth0 bridge.mode=bridge
lxc profile device set default eth0 network br0
lxc config device add win2022 eth1 nic network=br0 name=eth1
```

Tu VM Windows obtendrá una IP de tu red física (por ejemplo `10.0.0.x`) y podrás conectarte desde cualquier equipo de tu LAN.

---

### 🅲 Opción 3 — Usar túnel SSH

Mantiene el aislamiento y solo abre acceso temporal desde tu Mac:

```bash
ssh -L 3389:10.191.69.218:3389 azureuser@10.0.0.4
```

Después, abre **Microsoft Remote Desktop** y conecta a:
```
localhost:3389
```

---

## 🔁 4. Persistencia (opcional)

Haz que el proxy se active siempre con la VM:

```bash
lxc config set win2022 boot.autostart true
```

---

## ✅ Resultado esperado

- Windows Server 2022 accesible vía RDP desde tu Mac.  
- No se requiere dominio ni configuración extra de roles.  
- El tráfico RDP se enruta correctamente desde fuera del host Ubuntu hacia la VM.

---

## 🧾 Resumen rápido

| Acción | Comando |
|--------|----------|
| Activar RDP | `Set-ItemProperty ...` + `Enable-NetFirewallRule` |
| Crear túnel RDP en LXD | `lxc config device add win2022 rdp proxy ...` |
| Conexión desde Mac | `Microsoft Remote Desktop → IP_UBUNTU:3389` |
| Alternativas | SSH Tunnel o Bridge (br0) |

---

## 📘 Notas finales

- Puedes verificar puertos activos con:
  ```bash
  sudo ss -tlnp | grep 3389
  ```
- En entornos cloud (como Azure o VMware), asegúrate de abrir el puerto 3389 en el *Security Group* o *Firewall externo* si procede.
