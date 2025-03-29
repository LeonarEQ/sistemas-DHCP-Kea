# 📘 Guía 3: Servidor Web Apache Integrado con DNS

**Desde PASO 11 en adelante del PDF del profesor.** Esta guía continúa desde la configuración del servidor DNS (BIND9) y DHCP (Kea), agregando ahora un servidor web Apache que será accesible por nombre de dominio local (resuelto vía DNS).

---

## 🔹 PASO 11: Instalar Apache en `server1`

1. Cambia temporalmente la red de `server1` a NAT si no tienes acceso a internet.
2. Ejecuta:

```bash
sudo apt update
sudo apt install apache2 -y
```

3. Regresa la red a "Red Interna" (`redclase`).

---

## 🔹 PASO 12: Crear sitio web local

1. Crea la carpeta y el archivo HTML:

```bash
sudo mkdir -p /var/www/red.clase
sudo nano /var/www/red.clase/index.html
```

Contenido:
```html
<h1>Servidor Apache funcionando correctamente</h1>
```

2. Crea archivo de configuración para el sitio:

```bash
sudo nano /etc/apache2/sites-available/red.clase.conf
```

Contenido:
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName red.clase
    DocumentRoot /var/www/red.clase
</VirtualHost>
```

3. Activar sitio y reiniciar apache:

```bash
sudo a2ensite red.clase.conf
sudo systemctl reload apache2
```

Verifica que el servicio esté activo:
```bash
sudo systemctl status apache2
```

---

## 🔹 PASO 13: Probar acceso desde `cliente1`

### Verifica DNS:
```bash
cat /etc/resolv.conf
```
Debe decir:
```
nameserver 192.168.100.101
```
(Si no, reemplaza con: `sudo rm /etc/resolv.conf && echo "nameserver 192.168.100.101" | sudo tee /etc/resolv.conf`)

### Realiza pruebas:

```bash
nslookup red.clase
```
Debe responder con:
```
Name:    red.clase
Address: 192.168.100.1
```

```bash
dig red.clase
```
```bash
dig -x 192.168.100.1
```

Ambos deben devolver una respuesta con `NOERROR`, y en la sección ANSWER debe aparecer:
- red.clase → 192.168.100.1
- 192.168.100.1 → server1.red.clase

### Abre navegador Firefox (cliente1):
```
red.clase
```
Debe mostrar: `Servidor Apache funcionando correctamente`

---

## 📌 Notas adicionales

- El DNS está resolviendo desde el servidor local BIND9.
- Apache sirve contenido según el nombre virtual configurado.
- Toda la red está en "Red Interna" sin acceso a internet externo.

---

## 🧨 Posibles errores y soluciones

| Error | Causa | Solución |
|-------|--------|----------|
| ERR_NAME_NOT_RESOLVED | DNS mal configurado | Verifica `/etc/resolv.conf` |
| Apache página vacía | Index mal ubicado | Revisa ruta del `DocumentRoot` |
| SERVFAIL en `dig` | Zona DNS no encontrada | Verifica `named.conf.local` y reinicia BIND9 |

---

## ✅ Resultado esperado

Desde `cliente1`:
- Puedes abrir `red.clase` en el navegador.
- `nslookup`, `dig`, y `ping` a `red.clase` funcionan.
- Apache devuelve el HTML correcto.

