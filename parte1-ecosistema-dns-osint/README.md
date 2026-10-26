# Parte 1: El ecosistema DNS (OSINT y Web) [2p]

> El sistema DNS es una estructura jerárquica a nivel mundial. Para administrar redes, primero debemos entender quién gestiona cada parte del pastel.

---

## 1. Investigación de Jerarquía

### 1.1. ¿Qué organismo internacional coordina y asigna los parámetros a nivel global del sistema de nombres de dominio e IPs?

**ICANN (Internet Corporation for Assigned Names and Numbers)** es la organización sin ánimo de lucro que coordina a nivel global el sistema de nombres de dominio e IPs. Es responsable de la operación estable y segura de los identificadores únicos de Internet.

Dentro de ICANN, la **IANA (Internet Assigned Numbers Authority)** es el departamento operativo que gestiona directamente la raíz del DNS, los espacios de direcciones IP y los números de protocolo. IANA funciona bajo contrato con ICANN.

Jerarquía de autoridad:
- **ICANN**: coordinación global, política y supervisión
- **IANA**: gestión operativa de la raíz del DNS, IPs y protocolos
- **Root Servers**: 13 servidores raíz repartidos mundialmente (a.root-servers.org a m.root-servers.org)

Fuentes: https://www.icann.org/ | https://www.iana.org/

---

### 1.2. ¿Qué empresa u organismo gestiona (Registry) cada uno de los siguientes dominios de nivel superior (TLD)?

| TLD | Organismo Registry | Descripción |
|---|---|---|
| **.es** | **Red.es** (dominios.es) | Entidad pública empresarial del Ministerio de Asuntos Económicos y Transformación Digital. Gestiona el ccTLD de España. |
| **.cat** | **Fundació puntCAT** | Fundación privada sin ánimo de lucro con sede en Barcelona. Gestiona el dominio dedicado a la lengua y cultura catalana. |
| **.edu** | **EDUCAUSE** | Asociación sin ánimo de lucro estadounidense que gestiona el TLD .edu, reservado para instituciones educativas postsecundarias acreditadas. |
| **.ifp.es** | Subdominio bajo .es gestionado por **Red.es** como registry, delegado a la entidad propietaria (IFP). | El registry sigue siendo Red.es; la gestión del subdominio la realiza la entidad registrante. |

Fuentes:
- .es: https://www.dominios.es/ | https://www.red.es/
- .cat: https://www.puntcat.cat/
- .edu: https://www.educause.edu/

---

## 2. Herramientas OSINT (Whois y DNS Lookup)

### 2.1. ¿Qué información te brinda una consulta Whois sobre un dominio?

**Dominio consultado:** `amazon.com`

Una consulta Whois proporciona información pública sobre el registro de un dominio. Para el caso de **amazon.com**, la consulta devuelve:

- **Datos del registrante:** Amazon Technologies, Inc. — Hostmaster, Amazon Legal Dept. — P.O. Box 8102, Reno, NV 89507, US — Tel: +1.2062664064
- **Datos del registrador:** MarkMonitor Inc. (IANA ID: 292) — Email de abuso: abusecomplaints@markmonitor.com
- **Fechas:**
  - Creación (Registered On): 1994-11-01
  - Expiración (Expires On): 2027-10-31
  - Última actualización (Updated On): 2026-07-31
- **Servidores de nombres (Name Servers):**
  - ns-1447.awsdns-52.org
  - ns-1707.awsdns-21.co.uk
  - ns-264.awsdns-33.com
  - ns-521.awsdns-01.net
- **Estado del dominio (Domain Status):**
  - client delete prohibited
  - client transfer prohibited
  - client update prohibited
  - server delete prohibited
  - server transfer prohibited
  - server update prohibited

Estos estados indican que el dominio está protegido contra eliminaciones, transferencias y modificaciones no autorizadas, tanto a nivel de cliente (MarkMonitor) como de servidor (Verisign como registry de .com). Es algo típico en dominios corporativos críticos.

**Captura:**

![Consulta Whois de amazon.com](../capturas/parte1/whois-amazon.png)

Herramienta utilizada: https://www.whois.com/whois/amazon.com

---

### 2.2. Define brevemente la diferencia entre el Registry de la base de datos y el Registrar (Registrador) del dominio

| Concepto | Definición | Ejemplo |
|---|---|---|
| **Registry (Registro)** | Organización que gestiona y mantiene la base de datos central de un TLD concreto. Tiene el control autoritativo sobre todos los dominios de ese TLD. | Verisign para .com, Red.es para .es, Fundació puntCAT para .cat |
| **Registrar (Registrador)** | Empresa acreditada por el Registry (o por ICANN) que vende dominios al público final. Actúa como intermediario entre el usuario y el Registry. | GoDaddy, Namecheap, Arsys, DonDominio |

**Analogía:** El Registry es como la DGT, que define qué matrículas existen y quién las tiene asignadas. El Registrar es el concesionario donde vas a tramitar el registro de tu coche.

---

### 2.3. Investiga: ¿Qué es DNSSEC y qué problema de seguridad intenta resolver en las resoluciones DNS?

**DNSSEC (Domain Name System Security Extensions)** es un conjunto de extensiones del protocolo DNS que proporciona autenticación de origen e integridad de datos en las respuestas DNS.

**Problema que resuelve:** El DNS original no tiene mecanismos de seguridad. Un atacante puede realizar un ataque de DNS Spoofing / Cache Poisoning, que consiste en inyectar respuestas DNS falsas en la caché de un servidor resolutor, de modo que cuando un usuario intenta acceder a un dominio legítimo, es redirigido a una IP maliciosa (por ejemplo, para phishing).

**Cómo funciona:**
- Utiliza criptografía de clave pública (firmas digitales)
- Cada zona DNS genera un par de claves: KSK (Key Signing Key) y ZSK (Zone Signing Key)
- Los registros DNS se firman digitalmente con la ZSK
- La ZSK se firma con la KSK, que se publica en el registro padre (cadena de confianza)
- La clave pública KSK de la raíz se distribuye de forma segura y es el punto de partida de la cadena
- El resolutor valida las firmas al recibir las respuestas; si la firma no coincide, descarta la respuesta

Registros DNSSEC: DNSKEY, RRSIG, DS, NSEC, NSEC3

Fuentes: https://www.icann.org/dnssec | https://datatracker.ietf.org/doc/html/rfc4033

---

## 3. Rendimiento DNS: Benchmark de servidores DNS

> La página de GRC DNS Benchmark (https://www.grc.com/dns/benchmark.htm) ya no ofrece la versión gratuita v1. La retiraron porque sus rankings estaban desactualizados y ahora solo ofrecen la v2 de pago ($9.95). Por ello se utiliza DNS Jumper como alternativa gratuita.

### 3.1. Herramienta utilizada: DNS Jumper v2.3

DNS Jumper es una herramienta gratuita y portátil (no requiere instalación) para Windows que permite benchmarkar servidores DNS y cambiarlos fácilmente.

Descarga desde: https://www.sordum.org/dns-jumper/

### 3.2. Ejecución del benchmark

1. Abrir DNS Jumper (`DnsJumper.exe`)
2. Ir a la pestaña "DNS más rápido" (Fastest DNS)
3. Pulsar "Iniciar prueba de DNS" (Start DNS Test)
4. Esperar a que complete el test (~1-2 minutos)
5. Resultados ordenados por velocidad de respuesta

### 3.3. Resultados del benchmark

| Posición | Servidor DNS | DNS 1 | DNS 2 | Resultado 1 | Resultado 2 | Empresa | Descripción |
|---|---|---|---|---|---|---|---|
| 1º | US - OpenDNS - 2 | 208.67.220.222 | 208.67.222.220 | 12 ms | 46 ms | Cisco/OpenDNS | Servicio de DNS fundado en 2006 y adquirido por Cisco en 2015. Ofrece DNS rápido y seguro con opciones de filtrado de contenido. |
| 2º | US - Norton ConnectSafe | 199.85.127.10 | 199.85.126.10 | 12 ms | 19 ms | Norton/Symantec | Servicio DNS gratuito de Norton que ofrecía filtrado de malware y contenido. Los servidores siguen respondiendo aunque el soporte está descontinuado. |
| 3º | US - Quad9 Security | 9.9.9.9 | 149.112.112.112 | 12 ms | 14 ms | Quad9 Foundation | Fundación sin ánimo de lucro con sede en Suiza, creada en 2017. Ofrece DNS gratuito que bloquea dominios maliciosos y protege la privacidad. |

Los tres servidores respondieron en 12 ms en el primer test. Para desempatar, nos basamos en el segundo resultado: Quad9 (14 ms) fue el más estable, seguido de Norton (19 ms) y OpenDNS (46 ms).

**DNS seleccionados para la Parte 2:**
- DNS primario: 9.9.9.9 (Quad9 Security)
- DNS secundario: 149.112.112.112 (Quad9 Security)

**Captura:**

![Resultados del benchmark DNS Jumper](../capturas/parte1/dns-benchmark.png)
