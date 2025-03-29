# Guía Completa para Configurar un Servidor DHCP (Kea) en Ubuntu con Cliente Virtual

Esta guía documenta paso a paso el proceso de instalación, configuración de red y configuración del servidor DHCP Kea en Ubuntu, usando máquinas virtuales con VirtualBox.

---

## ✅ Requisitos Previos

- Oracle VirtualBox instalado.
- Imagen ISO de Ubuntu Server/Desktop.
- Netplan ya viene instalado por defecto en Ubuntu.
- 2 máquinas virtuales:
  - `server1`: actuando como servidor DHCP (con Kea).
  - `cliente1`: cliente que recibirá IP desde `server1`.

---

## 🗕️ Paso 1: Crear las Máquinas Virtuales

### server1

- Red: `Red interna` (nombre: `redclase`)
- Memoria: 2 GB o más
- Almacenamiento: 20 GB (dinámico)

### cliente1

- Red: `Red interna` (nombre: `redclase`)
- Igual configuración de hardware

---

## 🙼 Paso 2: Configurar Red en `server1`

1. Editar archivo Netplan:

   ```bash
   sudo nano /etc/netplan/01-network-manager-all.yaml
   ```

2. Contenido del archivo:

   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       enp0s3:
         dhcp4: no
         addresses:
           - 192.168.100.1/24
   ```

3. Corregir permisos si es necesario:

   ```bash
   sudo chmod 600 /etc/netplan/01-network-manager-all.yaml
   ```

4. Aplicar cambios:

   ```bash
   sudo netplan apply
   ```

5. Verifica IP:

   ```bash
   ip a
   ```

6. Detener y deshabilitar NetworkManager:

   ```bash
   sudo systemctl stop NetworkManager.service
   sudo systemctl disable NetworkManager.service
   ```

---

## 🚜 Paso 3: Instalar Kea DHCP en `server1`

> ⬆️ Si no tienes internet en `server1`, puedes temporalmente poner la red en modo `NAT` en VirtualBox para instalar los paquetes necesarios.

```bash
sudo apt update
sudo apt install isc-kea-dhcp4-server -y
```

---

## 📁 Paso 4: Configurar Kea DHCP

1. Editar el archivo de configuración:

   ```bash
   sudo nano /etc/kea/kea-dhcp4.conf
   ```

2. Agrega esta configuración:

   ```json
   {
     "Dhcp4": {
       "interfaces-config": {
         "interfaces": [ "enp0s3" ]
       },
       "subnet4": [
         {
           "subnet": "192.168.100.0/24",
           "pools": [
             { "pool": "192.168.100.100 - 192.168.100.200" }
           ],
           "option-data": [
             { "name": "routers", "data": "192.168.100.1" },
             { "name": "domain-name-servers", "data": "8.8.8.8" },
             { "name": "domain-name", "data": "red.clase" }
           ]
         }
       ]
     }
   }
   ```

> ❗ Advertencia: `gateway4` está obsoleto en Netplan. Usa mejor `routes` si lo necesitas.

### 📘 Explicación del JSON:

- `interfaces-config`: lista las interfaces donde Kea escuchará peticiones DHCP. En este caso: `enp0s3`.
- `subnet4`: define una subred.
- `pools`: rango de IPs que se asignarán dinámicamente.
- `option-data`:
  - `routers`: define la puerta de enlace.
  - `domain-name-servers`: define el DNS.
  - `domain-name`: define el nombre del dominio asignado.

3. Verifica sintaxis:

   ```bash
   sudo kea-dhcp4 -t -c /etc/kea/kea-dhcp4.conf
   ```

4. Inicia el servicio:

   ```bash
   sudo systemctl restart kea-dhcp4-server
   sudo systemctl enable kea-dhcp4-server
   sudo systemctl status kea-dhcp4-server
   ```

---

## 🧘‍ Paso 5: Configurar Red en `cliente1`

1. Editar Netplan:

   ```bash
   sudo nano /etc/netplan/01-network-manager-all.yaml
   ```

2. Contenido:

   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       enp0s3:
         dhcp4: true
   ```

3. Corregir permisos si es necesario:

   ```bash
   sudo chmod 600 /etc/netplan/01-network-manager-all.yaml
   ```

4. Aplicar y renovar IP:

   ```bash
   sudo netplan apply
   sudo dhclient -v enp0s3
   ```

5. Verificar IP asignada:

   ```bash
   ip a
   ```

   Debe estar dentro del rango 192.168.100.100-200

---

## 🔗 Verificar conectividad

Desde cliente1:

```bash
ping 192.168.100.1
```

Desde server1:

```bash
ping 192.168.100.X  # IP del cliente
```

---

## 🧩 Cambiar nombre de host

```bash
sudo nano /etc/hostname
sudo nano /etc/hosts
```

---

## 🗕️ Estado Final

- `server1`: tiene IP estática 192.168.100.1, ejecutando servidor Kea.
- `cliente1`: recibe IP por DHCP de `server1`.
- Ambas máquinas están en `redclase` (Red Interna).

> 🟡 Recuerda: `server1` debe estar encendido y ejecutando el servicio DHCP para que `cliente1` pueda obtener IP.

---

## 📚 Extras

- Logs de Kea:
  ```bash
  sudo journalctl -fu kea-dhcp4-server
  ```
- Ver puertos activos:
  ```bash
  sudo ss -tuln
  ```
- Reiniciar red:
  ```bash
  sudo systemctl restart systemd-networkd
  ```
- Para copiar y pegar entre host y VM:

  ### Instalar Guest Additions en VirtualBox

  1. Inicia la VM.
  2. Menú VirtualBox: `Dispositivos > Insertar imagen de CD de las Guest Additions`
  3. Se montará un CD (normalmente visible en escritorio o en `/media/cdrom`).
  4. Ejecutar:
     ```bash
     sudo apt update
     sudo apt install build-essential dkms linux-headers-$(uname -r)
     sudo sh /media/<usuario>/VBox_GAs*/VBoxLinuxAdditions.run
     ```
  5. Reinicia la VM.
  6. Luego activa:
     - **Dispositivos > Portapapeles compartido > Bidireccional**
     - **Dispositivos > Arrastrar y soltar > Bidireccional**

---

## ⚠️ Posibles Errores y Soluciones

| Problema | Solución |
|---------|----------|
| `ovsdb-server.service is not running` | ❗ No afecta, puedes ignorarlo. |
| `Permission denied` al aplicar Netplan | Asegúrate de usar `sudo chmod 600` en el archivo YAML. |
| `No DHCPACK received` | Asegúrate de que `server1` esté encendido y Kea corriendo. Verifica `interfaces-config` en Kea. |
| `gateway4 is deprecated` | Usa rutas estáticas o `option-data` con "routers" en Kea. |
| Internet no funciona | Cambiar red temporalmente a `NAT` para hacer `apt install`. Luego volver a `redclase`. |
| El cliente se queda esperando IP | Verifica configuración Netplan, que el servidor esté en marcha y la interfaz sea la correcta. |
| Diferencia entre `renderer` | `networkd`: sistema moderno por systemd (ideal para servidores). `NetworkManager`: usado en escritorios. Usa `networkd` para estas pruebas. |
| No tienes apt-cacher-ng configurado en server | El cliente no podrá instalar usando caché local. No es obligatorio para funcionar, pero útil. |

---

## 🧠 Tabla Explicativa de Comandos y Conceptos

| Comando / Término | Significado |
|------------------|-------------|
| `ip a` | Muestra interfaces de red y direcciones IP. |
| `sudo netplan apply` | Aplica configuración de red especificada en archivos YAML. |
| `sudo dhclient -v enp0s3` | Solicita IP al servidor DHCP. |
| `sudo systemctl status ...` | Muestra estado del servicio. |
| `sudo nano archivo` | Abre archivo para editar con Nano. |
| `sudo chmod 600 archivo` | Cambia permisos del archivo a solo lectura para root. |
| `kea` | Servidor DHCP moderno, modular, sucesor de `isc-dhcp-server`. |
| `NetworkManager` | Servicio para gestionar redes gráficamente (en desktops). |
| `networkd` | Backend ligero para redes controlado por `systemd`. Ideal para servidores. |
| `dhcp4` | Indica si se usará DHCP IPv4 (`true` o `no`). |
| `enp0s3` | Nombre de interfaz de red en Ubuntu. Puede cambiar dependiendo del hardware. |
| `kea-dhcp4.conf` | Archivo JSON de configuración principal de Kea para DHCPv4. |

---

## 🚀 Fin


