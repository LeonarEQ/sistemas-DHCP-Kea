# Guía Completa para Configurar un Servidor DHCP (Kea) y DNS (BIND9) en Ubuntu con Cliente Virtual

Esta guía documenta paso a paso el proceso de instalación, configuración de red, servidor DHCP Kea, y servidor DNS BIND9 en Ubuntu, usando máquinas virtuales con VirtualBox.

---

## ✅ Requisitos Previos

- Oracle VirtualBox instalado.
- Imagen ISO de Ubuntu Server/Desktop.
- Netplan ya viene instalado por defecto en Ubuntu.
- 2 máquinas virtuales:
  - `server1`: actuando como servidor DHCP y DNS.
  - `cliente1`: cliente que recibirá IP y resolución de nombres desde `server1`.

---

## 🧾 Segunda Parte: Configurar y Verificar Servidor DNS (BIND9)

---

## 📦 PASO 6: Instalar BIND9 en `server1`

### 🔌 Cambiar temporalmente a NAT si no hay internet:

```bash
sudo dhclient -v enp0s3
```

> ⚠️ Cuando cambies **de NAT a red interna (redclase)**, es necesario volver a ejecutar `sudo dhclient -v enp0s3` para que la interfaz reciba una IP válida dentro de la red interna y se libere la IP del rango NAT.
> También puede que el sistema vuelva a usar el stub DNS (`127.0.0.53`) y debas corregir `/etc/resolv.conf`.

### 🔧 Instalar BIND9:

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc -y
```

> 📌 Luego vuelve a "Red Interna" (`redclase`) en la configuración de red de VirtualBox.

---

## 🧱 PASO 7: Configurar zonas DNS en `server1`

### 1. Editar archivo de configuración de zonas:

```bash
sudo nano /etc/bind/named.conf.local
```

Agrega lo siguiente:

```conf
zone "red.clase" {
    type master;
    file "/etc/bind/db.red.clase";
};

zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.100.168.192";
};
```

### 2. Crear archivos de zona:

```bash
sudo cp /etc/bind/db.local /etc/bind/db.red.clase
sudo cp /etc/bind/db.127 /etc/bind/db.100.168.192
```

### 3. Editar zona directa (`db.red.clase`):

```bash
sudo nano /etc/bind/db.red.clase
```

```dns
$TTL 604800
@ IN SOA red.clase. root.red.clase. (
              2 ; Serial
         604800 ; Refresh
          86400 ; Retry
        2419200 ; Expire
         604800 ) ; Negative Cache TTL
;
@       IN      NS      red.clase.
@       IN      A       192.168.100.1
server1 IN      A       192.168.100.1
```

### 4. Editar zona inversa (`db.100.168.192`):

```bash
sudo nano /etc/bind/db.100.168.192
```

```dns
$TTL 604800
@ IN SOA red.clase. root.red.clase. (
              2 ; Serial
         604800 ; Refresh
          86400 ; Retry
        2419200 ; Expire
         604800 ) ; Negative Cache TTL
;
@       IN      NS      red.clase.
1       IN      PTR     server1.red.clase.
```

---

## 🛠️ Configurar `/etc/bind/named.conf.options`

Edita el archivo:

```bash
sudo nano /etc/bind/named.conf.options
```

Contenido recomendado:

```conf
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
        1.1.1.1;
    };

    allow-query { any; };
    recursion yes;
    dnssec-validation no;

    listen-on { any; };
    listen-on-v6 { any; };
};
```

### 🔍 Explicación:
- `forwarders`: permite reenviar las consultas que BIND no pueda resolver localmente.
- `allow-query { any; }`: permite que cualquier equipo pregunte al DNS.
- `recursion yes;`: activa la capacidad de resolver dominios que no son de sus zonas.
- `dnssec-validation no;`: evita errores si no se usa DNSSEC.
- `listen-on`: escucha en todas las interfaces disponibles.

Después de modificar:

```bash
sudo named-checkconf
sudo systemctl restart bind9
```

---

## 🚦 PASO 8: Verificar y activar BIND9

### Verificar archivos:

```bash
sudo named-checkzone red.clase /etc/bind/db.red.clase
sudo named-checkzone 100.168.192.in-addr.arpa /etc/bind/db.100.168.192
```

### Reiniciar y habilitar el servicio:

```bash
sudo systemctl restart bind9
sudo systemctl enable named
```

> 📌 Si `enable bind9` da error, usa `enable named`, que es el servicio real.

### Verificar estado:

```bash
sudo systemctl status bind9
```

### Verificar puertos:

```bash
sudo ss -tuln | grep :53
```

Debe mostrar `127.0.0.1:53`, `192.168.100.1:53`, etc.

---

## 🧭 PASO 9: Preparar `server1` antes de probar en cliente

Asegúrate de:

- Kea DHCP está activo: `sudo systemctl status kea-dhcp4-server`
- BIND9 está activo: `sudo systemctl status bind9`
- En `/etc/kea/kea-dhcp4.conf` tienes:

```json
{
  "name": "domain-name-servers",
  "data": "192.168.100.1"
}
```

Esto es lo que enviará a los clientes como servidor DNS.

> 💡 Si al volver a red interna pierdes conectividad o DNS, revisa la IP con `ip a` y edita el archivo `/etc/resolv.conf` para apuntar al nuevo DNS (por ejemplo `192.168.100.101`).

---

## 💡 Explicación del JSON (DHCP):

- `interfaces-config`: interfaces por donde Kea escucha.
- `subnet4`: subred usada para asignar IP.
- `pools`: rango de IP dinámicas.
- `option-data`:
  - `routers`: puerta de enlace.
  - `domain-name-servers`: IP del servidor DNS.
  - `domain-name`: dominio local (red.clase).

---

## 🧪 PASO 10: Probar desde `cliente1`

1. Obtener IP por DHCP:

```bash
sudo dhclient -v enp0s3
```

2. Ver `/etc/resolv.conf`:

```bash
cat /etc/resolv.conf
```

Si sale `127.0.0.53`, reemplazar por:

```bash
sudo rm /etc/resolv.conf
echo "nameserver 192.168.100.1" | sudo tee /etc/resolv.conf
```

3. Probar resolución:

```bash
nslookup red.clase

dig red.clase

dig -x 192.168.100.1
```

> Resultado esperado:
>
> ```
> ;; ANSWER SECTION:
> red.clase.   604800   IN  A   192.168.100.1
> ```

---

## 🔍 ¿Por qué no puedes resolver dominios externos en red interna?

Cuando una VM está conectada a una **red interna**, solo puede comunicarse con otras VMs en esa misma red. **No hay conexión con Internet** ni salida hacia los DNS públicos (como 8.8.8.8).

Por eso, si tu servidor `server1` no tiene acceso a Internet y no puedes usar `forwarders`, **no podrá resolver dominios como `google.com`**, aunque tengas configurado BIND con zonas internas.

> 💡 Este comportamiento es normal y aceptado en entornos de laboratorio, donde se simula una red cerrada sin acceso a Internet.

---

## 🧨 Posibles Errores y Soluciones

| Error | Causa | Solución |
|-------|--------|----------|
| `SERVFAIL` en dig | Zona no definida correctamente | Revisar `named.conf.local` y reiniciar BIND |
| `zonestatus not found` | Zona no activa aún | Asegúrate de que esté declarada y revisa el nombre |
| `network unreachable resolving` | No hay internet | No afecta si es red local |
| `127.0.0.53` en resolv.conf | Systemd-resolved en uso | Reemplaza resolv.conf manualmente |

---

## 📚 Tabla de Comandos y Conceptos

| Comando / Término | Explicación |
|------------------|-------------|
| `named.conf.local` | Archivo donde declaras las zonas DNS |
| `db.red.clase` | Zona directa: nombre → IP |
| `db.100.168.192` | Zona inversa: IP → nombre |
| `dig`, `nslookup` | Herramientas para consultar DNS |
| `systemctl status bind9` | Verifica si BIND9 está corriendo |
| `ss -tuln | grep :53` | Verifica puertos en uso para DNS |

---

## ✅ Resultado Esperado

Desde cliente1:

- `dig red.clase` devuelve 192.168.100.1
- `dig -x 192.168.100.1` devuelve server1.red.clase
- `nslookup red.clase` también resuelve correctamente
- Cliente recibió IP y DNS por DHCP

¡Todo listo para continuar con lo siguiente del laboratorio! 🚀

