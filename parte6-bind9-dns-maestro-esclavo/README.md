# Parte 6: Servidor DNS maestro y esclavo con BIND9

> El maestro (`ServidorDNS`, 192.168.56.10) es el que tiene las zonas originales de `lake.local`. El esclavo (`SlaveDnsServer`, 192.168.56.11) recibe una copia por transferencia de zona (AXFR) y puede responder igual que el maestro.

---

## 1. Instalación

En los dos servidores:

```bash
sudo apt update
sudo apt install bind9 bind9utils dnsutils
```

---

## 2. Configuración del maestro

### 2.1. Opciones generales (`/etc/bind/named.conf.options`)

```
options {
    directory "/var/cache/bind";
    listen-on { 127.0.0.1; 192.168.56.10; };
    allow-query { localhost; 192.168.56.0/24; };
    allow-transfer { 192.168.56.11; };
    recursion yes;
    allow-recursion { localhost; 192.168.56.0/24; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    dnssec-validation auto;
};
```

Qué hace cada línea:

- `listen-on`: el servidor solo escucha en localhost y en su IP de la red interna, no en la NAT.
- `allow-query`: solo responde a la propia máquina y a equipos de `192.168.56.0/24`.
- `allow-transfer`: la zona completa solo se puede copiar desde el esclavo (`192.168.56.11`). Si cualquier equipo pudiera pedir un AXFR, podría sacar todos los nombres e IPs de la red.
- `recursion` y `allow-recursion`: resuelve dominios de fuera (por ejemplo `google.com`), pero solo para nuestra red, para no convertirse en un resolver abierto.
- `forwarders`: lo que no conoce lo pregunta a `8.8.8.8` y `1.1.1.1`.
- `dnssec-validation auto`: valida las firmas DNSSEC de las respuestas que recibe.

### 2.2. Zonas (`/etc/bind/named.conf.local`)

```
zone "lake.local" {
    type master;
    file "/var/lib/bind/db.lake.local";
    dnssec-policy default;
    inline-signing yes;
    key-directory "/var/cache/bind/keys";
};

zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.56";
};
```

- `lake.local` es la **zona directa** (nombre → IP). Está firmada con DNSSEC; eso se explica en la [Parte 8](../parte8-dnssec-alta-disponibilidad/README.md).
- `56.168.192.in-addr.arpa` es la **zona inversa** (IP → nombre) de la red `192.168.56.0/24`.
- El fichero de `lake.local` va en `/var/lib/bind/` y no en `/etc/bind/` porque, con la firma en línea, BIND necesita poder escribir junto a él los ficheros firmados.

Registros que comprobamos desde el cliente:

| Nombre | Tipo | Valor |
|---|---|---|
| `www.lake.local` | A | `192.168.56.20` |
| `web.lake.local` | CNAME | `www.lake.local` |

Antes de reiniciar el servicio comprobamos la sintaxis:

```bash
sudo named-checkconf
sudo named-checkzone lake.local /var/lib/bind/db.lake.local
sudo systemctl restart named
```

---

## 3. Configuración del esclavo

### 3.1. Opciones generales (`/etc/bind/named.conf.options`)

```
options {
    directory "/var/cache/bind";
    listen-on { 127.0.0.1; 192.168.56.11; };
    allow-query { localhost; 192.168.56.0/24; };
    recursion yes;
    allow-recursion { localhost; 192.168.56.0/24; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    dnssec-validation auto;
};
```

**Corrección aplicada:** al principio el esclavo tenía opciones distintas a las del maestro. Las dejamos iguales (seguridad, recursión y reenviadores) para que el cliente reciba el mismo comportamiento pregunte a quien pregunte. La única diferencia es que el esclavo no lleva `allow-transfer`, porque nadie copia zonas de él.

### 3.2. Zonas (`/etc/bind/named.conf.local`)

```
zone "lake.local" {
    type slave;
    masters { 192.168.56.10; };
    file "/var/cache/bind/db.lake.local";
};

zone "56.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.56.10; };
    file "/var/cache/bind/db.192.168.56";
};
```

Con `type slave` y `masters` el esclavo pide las dos zonas al maestro y las guarda en `/var/cache/bind/`. No hay que crear ningún fichero de zona a mano en el esclavo.

---

## 4. Verificación

### 4.1. Transferencia de zona

En el esclavo:

```bash
sudo rndc zonestatus lake.local
ls -l /var/cache/bind/
sudo journalctl -u named --no-pager | grep -i transfer | tail -5
```

Los ficheros `db.lake.local` y `db.192.168.56` aparecen en `/var/cache/bind/` y en el log se ve la transferencia desde `192.168.56.10` terminada.

**Captura:** `capturas/parte6/zonestatus-esclavo.png`

### 4.2. Resolución desde el cliente

En `cliente2`:

```bash
dig www.lake.local +short
dig web.lake.local
```

- `www.lake.local` devuelve `192.168.56.20`.
- `web.lake.local` devuelve el CNAME hacia `www.lake.local` y después su IP.

**Captura:** `capturas/parte6/dig-cliente.png`

### 4.3. La transferencia está protegida

Desde el cliente, intentamos copiar la zona entera:

```bash
dig @192.168.56.10 lake.local AXFR
```

Sale `Transfer failed`, porque solo el esclavo está en `allow-transfer`.

---

## Comprobación en directo

```bash
# En ServidorDNS
sudo named-checkconf && echo "BIND OK"
systemctl status named --no-pager

# En SlaveDnsServer
sudo rndc zonestatus lake.local                       # type: secondary
sudo journalctl -u named --no-pager | grep -i transfer | tail -5

# En cliente2
dig www.lake.local +short                             # 192.168.56.20
dig web.lake.local                                    # CNAME
dig @192.168.56.10 lake.local AXFR                    # Transfer failed
```

---

## Checklist Parte 6

- [x] BIND9 instalado en maestro y esclavo
- [x] Zona directa `lake.local` y zona inversa `56.168.192.in-addr.arpa` en el maestro
- [x] Transferencia de zona limitada al esclavo (`allow-transfer`)
- [x] Esclavo sincronizado con las dos zonas
- [x] Opciones del esclavo igualadas con las del maestro
- [x] Resolución de `www.lake.local` (A) y `web.lake.local` (CNAME) desde el cliente
- [ ] Capturas de `rndc zonestatus` en el esclavo y de `dig` en el cliente
