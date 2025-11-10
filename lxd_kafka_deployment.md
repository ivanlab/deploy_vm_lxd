# 🚀 Despliegue de Kafka en Contenedor LXC (Opción Ligera y Práctica)

## 🧩 Objetivo
Desplegar un contenedor Ubuntu 22.04 dentro de LXD para ejecutar Apache Kafka, en la misma red (`lxdbr0`) que el servidor Windows 2022, de modo que ambos puedan comunicarse internamente.

---

## 🧱 1. Crear el contenedor conectado a la red `lxdbr0`

```bash
lxc launch ubuntu:22.04 kafka -p default
lxc config device add kafka eth0 nic network=lxdbr0 name=eth0
```

Esto:
- Lanza un contenedor Ubuntu 22.04 con el perfil `default`.
- Añade la interfaz `eth0` conectada a la red `lxdbr0` (la misma red usada por la VM Windows).

---

## 🧠 2. Acceder al contenedor

```bash
lxc exec kafka bash
```

---

## ⚙️ 3. Instalar dependencias y Kafka

```bash
apt update
apt install -y openjdk-17-jdk wget
cd /opt
wget https://downloads.apache.org/kafka/3.8.0/kafka_2.13-3.8.0.tgz
tar -xzf kafka_2.13-3.8.0.tgz
mv kafka_2.13-3.8.0 kafka
```

---

## ▶️ 4. Iniciar Zookeeper y Kafka

```bash
/opt/kafka/bin/zookeeper-server-start.sh -daemon /opt/kafka/config/zookeeper.properties
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```

---

## 🌐 5. Verificar conectividad desde el host

```bash
lxc list
```

Ejemplo de salida:

```
| kafka  | RUNNING | 10.191.69.45 (eth0) |
```

---

## 💻 6. Comprobar acceso desde Windows Server

En tu VM Windows Server (que tiene IP 10.191.69.218, por ejemplo):

### PowerShell
```powershell
ping 10.191.69.45
```

### Kafka Client
Conéctate usando la IP del contenedor:
```
bootstrap.servers=10.191.69.45:9092
```

---

## 🔁 7. Hacer el contenedor persistente

```bash
lxc config set kafka boot.autostart true
```

---

## ✅ Resultado Esperado

- Contenedor `kafka` corriendo sobre Ubuntu 22.04.
- Kafka y Zookeeper activos dentro del contenedor.
- Comunicación bidireccional entre la VM Windows Server 2022 y el contenedor Kafka vía `lxdbr0`.
- Red totalmente aislada del host, pero con salida a Internet mediante NAT de LXD.

---

## 🧾 Notas finales

- Puedes listar redes y comprobar su configuración con:
  ```bash
  lxc network list
  lxc network show lxdbr0
  ```
- Si deseas exponer Kafka fuera del bridge (por ejemplo, hacia la red física), será necesario usar un *proxy device* o una *macvlan* adicional.
