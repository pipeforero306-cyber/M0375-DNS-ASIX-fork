# Parte 3: Administración - Troubleshooting con DIG y CLI [3p]

> En un entorno profesional, especialmente servidores Linux, la herramienta nslookup se considera obsoleta, siendo dig - Domain Information Groper - el estándar de la industria.

Requisitos: Ejecutado en WSL (Windows Subsystem for Linux) con Ubuntu. Instalación de dig:
```bash
sudo apt install dnsutils
```

---

## 1. Consultas específicas de registros (Usa dig en Linux o WSL)

### 1.1. Registro A

```bash
dig aliexpress.com
```

La salida de dig se divide en secciones:

- **HEADER:** DiG 9.20.24, opcode QUERY, status NOERROR, flags qr rd ra. Indica 1 consulta y 2 respuestas.
- **QUESTION SECTION:** `aliexpress.com. IN A` — la consulta original.
- **ANSWER SECTION:** Devuelve dos registros A:
  - `aliexpress.com. 519 IN A 47.246.75.137`
  - `aliexpress.com. 519 IN A 47.246.111.53`
  - El TTL es de 519 segundos y las dos IPs corresponden a los servidores de AliExpress (round-robin para distribuir carga).
- **Footer:** Query time 12 msec, servidor `10.255.255.254#53`.

El registro A devuelve la dirección IPv4 asociada al dominio. Aliexpress.com devuelve múltiples IPs (round-robin) para distribuir la carga entre servidores.

**Captura:**

![dig A aliexpress.com](../capturas/parte3/dig-a.png)

---

### 1.2. Short

```bash
dig +short aliexpress.com
```

El formato `+short` devuelve únicamente las IPs sin cabeceras ni metadatos:
```
47.246.75.137
47.246.111.53
```

Esto es útil en scripts de Bash porque la salida se puede:
- Asignar a una variable: `IP=$(dig +short aliexpress.com)`
- Usar en pipes: `dig +short aliexpress.com | head -1`
- Usar en condiciones: `if ping -c 1 $(dig +short google.com); then ...`

Con `dig` normal habría que parsear la salida con grep/awk para extraer la IP. Con `+short`, la salida ya está limpia.

**Captura:**

![dig +short aliexpress.com](../capturas/parte3/dig-short.png)

---

### 1.3. MX

```bash
dig MX aliexpress.com
```

El registro MX (Mail Exchange) indica qué servidores de correo reciben los emails del dominio. La respuesta muestra:

```
aliexpress.com. 600 IN MX 10 mx2.mail.aliyun.com.
```

- **Dominio:** aliexpress.com.
- **TTL:** 600 segundos
- **Tipo:** MX
- **Prioridad/Preference:** 10 (número entero; cuanto menor sea, mayor prioridad)
- **Servidor de correo:** mx2.mail.aliyun.com.

El servidor de correo de AliExpress es mx2.mail.aliyun.com (servidor de Alibaba Cloud), con prioridad 10. Al ser el único servidor MX, es el que recibe todo el correo del dominio.

**Captura:**

![dig MX aliexpress.com](../capturas/parte3/dig-mx.png)

---

### 1.4. NS

```bash
dig NS aliexpress.com
```

El registro NS (Name Server) lista los servidores autoritativos para el dominio:

```
aliexpress.com. 2996 IN NS ns1.alibabadns.com.
aliexpress.com. 2996 IN NS ns2.alibabadns.com.
```

Estos son los servidores que tienen la zona del dominio de aliexpress.com y las respuestas definitivas. Cualquier cambio en los registros DNS de aliexpress.com debe hacerse en estos servidores autoritativos. El TTL de los registros NS es de 2996 segundos, relativamente alto porque los servidores autoritativos raramente cambian.

**Captura:**

![dig NS aliexpress.com](../capturas/parte3/dig-ns.png)

---

## 2. Autoridad y Caché (TTL)

### 2.1. ¿Qué diferencia existe entre un registro SOA (Start of Authority) y un registro NS (Name Server)?

| Aspecto | Registro SOA | Registro NS |
|---|---|---|
| Función | Define el inicio de una zona DNS y contiene metadatos de gestión | Lista los servidores autoritativos para una zona |
| Información | Servidor primario, email del administrador, número de serie, tiempos de refresco/reintento/expiración y TTL mínimo | Solo nombres de los servidores autoritativos |
| Cantidad | Un registro SOA por zona | Múltiples registros NS por zona (mínimo 2 recomendado) |
| Uso | Transferencias de zona entre primario y secundarios | Delegación de autoridad; los resolutores consultan estos servidores |

El SOA define cómo se gestiona la zona (metadatos de sincronización). El NS define quién tiene autoridad sobre la zona (qué servidores responden).

**Consulta del registro SOA:**
```bash
dig SOA aliexpress.com
```

Resultado:
```
aliexpress.com. 600 IN SOA ns1.alibabadns.com. hostmaster.alibabadns.com. 2018051019 3600 1200 86400 360
```

- **Servidor primario:** ns1.alibabadns.com.
- **Email del administrador:** hostmaster.alibabadns.com.
- **Serial:** 2018051019 (formato fecha: 2018-05-10, versión 19)
- **Refresh:** 3600 (el servidor secundario comprueba cambios cada hora)
- **Retry:** 1200 (si falla el refresh, reintenta cada 20 minutos)
- **Expire:** 86400 (si no puede contactar al primario en 24h, deja de responder)
- **Minimum TTL:** 360 (TTL mínimo para respuestas negativas)

**Captura:**

![dig SOA aliexpress.com](../capturas/parte3/dig-soa.png)

---

### 2.2. Comportamiento del TTL

Se ejecutaron dos consultas con 10 segundos de diferencia:
```bash
dig aliexpress.com
# esperar 10 segundos
dig aliexpress.com
```

**Primera consulta** (20:57:20): TTL = **267** segundos
**Segunda consulta** (20:57:30): TTL = **257** segundos

El TTL disminuyó en 10 segundos (de 267 a 257), coincidiendo exactamente con el tiempo transcurrido entre ambas consultas.

Esto demuestra que la segunda respuesta viene de la **caché local**, no del servidor autoritativo. El resolutor DNS local cacheó la respuesta de la primera consulta y, en la segunda, devolvió la versión cacheada con el TTL decrementado. Si la respuesta viniera del servidor autoritativo, el TTL volvería a ser el valor original configurado en la zona (en este caso 600, como se vio en la consulta SOA).

La disminución del TTL demuestra que la respuesta proviene de la caché y no del servidor autoritativo. Esto es relevante porque las respuestas cacheadas son más rápidas (4 ms en la segunda consulta vs 12 ms en la primera) pero pueden estar desactualizadas si el dominio cambió de IP recientemente.

**Captura:**

![Comparativa de TTL entre dos consultas](../capturas/parte3/ttl-comparativa.png)

---

## 3. Trazabilidad Completa (Trace)

```bash
dig +trace aliexpress.com
```

**Captura:**

![dig +trace aliexpress.com](../capturas/parte3/dig-trace.png)

Al ejecutar `dig +trace aliexpress.com` se produjo un timeout de conexión con el servidor DNS configurado por defecto en WSL (`10.255.255.254#53`). El comando intentó contactar tres veces con el servidor y, al no recibir respuesta, mostró el mensaje `no servers could be reached`.

Esto ocurre porque `dig +trace` no utiliza el resolver local de la misma forma que las consultas normales; necesita contactar directamente con los servidores raíz, y el DNS interno de WSL no estaba reenviando esas consultas. Para solucionarlo se puede especificar un servidor DNS público explícito:

```bash
dig +trace aliexpress.com @9.9.9.9
```

El proceso que debería seguir el trace es el siguiente:

**Paso 1: Servidores Raíz (Root Servers)**
```
.           518400  IN  NS  a.root-servers.net.
.           518400  IN  NS  b.root-servers.net.
```
Se contacta con los 13 servidores raíz (a.root-servers.net a m.root-servers.net). Son el punto de partida de toda resolución DNS. Su función es redirigir la consulta al TLD correspondiente.

**Paso 2: Servidores del TLD (.com)**
```
com.        172800  IN  NS  a.gtld-servers.net.
com.        172800  IN  NS  b.gtld-servers.net.
```
Los servidores raíz responden con la lista de servidores autoritativos para el TLD .com (gestionados por Verisign). Estos servidores conocen todos los dominios registrados bajo .com.

**Paso 3: Servidores autoritativos del dominio**
```
aliexpress.com.  172800  IN  NS  ns1.alibabadns.com.
aliexpress.com.  172800  IN  NS  ns2.alibabadns.com.
```
Los servidores del TLD .com responden con los servidores autoritativos de aliexpress.com. Son los servidores que tienen la zona del dominio y las respuestas definitivas.

**Paso 4: Resolución final (Registro A)**
```
aliexpress.com.  300  IN  A  47.246.31.83
```
Se consulta a los servidores autoritativos de aliexpress.com, que responden con el registro A (la dirección IP del dominio).

Recorrido completo:
1. Mi PC → Root Servers (.) → "Pregúntale a los servidores de .com"
2. Mi PC → Servidores .com → "Pregúntale a los servidores autoritativos de aliexpress.com"
3. Mi PC → Servidores autoritativos de aliexpress.com → "La IP es 47.246.31.83"

---

## Checklist Parte 3

- [x] dig A ejecutado y capturado
- [x] dig +short ejecutado y explicado
- [x] dig MX ejecutado con prioridades
- [x] dig NS ejecutado con servidores autoritativos
- [x] dig SOA ejecutado
- [x] Diferencia SOA vs NS explicada
- [x] Comportamiento TTL documentado (267 → 257 en 10 segundos)
- [x] dig +trace ejecutado (timeout documentado) y proceso explicado paso a paso
