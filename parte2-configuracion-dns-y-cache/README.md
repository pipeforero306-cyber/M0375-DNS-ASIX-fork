# Parte 2: Configuración y Caché [2p]

> Todo sistema operativo guarda las resoluciones DNS para no saturar la red.

---

## 1. Cambio de servidores DNS

### 1.1. ¿Cómo puedes ver mediante consola (CLI) qué servidores DNS tienes asignados actualmente en Windows y en Linux?

**Windows (PowerShell o CMD):**
```powershell
ipconfig /all
```

Este comando muestra toda la configuración de red. En la sección del adaptador activo se encuentra la línea "Servidores DNS" con las IPs de los DNS primario y secundario configurados.

**Linux:**
```bash
resolvectl status
```

En distribuciones modernas con systemd-resolved, muestra el estado de los resolutores DNS. La sección "Current DNS Server" indica el servidor en uso. También se puede consultar el archivo `/etc/resolv.conf` que contiene líneas `nameserver` con las IPs de los servidores DNS.

**Captura:**

![ipconfig /all mostrando DNS actuales](../capturas/parte2/ipconfig-all.png)

---

### 1.2. Cambia la configuración de red de tu equipo principal poniendo como DNS primario y secundario los que obtuviste en el Benchmark de la Fase 1

DNS a configurar (obtenidos del benchmark con DNS Jumper):
- DNS primario: 9.9.9.9 (Quad9 Security)
- DNS secundario: 149.112.112.112 (Quad9 Security)

**Pasos en Windows:**
1. Configuración → Red e Internet → Ethernet (adaptador activo)
2. Propiedades → Editar asignación DNS → Manual → IPv4 Activado
3. DNS preferido: 9.9.9.9
4. DNS alternativo: 149.112.112.112
5. Guardar

Verificación del cambio:
```powershell
ipconfig /all
```

Se puede comprobar que en la sección del adaptador Ethernet los servidores DNS ahora son 9.9.9.9 y 149.112.112.112.

**Captura del cambio de DNS:**

![Cambio de DNS a Quad9](../capturas/parte2/cambio-dns.png)

**Captura de verificación con ipconfig /all:**

![Comprobación DNS actualizado](../capturas/parte2/comprobacion-dns.png)

---

## 2. Gestión de la caché DNS (ipconfig / resolvectl)

### 2.1. Muestra una captura de pantalla de algunas direcciones almacenadas en la caché de tu equipo

```powershell
ipconfig /displaydns
```

Este comando muestra el contenido de la caché DNS del cliente de Windows. Cada entrada contiene:
- Nombre de registro (el dominio resuelto)
- Tipo de registro (A = 1, CNAME = 5, etc.)
- Tiempo de vida (TTL): segundos restantes antes de expirar
- Datos: la IP asociada

En la captura se pueden ver entradas como `github.githubassets.com` (registro A, TTL 2119, IP 185.199.110.215) y entradas CNAME de Sony que apuntan a Akamai Edge.

**Captura:**

![Caché DNS con ipconfig /displaydns](../capturas/parte2/cache-dns.png)

---

### 2.2. Vacía la caché de tu equipo y explica para qué es útil esta acción

```powershell
ipconfig /flushdns
```

Respuesta del sistema:
```
Configuración IP de Windows

Se vació correctamente la caché de resolución DNS.
```

**Captura:**

![Flush DNS con ipconfig /flushdns](../capturas/parte2/flush-dns.png)

Vaciar la caché DNS es útil para un administrador de sistemas por varias razones:

1. **Aplicar cambios de IP inmediatamente:** Cuando se migra un servicio a un nuevo servidor, los equipos con la IP antigua en caché no verán el cambio hasta que expire el TTL. Vaciar la caché fuerza la resolución desde el servidor DNS.

2. **Troubleshooting de conectividad:** Si un usuario no puede acceder a un sitio pero otros sí, puede deberse a una entrada DNS corrupta o desactualizada en la caché local.

3. **Prevenir DNS poisoning:** Si un equipo está resolviendo dominios a IPs maliciosas por un ataque de envenenamiento de caché, vaciar la caché elimina las entradas comprometidas.

4. **Forzar uso de servidores DNS nuevos:** Tras cambiar los servidores DNS, vaciar la caché garantiza que las nuevas resoluciones pasen por los nuevos servidores.

5. **Eliminación de entradas NXDOMAIN cacheadas:** Las respuestas negativas (dominio no existe) también se cachean. Si el dominio se registra después, el equipo no lo resolverá hasta que expire el TTL negativo.

---

## Checklist Parte 2

- [x] Comandos CLI para ver DNS (Windows y Linux) ejecutados y capturados
- [x] DNS cambiados a 9.9.9.9 y 149.112.112.112 (Quad9)
- [x] Captura del cambio de DNS
- [x] Captura de verificación con ipconfig /all
- [x] ipconfig /displaydns ejecutado y capturado
- [x] ipconfig /flushdns ejecutado y capturado
