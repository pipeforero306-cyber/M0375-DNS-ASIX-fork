# Parte 4: Análisis de Tráfico de Red (Wireshark) [3p]

> Esta parte la realiza un compañero de equipo.

---

## 1. Captura de tráfico DNS

### Pasos:

1. Abrir Wireshark
2. Seleccionar la tarjeta de red principal
3. Iniciar la captura
4. Aplicar el filtro: `dns` o `udp.port == 53`
5. Abrir una terminal y ejecutar:
   ```bash
   nslookup -type=mx google.com
   ```
6. Detener la captura
7. Localizar el intercambio Query → Response

**Captura general:** `capturas/parte4/wireshark-captura-general.png`

---

## 2. Preguntas a responder

### 2.1. Capa de Transporte

¿Qué protocolo se utiliza (TCP o UDP)?

DNS usa UDP por defecto porque las consultas son pequeñas (normalmente <512 bytes), UDP es más rápido al no requerir handshake (establecimiento de conexión), y no necesita conexión. TCP se usa solo para respuestas grandes (transferencias de zona) o cuando se supera el límite de 512 bytes sin EDNS0.

**Captura:** `capturas/parte4/wireshark-protocolo.png`

---

### 2.2. Puertos

- Puerto de origen (cliente): dinámico, >1024 (ej. 51234)
- Puerto de destino (servidor): 53 (well-known)

**Captura:** `capturas/parte4/wireshark-puertos.png`

---

### 2.3. Identificador de transacción (Transaction ID)

En la sección "Domain Name System" de Wireshark, el Transaction ID vincula la petición (Query) con la respuesta (Response). Ambos paquetes comparten el mismo identificador.

**Captura:** `capturas/parte4/wireshark-transaction-id.png`

---

### 2.4. Flags

En el paquete de Response, desplegar la sección Flags:
- Authoritative Answer (AA): ¿Está a 0 o a 1?
- Si está a 0: la respuesta no es autoritativa (viene de un resolver/caché)
- Si está a 1: la respuesta es autoritativa (viene directamente del servidor autoritativo del dominio)

**Captura:** `capturas/parte4/wireshark-flags.png`

---

### 2.5. Respuestas (Answers)

Desplegar el bloque de respuestas del registro MX de google.com. El servidor de correo con mayor prioridad es el que tiene el número de preference más bajo.

**Captura:** `capturas/parte4/wireshark-answers.png`

---

## Checklist Parte 4

- [ ] Captura de tráfico DNS realizada
- [ ] Filtro dns aplicado
- [ ] Protocolo de transporte identificado (UDP)
- [ ] Puertos origen y destino documentados
- [ ] Transaction ID identificado
- [ ] Flags analizadas (Authoritative Answer)
- [ ] Answers del MX analizadas
- [ ] Todas las capturas con recortes del panel de detalles
