# Parte 7: Servidor DHCP

> El maestro `ServidorDNS` también hace de servidor DHCP en la red interna. Así el cliente no necesita configuración manual: recibe su IP, el dominio y los dos servidores DNS automáticamente.

---

> **Observación: `isc-dhcp-server` en lugar de Kea**
>
> El profesor pidió que el DHCP se hiciera con **Kea** (`kea-dhcp4-server`), pero cuando nos lo indicó ya habíamos empezado la actividad con el DHCP de siempre, `isc-dhcp-server`. Por eso toda esta parte y sus pruebas están hechas con `isc-dhcp-server`. Tenemos previsto pasarlo a Kea manteniendo la misma red, el mismo rango y los mismos DNS, y actualizaremos esta parte cuando esté hecho.

---

## 1. Instalación

En `ServidorDNS`:

```bash
sudo apt update
sudo apt install isc-dhcp-server
```

Al acabar la instalación el servicio da error al arrancar. Es normal, porque todavía no tiene ninguna subred configurada.

---

## 2. Copia de seguridad

Antes de tocar el fichero de configuración guardamos el original:

```bash
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.BKP
ls /etc/dhcp
```

---

## 3. Configuración (`/etc/dhcp/dhcpd.conf`)

```
authoritative;
default-lease-time 600;
max-lease-time 7200;
option domain-name "lake.local";
option domain-name-servers 192.168.56.10, 192.168.56.11;

subnet 192.168.56.0 netmask 255.255.255.0 {
    range 192.168.56.100 192.168.56.200;
}
```

Explicación:

- `authoritative`: este servidor es el DHCP oficial de la red. Si un cliente pide una IP que no le corresponde, el servidor se la deniega para que pida una nueva.
- `default-lease-time 600` y `max-lease-time 7200`: la concesión dura 10 minutos por defecto y como máximo 2 horas. Al ser un laboratorio usamos tiempos cortos para ver las renovaciones.
- `option domain-name`: el cliente recibe `lake.local` como dominio de búsqueda.
- `option domain-name-servers`: el cliente recibe **los dos DNS**, primero el maestro y después el esclavo. Si el maestro cae, el cliente sigue resolviendo con el esclavo.
- `range`: las IPs que se reparten van de la `.100` a la `.200`. Las de abajo quedan libres para equipos con IP fija, como los servidores (`.10`, `.11`) o `www.lake.local` (`.20`).

**Corrección aplicada:** en la primera versión solo se repartía el DNS del maestro. Añadimos `192.168.56.11` para que el esclavo sirva de respaldo de verdad.

Comprobamos la sintaxis y reiniciamos:

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
systemctl status isc-dhcp-server --no-pager
```

**Captura:** `capturas/parte7/dhcp-status.png` (servicio en `active (running)`)

---

## 4. Verificación en el cliente

En `cliente2` (Ubuntu Desktop, adaptador solo en la red interna):

```bash
ip -br a show enp0s3
resolvectl status
```

Resultado:

- IP recibida: `192.168.56.101/24`, dentro del rango.
- DNS recibidos: `192.168.56.10` y `192.168.56.11`.

**Captura:** `capturas/parte7/cliente-dhcp.png`

En el servidor la concesión queda registrada en el fichero de arrendamientos:

```bash
cat /var/lib/dhcp/dhcpd.leases
```

Ahí aparece la IP `192.168.56.101` con la MAC del cliente (`08:00:27:d2:18:a6`) y las fechas de inicio y fin de la concesión.

---

## 5. Problemas típicos

- **El servicio no arranca:** casi siempre es un `;` o una `}` que falta en `dhcpd.conf`. `sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf` dice la línea exacta.
- **El cliente no recibe IP:** comprobar que el cliente y el servidor están en la misma red interna (`redlab`) y que no hay otro DHCP en esa red.
- **El servidor se pide IP a sí mismo:** era nuestro caso antes de poner `dhcp4: false` en netplan (ver [Parte 5](../parte5-entorno-red-laboratorio/README.md)).

---

## Comprobación en directo

```bash
# En ServidorDNS
systemctl status isc-dhcp-server --no-pager      # active (running)
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf           # sintaxis correcta
cat /var/lib/dhcp/dhcpd.leases                   # concesión de cliente2

# En cliente2: pedir IP de nuevo delante del profesor
sudo nmcli device disconnect enp0s3 && sudo nmcli device connect enp0s3
ip -br a show enp0s3                             # IP del rango .100-.200
resolvectl dns enp0s3                            # 192.168.56.10 y 192.168.56.11
```

---

## Checklist Parte 7

- [x] `isc-dhcp-server` instalado y copia de seguridad del fichero original
- [x] Subred `192.168.56.0/24` con rango `.100`-`.200`
- [x] Dominio `lake.local` y los dos DNS repartidos por DHCP
- [x] Cliente recibe IP y DNS correctamente
- [ ] Captura del servicio activo y del cliente con su IP
- [ ] Migración de `isc-dhcp-server` a Kea
