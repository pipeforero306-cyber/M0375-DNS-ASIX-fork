# Parte 8: DNSSEC y alta disponibilidad

> El DNS normal no comprueba que una respuesta sea auténtica, y por eso existen ataques como el envenenamiento de caché. Con DNSSEC la zona `lake.local` va firmada y cada respuesta lleva su firma (`RRSIG`). Además comprobamos que, si el maestro cae, el esclavo sigue respondiendo con las firmas intactas.

---

## 1. Firma de la zona en el maestro

La firma se activa en la definición de la zona, en `/etc/bind/named.conf.local` de `ServidorDNS`:

```
zone "lake.local" {
    type master;
    file "/var/lib/bind/db.lake.local";
    dnssec-policy default;
    inline-signing yes;
    key-directory "/var/cache/bind/keys";
};
```

- `dnssec-policy default`: BIND genera y gestiona las claves solo, con la política por defecto. Usa el algoritmo 13 (ECDSA P-256 con SHA-256), que es el que luego aparece en las firmas.
- `inline-signing yes`: nosotros editamos el fichero de zona normal y BIND mantiene aparte una versión firmada. No hay que firmar a mano cada vez que se cambia un registro.
- `key-directory`: carpeta donde se guardan las claves. Tiene que existir y ser del usuario `bind`:

```bash
sudo mkdir -p /var/cache/bind/keys
sudo chown bind:bind /var/cache/bind/keys
sudo systemctl restart named
```

**Zona inversa sin firmar:** la zona `56.168.192.in-addr.arpa` no está firmada. Fue una decisión nuestra: lo que se quería proteger eran los nombres del dominio `lake.local`, y la inversa en un laboratorio no lo necesita.

---

## 2. Verificación de DNSSEC

En el maestro y en el esclavo:

```bash
sudo rndc zonestatus lake.local
```

En los dos sale `secure: yes`, así que el esclavo también ha recibido la zona ya firmada.

**Captura:** `capturas/parte8/zonestatus-secure.png` (una del maestro y otra del esclavo)

Desde el cliente:

```bash
dig www.lake.local +dnssec
dig @192.168.56.10 lake.local DNSKEY +multi
```

- Con `+dnssec`, junto al registro A aparece su `RRSIG`, que es la firma.
- La consulta `DNSKEY` devuelve las claves públicas de la zona, que son las que permiten comprobar esas firmas.

---

## 3. Prueba de alta disponibilidad

Para demostrar que el esclavo sirve de respaldo, **apagamos del todo** `ServidorDNS`:

```bash
sudo poweroff
```

Y desde `cliente2` preguntamos directamente al esclavo:

```bash
dig @192.168.56.11 www.lake.local +dnssec
```

Resultado:

- Estado: `NOERROR`
- Flags: `qr aa rd ra`. El flag `aa` significa que la respuesta es autoritativa: el esclavo responde porque tiene la zona, no porque la haya guardado en caché.
- Respuesta: `192.168.56.20`, con su firma `RRSIG` (algoritmo 13).

**Captura:** `capturas/parte8/alta-disponibilidad.png`

Con esto se ve que el esclavo tiene una copia completa de la zona, firmas incluidas. Como el DHCP reparte los dos DNS (ver [Parte 7](../parte7-servidor-dhcp-isc/README.md)), los clientes siguen resolviendo nombres aunque el maestro no esté.

---

## Comprobación en directo

```bash
# En ServidorDNS y en SlaveDnsServer
sudo rndc zonestatus lake.local                  # secure: yes

# En cliente2
dig www.lake.local +dnssec                       # aparece el RRSIG

# Apagar ServidorDNS (sudo poweroff) y en cliente2:
dig @192.168.56.11 www.lake.local +dnssec        # NOERROR, flag aa y RRSIG
```

Esta prueba conviene dejarla para el final, porque después hay que volver a encender el maestro.

---

## Checklist Parte 8

- [x] Zona `lake.local` firmada con `dnssec-policy default` e `inline-signing`
- [x] `secure: yes` en maestro y esclavo
- [x] `RRSIG` visible en las respuestas desde el cliente
- [x] Prueba con el maestro apagado superada (`NOERROR`, `aa`, `RRSIG`)
- [ ] Capturas de `zonestatus` y de la prueba de alta disponibilidad
