# 🧩 Comprobación y habilitación de la Web UI de LXD en Ubuntu 24.04

Esta guía explica cómo asegurarte de que la **interfaz web de LXD** (la
Web UI de Canonical LXD) esté correctamente habilitada y accesible desde
tu máquina local o desde otro equipo (por ejemplo, tu Mac).

------------------------------------------------------------------------

## 1️⃣ Verificar que LXD expone su API HTTPS

Ejecuta en tu host LXD:

``` bash
lxc config get core.https_address
```

Si la salida muestra algo como:

    [::]:8443

o

    0.0.0.0:8443

➡️ significa que el servidor LXD está **escuchando en el puerto 8443**,
y por tanto la Web UI está habilitada.

Si está vacío, actívalo con:

``` bash
sudo lxc config set core.https_address "[::]:8443"
```

Esto permite conexiones HTTPS en todas las interfaces IPv4 e IPv6.

------------------------------------------------------------------------

## 2️⃣ Acceder desde tu Mac o navegador

Abre en tu navegador (Safari, Chrome o Edge):

    https://<IP-del-host-LXD>:8443/ui

Por ejemplo:

    https://192.168.55.43:8443/ui

> 💡 El navegador mostrará una advertencia de **certificado
> autofirmado**.\
> Pulsa "Avanzado" → "Continuar de todos modos".

------------------------------------------------------------------------

## 3️⃣ Confirmar el estado del servicio LXD

Comprueba que el servicio del demonio LXD está activo:

``` bash
sudo snap services lxd
```

La salida debería mostrar algo como:

    Service         Startup  Current  Notes
    lxd.daemon      enabled  active   -

Si no está activo, inícialo manualmente:

``` bash
sudo snap start lxd.daemon
```

Para revisar los últimos logs (por si algo falla):

``` bash
sudo snap logs -n 50 lxd.daemon
```

------------------------------------------------------------------------

## 4️⃣ Verificar el puerto de red

Comprueba que el puerto 8443 está escuchando en tu host:

``` bash
sudo ss -tlnp | grep 8443
```

O desde otra máquina (por ejemplo tu Mac):

``` bash
nc -zv 192.168.55.43 8443
```

Deberías obtener algo como:

    Connection to 192.168.55.43 port 8443 [tcp/pcsync-https] succeeded!

Si no, abre el firewall:

``` bash
sudo ufw allow 8443/tcp
sudo ufw reload
```

------------------------------------------------------------------------

## ✅ Resumen rápido

  ------------------------------------------------------------------------------------
  Componente         Comando / Acción                      Resultado esperado
  ------------------ ------------------------------------- ---------------------------
  LXD escucha en     `lxc config get core.https_address`   `[::]:8443`
  HTTPS                                                    

  Servicio activo    `sudo snap services lxd`              `active`

  UI accesible       `https://IP:8443/ui`                  Abre interfaz web

  Firewall           `sudo ufw status`                     Puerto 8443 permitido
  ------------------------------------------------------------------------------------

------------------------------------------------------------------------

### 🧠 Consejos

-   No es necesario instalar ningún paquete adicional: la **Web UI viene
    incluida** con el snap de LXD.\

-   Puedes añadir usuarios o tokens de confianza con:

    ``` bash
    lxc config trust add
    ```

-   Si usas una red privada o VPN (como VMware o Proxmox), asegúrate de
    que el puerto 8443 no esté bloqueado por el hipervisor.

------------------------------------------------------------------------

## 🚀 Prueba final

Abre tu navegador y ve a:

    https://<tu_IP_de_LXD>:8443/ui/project/default/instance/

Si aparece el panel de **Canonical LXD UI**, ¡estás listo para
administrar tus contenedores y VMs visualmente!
