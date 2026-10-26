# Análisis, Resolución y Despliegue del Sistema DNS

## M0375 - Servicios de Red | Ciclo ASIX

Este repositorio recoge la actividad completa sobre DNS. Las partes 1 a 4 estudian el DNS desde fuera: quién lo gestiona, cómo lo usa el sistema operativo, cómo se diagnostica con `dig` y cómo se ve en Wireshark. Las partes 5 a 8 son el laboratorio práctico, donde montamos nuestra propia infraestructura en VirtualBox: servidor DNS maestro y esclavo con BIND9, servidor DHCP y firma de la zona con DNSSEC.

> **Observación sobre el DHCP:** el profesor pidió hacer el servidor DHCP con **Kea**, pero cuando nos lo indicó ya habíamos empezado la actividad con `isc-dhcp-server`, el DHCP de siempre. Todo el laboratorio está hecho y probado con `isc-dhcp-server`, y tenemos previsto migrarlo a Kea con la misma configuración de red. Más detalles en la [Parte 7](parte7-servidor-dhcp-isc/README.md).

---

### Estructura del repositorio

```
M0375-DNS-ASIX/
├── README.md                              (este archivo)
├── parte1-ecosistema-dns-osint/
│   └── README.md                          Parte 1: OSINT y Web [2p]
├── parte2-configuracion-dns-y-cache/
│   └── README.md                          Parte 2: Configuración y Caché [2p]
├── parte3-troubleshooting-dig-cli/
│   └── README.md                          Parte 3: DIG y CLI [3p]
├── parte4-analisis-trafico-wireshark/
│   └── README.md                          Parte 4: Wireshark [3p]
├── parte5-entorno-red-laboratorio/
│   └── README.md                          Parte 5: VMs, red interna y netplan
├── parte6-bind9-dns-maestro-esclavo/
│   └── README.md                          Parte 6: BIND9 maestro y esclavo
├── parte7-servidor-dhcp-isc/
│   └── README.md                          Parte 7: Servidor DHCP
├── parte8-dnssec-alta-disponibilidad/
│   └── README.md                          Parte 8: DNSSEC y alta disponibilidad
└── capturas/
    ├── parte1/
    ├── parte2/
    ├── parte3/
    ├── parte4/
    ├── parte5/
    ├── parte6/
    ├── parte7/
    └── parte8/
```

### Contenido

| Sección | Tema |
|---|---|
| [Parte 1](parte1-ecosistema-dns-osint/README.md) | El ecosistema DNS: jerarquía, OSINT, Whois y benchmark |
| [Parte 2](parte2-configuracion-dns-y-cache/README.md) | Cambio de servidores DNS y caché del sistema |
| [Parte 3](parte3-troubleshooting-dig-cli/README.md) | Troubleshooting con `dig`: registros A, MX, NS, SOA, TTL y `+trace` |
| [Parte 4](parte4-analisis-trafico-wireshark/README.md) | Análisis del tráfico DNS con Wireshark |
| [Parte 5](parte5-entorno-red-laboratorio/README.md) | Entorno del laboratorio: topología, máquinas virtuales, direccionamiento y netplan |
| [Parte 6](parte6-bind9-dns-maestro-esclavo/README.md) | BIND9: zonas directa e inversa en el maestro, transferencia al esclavo y resolución desde el cliente |
| [Parte 7](parte7-servidor-dhcp-isc/README.md) | Servidor DHCP (`isc-dhcp-server`) repartiendo IP, dominio y los dos DNS |
| [Parte 8](parte8-dnssec-alta-disponibilidad/README.md) | Firma de `lake.local` con DNSSEC y prueba con el maestro apagado |

---

### Laboratorio en resumen

| Elemento | Valor |
|---|---|
| Dominio | `lake.local` |
| Red interna | `192.168.56.0/24` (red interna de VirtualBox `redlab`) |
| Software | Ubuntu 24.04.4 LTS, BIND 9.18.39, ISC DHCP Server |
| DNS maestro + DHCP | `ServidorDNS` - `192.168.56.10` |
| DNS esclavo | `SlaveDnsServer` - `192.168.56.11` |
| Cliente | `cliente2` - `192.168.56.101` (por DHCP) |

```
                         Internet
                            |
                   (NAT de VirtualBox)
                            |
           +----------------+----------------+
           |                                 |
 +---------+-----------+          +----------+-----------+
 |     ServidorDNS     |          |    SlaveDnsServer    |
 |  DNS maestro + DHCP |          |      DNS esclavo     |
 |    192.168.56.10    |          |    192.168.56.11     |
 +---------+-----------+          +----------+-----------+
           |                                 |
           +------ Red interna "redlab" -----+
                     192.168.56.0/24
                            |
                   +--------+--------+
                   |    cliente2     |
                   |  192.168.56.101 |
                   +-----------------+
```

**Estado final:**

| Componente | Estado |
|---|---|
| DNS maestro (zona directa e inversa) | Funcionando |
| DNS esclavo con transferencia de zona | Funcionando (sincronizado) |
| DHCP en la red interna | Funcionando (el cliente recibe IP y los dos DNS) |
| DNSSEC en `lake.local` | Funcionando (`secure: yes` en maestro y esclavo) |
| Zona inversa firmada | No (decisión de diseño) |
| Prueba de alta disponibilidad | Superada (el esclavo responde con firmas con el maestro apagado) |
| Migración del DHCP a Kea | Pendiente |

**Correcciones aplicadas durante el laboratorio:**

1. Netplan del maestro: `dhcp4: false` en la interfaz interna, para que el servidor no se pida IP a sí mismo ([Parte 5](parte5-entorno-red-laboratorio/README.md)).
2. Opciones del esclavo igualadas con las del maestro ([Parte 6](parte6-bind9-dns-maestro-esclavo/README.md)).
3. El DHCP reparte los dos DNS, no solo el del maestro ([Parte 7](parte7-servidor-dhcp-isc/README.md)).

---

### Demostración en directo

Orden recomendado: encender `ServidorDNS`, después `SlaveDnsServer` y por último `cliente2`. Cada parte tiene al final su apartado "Comprobación en directo" con los comandos completos; este es el recorrido corto:

| Paso | Dónde | Comando | Qué se ve |
|---|---|---|---|
| 1 | ServidorDNS | `ip -br a` | `192.168.56.10` en la red interna |
| 2 | ServidorDNS | `systemctl status named isc-dhcp-server --no-pager` | Los dos servicios `active (running)` |
| 3 | ServidorDNS | `sudo rndc zonestatus lake.local` | `type: primary`, `secure: yes` |
| 4 | SlaveDnsServer | `sudo rndc zonestatus lake.local` | `type: secondary`, `secure: yes` |
| 5 | cliente2 | `sudo nmcli device disconnect enp0s3 && sudo nmcli device connect enp0s3` | Pide IP de nuevo por DHCP |
| 6 | cliente2 | `ip -br a show enp0s3` y `resolvectl dns enp0s3` | IP del rango y los dos DNS |
| 7 | cliente2 | `dig www.lake.local +short` y `dig web.lake.local` | `192.168.56.20` y el CNAME |
| 8 | cliente2 | `dig @192.168.56.10 lake.local AXFR` | `Transfer failed` (zona protegida) |
| 9 | cliente2 | `dig www.lake.local +dnssec` | Aparece el `RRSIG` |
| 10 | ServidorDNS → cliente2 | `sudo poweroff` en el maestro y `dig @192.168.56.11 www.lake.local +dnssec` en el cliente | `NOERROR`, flag `aa` y `RRSIG` |

---

### Rúbrica de Evaluación

| Criterio | Puntuación | Descripción |
|---|---|---|
| Investigación y OSINT | 40% (4p) | Identifica correctamente las entidades de gestión, explica los roles y configura con éxito la red del sistema operativo. |
| CLI y Troubleshooting | 20% (2p) | Uso impecable de dig, identificando correctamente los tipos de registro (A, MX, NS) y analizando el comportamiento del TTL y las trazas. |
| Análisis de Tráfico | 30% (3p) | Filtrado correcto en Wireshark, identificación exacta de Puertos, Transaction IDs, Flags y análisis riguroso de las tramas. |
| Presentación y Actitud | Penalización | Se restará hasta 2 puntos si el Gitbook carece de capturas de pantalla legibles, si el texto es copiado directamente de internet sin razonamiento propio, o si el formato es descuidado. |

### Formato de Entrega

URL al informe documentado en Gitbook personal.
