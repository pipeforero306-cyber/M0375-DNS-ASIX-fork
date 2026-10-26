# Parte 5: Entorno del laboratorio (VirtualBox y red interna)

> Después de estudiar cómo funciona el DNS desde fuera, montamos nuestra propia infraestructura: un servidor DNS maestro que además reparte IPs por DHCP, un servidor DNS esclavo y un cliente, todos conectados por una red interna de VirtualBox.

---

## 1. Planificación

Antes de instalar nada decidimos la red y los nombres para que todas las máquinas encajaran:

| Elemento | Valor |
|---|---|
| Dominio | `lake.local` |
| Red interna | `192.168.56.0/24` (red interna de VirtualBox llamada `redlab`) |
| Sistema operativo | Ubuntu 24.04.4 LTS (Server en los servidores, Desktop en el cliente) |
| Software | BIND 9.18.39, ISC DHCP Server (`isc-dhcp-server`) |
| Virtualizador | Oracle VirtualBox |
| Usuario en todas las VMs | `lake` |

Cada servidor tiene dos adaptadores: uno en **NAT** para tener internet e instalar paquetes, y otro en la **red interna** `redlab`, que es donde funcionan los servicios. El cliente solo está en la red interna, así que todo lo que necesita (IP y DNS) lo recibe de nuestros servidores.

---

## 2. Topología

```
                         Internet
                            |
                   (NAT de VirtualBox)
                            |
           +----------------+----------------+
           |                                 |
   enp0s3 (NAT) 10.0.2.15           enp0s8 (NAT) 10.0.3.15
           |                                 |
 +---------+-----------+          +----------+-----------+
 |     ServidorDNS     |          |    SlaveDnsServer    |
 |  DNS maestro + DHCP |          |      DNS esclavo     |
 |  enp0s8 .56.10      |          |  enp0s3 .56.11       |
 +---------+-----------+          +----------+-----------+
           |                                 |
           +------ Red interna "redlab" -----+
                     192.168.56.0/24
                            |
                   +--------+--------+
                   |    cliente2     |
                   |  enp0s3 (DHCP)  |
                   |  192.168.56.101 |
                   +-----------------+
```

---

## 3. Máquinas virtuales

| VM | Hostname | RAM | CPU | Disco | Adaptador 1 | Adaptador 2 |
|---|---|---|---|---|---|---|
| `ServidorDNS` | `ServidorDNS` | 5737 MB | 2 | 25 GB (VDI) | NAT | Red interna `redlab` |
| `SLAVE SERVER DNS` | `SlaveDnsServer` | 3840 MB | 2 | 25 GB (VDI) | Red interna `redlab` | NAT |
| Cliente | `cliente2` | 2048 MB | 1 | 25 GB (VDI) | Red interna `redlab` | - |

ISOs utilizadas:
- Servidores: `ubuntu-24.04.4-live-server-amd64.iso`
- Cliente: `ubuntu-24.04.4-desktop-amd64.iso`

En el esclavo los adaptadores quedaron al revés que en el maestro (la red interna es `enp0s3` y la NAT es `enp0s8`). No afecta al funcionamiento, pero hay que tenerlo en cuenta al configurar netplan.

---

## 4. Direccionamiento

| VM | Interfaz | Función | Dirección | MAC |
|---|---|---|---|---|
| ServidorDNS | `enp0s3` | NAT | `10.0.2.15/24` (DHCP de VirtualBox) | `08:00:27:f5:63:a1` |
| ServidorDNS | `enp0s8` | Red interna | `192.168.56.10/24` (fija) | `08:00:27:6a:11:a4` |
| SlaveDnsServer | `enp0s3` | Red interna | `192.168.56.11/24` (fija) | `08:00:27:97:9c:40` |
| SlaveDnsServer | `enp0s8` | NAT | `10.0.3.15/24` (DHCP de VirtualBox) | `08:00:27:e7:9b:53` |
| cliente2 | `enp0s3` | Red interna | `192.168.56.101/24` (DHCP) | `08:00:27:d2:18:a6` |

Los dos servidores llevan IP fija en la red interna. Un servidor DNS o DHCP no puede depender de que otro le dé la IP, porque los clientes tienen que saber siempre dónde encontrarlo.

---

## 5. Configuración de red del maestro (netplan)

Fichero `/etc/netplan/50-cloud-init.yaml` en `ServidorDNS`:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses: [192.168.56.10/24]
```

Para aplicarlo:

```bash
sudo netplan apply
ip -br a
```

**Corrección aplicada:** al principio `enp0s8` no tenía `dhcp4: false`, así que el servidor pedía IP por DHCP a su propio servicio además de tener la fija. Con `dhcp4: false` la interfaz se queda solo con `192.168.56.10`.

En el esclavo se hizo lo mismo con su interfaz interna (`enp0s3` con `192.168.56.11/24` y `dhcp4: false`).

**Captura:** `capturas/parte5/ip-maestro.png` (salida de `ip -br a` en `ServidorDNS`)

---

## Comprobación en directo

```bash
# En ServidorDNS
ip -br a                                   # enp0s8 con 192.168.56.10
cat /etc/netplan/50-cloud-init.yaml        # dhcp4: false en enp0s8

# En SlaveDnsServer
ip -br a                                   # enp0s3 con 192.168.56.11
```

---

## Checklist Parte 5

- [x] Red y dominio planificados (`192.168.56.0/24`, `lake.local`)
- [x] Tres VMs creadas con sus adaptadores (NAT + red interna `redlab`)
- [x] IP fija en la red interna de los dos servidores
- [x] Netplan del maestro corregido (`dhcp4: false` en `enp0s8`)
- [ ] Captura de `ip -br a` en el maestro
