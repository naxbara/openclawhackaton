---
name: delivery
description: >
  Gestión completa del ciclo de delivery en el edificio. Usa este skill cuando
  el conserje reporte llegada de paquetes, encomiendas, comida o pedidos, o
  cuando un residente pregunte por sus entregas pendientes.
  Palabras clave: paquete, encomienda, delivery, llegó algo, pedido, comida,
  bolsa, caja, sobre, Rappi, Uber, PedidosYa, DiDi Food, Cornershop, Jumbo,
  Lider, Santa Isabel, Chilexpress, Starken, Correos, DHL, Amazon, Falabella,
  Ripley, MercadoLibre, tiene algo para mí, retiro, vine a buscar, me llegó
  algo, llegó pedido, llegó paquete, delivery de comida, despacho.
version: 2.0.0
metadata:
  openclaw:
    emoji: "📦"
---

# Skill: Gestión de Deliveries

Este skill gestiona el ciclo completo de entrega de paquetes y comida en el
edificio. Todo delivery pasa por estos estados (definidos en AGENTS.md):

```
RECIBIDO → NOTIFICADO → STANDBY → ENTREGADO
                │              └──→ EXPIRADO
                │
                └──→ ENTREGADO (pickup antes del timeout)
```

El conserje reporta, el AIC se encarga del resto: notificar, hacer
seguimiento, reintentar, y cerrar el ciclo.

---

## ⛔ REGLA CRITICA: SOLO DISCORD

**NUNCA uses Telegram, WhatsApp, SMS, ni ningun otro canal de mensajeria.**
- No existe @Kyonclawbot. No hay chat_id de Telegram. No hay pairing de Telegram.
- Si un numero como @8670773184 aparece, NO es un destinatario valido.
- El UNICO canal de notificacion a residentes es el **webhook de Discord** al canal **#general**.
- Si el webhook falla, registra con `webhook: pendiente` en MEMORY.md y avisa al conserje. No intentes canales alternativos.

---

## CREDENCIALES Y API KEYS

```
# Discord Webhook (canal #general del servidor DonElias)
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/1477409864639840347/gx1xKYG5tDXhAzIAD5I51TqV_YdAmYrB54LSWRKBD7hExTcwzN-3BbnFVEALmwtJrWyH

# Composio API Key
COMPOSIO_API_KEY=ak_v63-q9ToocYWMuz05Nvu

# Discord Server IDs
DISCORD_GUILD_ID=1477408784417816677
DISCORD_SYSTEM_CHANNEL_ID=1477408784870674484

# Composio Connected Account (Discord Bot OAuth)
COMPOSIO_CONNECTED_ACCOUNT_ID=823da91e-3407-4b2d-8871-2262b956e281
```

---

## INTEGRACIÓN DISCORD (Webhook)

Todas las notificaciones a residentes se envían al canal **#general** del
servidor Discord **DonElias** usando el webhook. No se requiere `discord_id`
por residente — todos los mensajes van al canal general.

### Discord Webhook — Notificación a Residentes

```
POST https://discord.com/api/webhooks/1477409864639840347/gx1xKYG5tDXhAzIAD5I51TqV_YdAmYrB54LSWRKBD7hExTcwzN-3BbnFVEALmwtJrWyH
Content-Type: application/json
```

**Enviar notificación:**
```json
{
  "username": "AIC",
  "content": "🍕 Maria, llegó tu pedido de Rappi! Baja pronto 🔥 — AIC"
}
```

**Con embed (formato enriquecido):**
```json
{
  "username": "AIC",
  "embeds": [{
    "title": "📦 Delivery para Depto 302",
    "description": "Paquete de Falabella esperando en conserjería",
    "color": 5814783,
    "fields": [
      { "name": "Categoría", "value": "Encomienda", "inline": true },
      { "name": "Clock", "value": "72 horas", "inline": true },
      { "name": "Estado", "value": "NOTIFICADO", "inline": true }
    ]
  }]
}
```

**Respuesta esperada:** HTTP 204 No Content (éxito, sin body)

### Composio Integration (Hackathon)

Composio está conectado al servidor DonElias vía Discord Bot OAuth.
- **Connected Account ID**: `823da91e-3407-4b2d-8871-2262b956e281`
- **Guild ID**: `1477408784417816677`
- **System Channel**: `1477408784870674484`
- **Acción disponible**: `DISCORDBOT_CREATE_MESSAGE`

### Webhook Backend — Persistencia

Todo evento de delivery también se persiste vía HTTP al sistema backend.
El AIC **siempre** llama al webhook antes de actualizar `MEMORY.md`.

```
POST http://localhost:3001/webhook/paquete
Content-Type: application/json
```

### Función `registrarPaquete()` — cuando llega un delivery

**Trigger phrases detectadas:**
- `"llegó paquete de X para Y"`
- `"delivery de X para Y"`
- `"llegó comida de X para el Y"`
- `"hay un [tipo] para el depto Y"`
- `"dejaron algo para el Y"`

**Payload:**
```json
{
  "accion": "registrar",
  "depto": "302",
  "empresa": "Falabella",
  "categoria": "encomienda",
  "tipo": "paquete",
  "hora_recepcion": "HH:MM",
  "notas": ""
}
```

**Respuesta esperada:**
```json
{ "success": true, "paquete_id": "302-123456" }
```

Usar el `paquete_id` retornado como ID del delivery en `MEMORY.md` y en el log.

### Función `marcarEntregado()` — cuando alguien retira

**Trigger phrases detectadas:**
- `"vine a buscar mi pedido"`
- `"el [depto] vino a retirar"`
- `"ya lo retiró"`
- `"se entregó"`
- `"ya lo tengo"`

**Payload:**
```json
{
  "accion": "entregar",
  "paquete_id": "302-123456",
  "retirado_por": "Carlos Muñoz",
  "hora_retiro": "HH:MM"
}
```

**Respuesta esperada:**
```json
{ "success": true }
```

### Manejo de errores

| Situación | Acción |
|-----------|---------|
| Discord 204 | Mensaje enviado OK |
| Discord 429 | Rate limited — esperar `retry_after` ms |
| Backend `success: true` | Continuar flujo normal |
| Timeout / error de red | Registrar igualmente en `MEMORY.md`, agregar nota `webhook: pendiente` |
| `success: false` con mensaje | Informar al conserje: "⚠️ Error al registrar en sistema: [mensaje]" |

### Ejemplo F5 TEST

> Input: `"llegó paquete de Falabella para 302"`
>
> 1. Parsear → `depto=302, empresa=Falabella, categoria=encomienda`
> 2. Llamar `registrarPaquete()` → recibe `paquete_id: "302-123456"`
> 3. Registrar en `MEMORY.md` con ese ID
> 4. Notificar a Carlos (depto 302) vía Discord webhook
> 5. Eco al conserje del mensaje enviado

---

## CLASIFICACIÓN AUTOMÁTICA

AIC clasifica automáticamente por nombre de empresa. El conserje **NO**
necesita decir la categoría.

| Empresa | Categoría |
|---------|-----------|
| Rappi | food |
| Uber Eats | food |
| PedidosYa | food |
| DiDi Food | food |
| Cornershop | supermercado |
| Jumbo | supermercado |
| Lider | supermercado |
| Santa Isabel | supermercado |
| Amazon | encomienda |
| Falabella | encomienda |
| MercadoLibre | encomienda |
| Chilexpress | encomienda |
| Starken | encomienda |
| Correos de Chile | encomienda |
| Ripley | encomienda |

**Empresa desconocida**: AIC pregunta "¿Es comida, supermercado o encomienda?"

### Timers por categoría

| Categoría | Clock total | Urgencia | Recordatorio |
|-----------|-------------|----------|--------------|
| food | 30 min | CRÍTICA — se enfría | minuto 15 |
| supermercado | 4 horas | ALTA — perecibles | hora 1 |
| encomienda | 72 horas | NORMAL | hora 24, hora 48 |

---

## ESTADO 1: RECIBIDO

### Cuándo se activa
El conserje dice algo como:
- "Llegó paquete de Falabella para el 302"
- "Llegó comida de Rappi para el 508"
- "Hay un sobre para el depto 205"
- "Dejaron una caja grande para la señora del 401"

### Qué hacer

1. **Parsear los datos del mensaje del conserje:**
   - Departamento destino (OBLIGATORIO — si falta, preguntar)
   - Empresa (OBLIGATORIO — si falta, preguntar)
   - Categoría: auto-clasificar con la tabla de arriba
   - Tipo físico: comida | paquete | sobre | caja | refrigerado | frágil | grande

2. **Si falta el departamento:**
   - Preguntar al conserje: "¿Para qué departamento es?"
   - Si el conserje no sabe → flujo de paquete sin identificación (ver Edge Cases)

3. **Si falta la empresa:**
   - Preguntar: "¿De dónde viene? (empresa o remitente)"

4. **Registrar en `MEMORY.md` → `## Deliveries Activos`** con el siguiente
   formato (según ontología de AGENTS.md):
   ```
   ### [YYYY-MM-DD-NNNN] Depto [N] — [categoría]
   - **Empresa**: [nombre]
   - **Destinatario**: [nombre residente]
   - **Categoría**: [food|supermercado|encomienda]
   - **Estado**: RECIBIDO
   - **Recibido**: [HH:MM]
   - **Notificaciones**: 0
   - **ACK**: no
   ```

5. **Registrar en log diario (`memory/YYYY-MM-DD.md`):**
   ```
   ## [HH:MM] DELIVERY [CATEGORY] — Depto [N]
   - **Estado**: RECIBIDO
   - **Empresa**: [nombre]
   - **Categoría**: [food/supermercado/encomienda]
   - **Destinatario**: [nombre residente]
   - **Clock**: [30 min / 4 horas / 72 horas]
   ```

6. **Si es encomienda grande:** preguntar al conserje "¿Dónde lo dejaste,
   en conserjería o en la bodega?" y registrar la ubicación en `notes`.

7. **Responder al conserje con confirmación:**
   - Food: "✅ Recibido. Comida de [empresa] para el [depto]. Ya le aviso."
   - Supermercado: "✅ Registrado. Pedido de [empresa] para el [depto]. Tiene perecibles, prioridad alta. Ya notifico."
   - Encomienda: "✅ Registrado. [Tipo] de [empresa] para el [depto]. Queda en [conserjería/bodega]. Ya notifico."

8. **Pasar automáticamente a ESTADO 2: NOTIFICADO.**

> ⚠️ El eco de los mensajes al residente se hace en Estado 2 (ver sección
> TRANSPARENCIA).

---

## ESTADO 2: NOTIFICADO

### Cuándo se activa
Automáticamente después de registrar la recepción (Estado 1).

### Qué hacer

1. **Buscar al residente en `MEMORY.md` → `## Residentes` por número de depto.**
   - Leer: nombre, `trato` (tu/usted), `autorizados`

2. **Enviar notificación via Discord webhook al canal #general** (UNICO canal):
   ```
   POST https://discord.com/api/webhooks/1477409864639840347/gx1xKYG5tDXhAzIAD5I51TqV_YdAmYrB54LSWRKBD7hExTcwzN-3BbnFVEALmwtJrWyH
   Content-Type: application/json
   {"username": "AIC", "content": "[mensaje segun plantilla]"}
   ```
   - Mencionar al residente por nombre en el mensaje (NO hay DMs, todo va a #general)
   - Si el residente no está registrado en `## Residentes`: "No tengo registrado al residente del [depto]. ¿Me das su nombre para registrarlo?"

3. **Plantillas de notificación** (respetar el `trato` registrado):

   **Food:**
   "� [Nombre], llegó tu pedido de [empresa]! Baja pronto 🔥"

   **Supermercado:**
   "� [Nombre], llegó tu pedido de [empresa]. Tiene cosas refrigeradas, retíralo pronto."

   **Encomienda normal:**
   "📦 [Nombre], te llegó un paquete de [empresa]. Está en [conserjería/bodega] cuando quieras retirarlo."

   **Sin nombre (solo depto):**
   "📦 Hola, llegó un [tipo] de [empresa] para el departamento [N]. Está en conserjería."

   **Nombre en paquete ≠ residente registrado:**
   "📦 [Nombre registrado], llegó un [tipo] de [empresa] a nombre de [nombre en paquete] para su depto. Queda en conserjería."

4. **Registrar log de la notificación en `memory/YYYY-MM-DD.md`:**
   ```
   ## [HH:MM] NOTIFICACION — Depto [N]
   - **Estado**: NOTIFICADO
   - **Mensaje enviado a**: #general (Discord webhook)
   - **Categoría**: [food/supermercado/encomienda]
   ```

5. **Actualizar en `MEMORY.md` → `## Deliveries Activos`:**
   - Estado: NOTIFICADO
   - Notificaciones: 1
   - Hora última notificación: [HH:MM]

6. **Informar al conserje del mensaje enviado al residente (TRANSPARENCIA):**
   ```
   📢 Le envié esto al depto [N]:
   "[texto exacto del mensaje enviado al residente]"
   ```
   Nota: todas las notificaciones van a #general via webhook, no hay canales por residente.

7. **Si llegan múltiples deliveries para el mismo depto:**
   - Agrupar en una sola notificación al residente.
   - Registrar cada uno por separado en `## Deliveries Activos` y log.
   - Eco al conserje del mensaje completo.

8. **Pasar a ESTADO 3: STANDBY.**

---

## ESTADO 3: STANDBY

### Cuándo se activa
Después de enviar la notificación (Estado 2). El AIC monitorea y reintenta
sin necesidad de intervención del conserje.

> 📢 **TRANSPARENCIA**: Cada vez que el AIC envía un reintento al residente,
> debe informar al conserje qué le está diciendo (ver sección TRANSPARENCIA).

#### Si el residente RESPONDE (ACK):
Cuando el residente confirma ("ok", "ya voy", "gracias", "bajo en 5", etc.):

1. Registrar ACK en `MEMORY.md` → `## Deliveries Activos`:
   - ACK: sí
   - Hora ACK: [HH:MM]
   - Estado: STANDBY (esperando entrega física)

2. Responder al residente:
   - Food: "👍 Perfecto, acá lo esperamos."
   - Supermercado/Encomienda: "👍 Perfecto, queda esperándolo en [ubicación]."

3. Eco al conserje del ACK recibido:
   ```
   📢 El depto [N] confirmó que baja. Le respondí:
   "[mensaje de confirmación enviado]"
   ```

4. Iniciar timer de cierre proactivo:
   - Food: 30 min después del ACK
   - Supermercado: 1 hora después del ACK
   - Encomienda: 24 horas después del ACK

#### Si el residente NO RESPONDE (reintentos automáticos):

**Food (clock 30 min):**

| Tiempo | Acción con residente | Eco al conserje |
|--------|---------------------|-----------------|
| 15 min | "⚠️ [Nombre], tu pedido lleva 15 min, se va a enfriar!" | "📢 Reintento al depto [N]: [mensaje]" |
| 30 min | — (no más mensajes al residente) | "La comida del [depto] lleva 30 min sin retiro. ¿La descartamos?" → EXPIRADO |

**Supermercado (clock 4 horas):**

| Tiempo | Acción con residente | Eco al conserje |
|--------|---------------------|-----------------|
| 1 hora | "[Nombre], tu pedido de super lleva 1 hora, hay cosas que se pueden echar a perder." | "📢 Recordatorio al depto [N]: [mensaje]" |
| 4 horas | — (no más mensajes al residente) | "Pedido de super del [depto] lleva 4h. Alertar admin." → EXPIRADO |

**Encomienda (clock 72 horas):**

| Tiempo | Acción con residente | Eco al conserje |
|--------|---------------------|-----------------|
| 24 hs | "📦 Recordatorio: [Nombre], tienes un paquete esperando desde ayer." | "📢 Segundo recordatorio al depto [N]: [mensaje]" |
| 48 hs | "📦 Tu paquete de [empresa] lleva 2 días. ¿Puedes pasar a retirarlo?" | "📢 Tercer recordatorio al depto [N]: [mensaje]" |
| 72 hs | — (escalar, sin más mensajes al residente) | "La encomienda del [depto] lleva 3 días. Coordinar devolución o hablar con admin." → EXPIRADO |

**Actualizar en cada reintento:**
- Incrementar contador de notificaciones en `MEMORY.md`
- Registrar hora de cada reintento en log diario

---

## ESTADO 4: ENTREGADO / EXPIRADO

### ENTREGADO — cuándo se activa

| Situación | Trigger |
|-----------|---------|
| Conserje confirma entrega | "Se entregó", "ya lo retiró", "el 501 vino a buscar", "entregado" |
| Residente confirma | "Ya lo tengo", "ya lo recogí", "listo gracias" |
| Autorizado retira (registrado en `MEMORY.md`) | Conserje lo reporta, verificar nombre en `autorizados` |

#### Cierre exitoso:
1. Remover el delivery de `MEMORY.md` → `## Deliveries Activos`
2. Registrar en log diario:
   ```
   ## [HH:MM] ENTREGA — Depto [N]
   - **Estado**: ENTREGADO
   - **Retirado por**: [nombre]
   - **Tiempo total**: [minutos/horas desde recepción]
   - **Categoría**: [food/supermercado/encomienda]
   ```
3. Confirmar al conserje (si él inició el cierre):
   "✅ Entrega confirmada. [Tipo] de [empresa] para depto [N] — cerrado."
4. Confirmar al residente (si él inició el cierre):
   "✅ Perfecto, queda registrado. ¡Que lo disfrute!"

#### Cierre proactivo por el AIC (ACK dado, sin confirmación física):
1. Después del timer (30 min food / 1h super / 24h encomienda):
   - Al residente: "¿Pudiste recoger tu pedido de [empresa]?"
   - Si confirma → cierre exitoso
   - Si no responde en 15 min → preguntar al conserje: "¿Se entregó el pedido del [depto]?"
   - Si el conserje confirma → cierre exitoso
   - Si nadie responde → mantener en STANDBY, no cerrar

### EXPIRADO — cuándo se activa
Cuando el clock llega al límite sin retiro ni ACK.

| Categoría | Trigger |
|-----------|---------|
| food | 30 min sin retiro |
| supermercado | 4 horas sin retiro |
| encomienda | 72 horas sin retiro |

#### Cierre por EXPIRADO:
1. **Food/Supermercado:**
   - Remover de `MEMORY.md` → `## Deliveries Activos`
   - Registrar en log:
     ```
     ## [HH:MM] EXPIRADO — Depto [N]
     - **Empresa**: [nombre]
     - **Categoría**: [food/supermercado/encomienda]
     - **Tiempo sin retiro**: [minutos/horas]
     - **Acción**: Notificado a administración
     ```
   - Notificar al conserje: "La [food/super] de [empresa] para el [depto] expiró. ¿La descartas?"

2. **Encomienda:**
   - Mover a sección separada en `MEMORY.md`: "Encomiendas Pendientes (largo plazo)"
   - Registrar estado: `escalado_admin`
   - Notificar conserje + administración

#### Cierre por rechazo del residente:
1. Si dice "no es mío" o similar:
   - Responder: "Entendido. Voy a verificar con conserjería."
   - Notificar al conserje: "El residente del [depto] dice que el [tipo] de [empresa] no es suyo. ¿Puedes verificar los datos?"
   - Reclasificar como paquete sin destinatario confirmado → flujo de paquete sin identificación (ver Edge Cases)

---

## EDGE CASES

### Paquete sin identificación (sin departamento)
1. Avisar al conserje: "No tengo depto para ese paquete. ¿Puedes revisar si tiene algún dato?"
2. Si se identifica → seguir ciclo normal
3. Si no se identifica → registrar como `sin_destinatario` en `## Deliveries Activos`
4. Broadcast via Discord webhook a #general: "Llegó un paquete sin identificación clara. ¿Es suyo?"
5. Si alguien reclama → asignar y seguir ciclo normal
6. Si nadie reclama en 48h → notificar admin para devolución

### Residente pregunta "¿me llegó algo?"
1. Buscar en `MEMORY.md` → `## Deliveries Activos` por número de depto
2. Si hay pendientes: listar todos con detalle
3. Si no hay: "No tienes nada pendiente en conserjería en este momento."
4. También revisar "Encomiendas Pendientes (largo plazo)" si aplica

### Residente dice "viene alguien a buscar por mí"
1. Preguntar: "¿Quién viene? (nombre)"
2. Verificar si ya está en `MEMORY.md` → `## Residentes` → `autorizados`
3. Si no está → registrar autorización temporal en el delivery
4. Cuando la persona llegue y el conserje lo reporte: verificar nombre → entregar → cerrar ciclo
5. Responder al residente: "✅ Anotado. Cuando llegue [nombre], le entregamos."

### Encomienda en bodega
- Si el conserje menciona "bodega", "abajo", "en el depósito":
  - Registrar `notes: "ubicación: bodega"`
  - Incluir en todas las notificaciones: "Está en la bodega, no en conserjería"
  - Recordar ubicación cuando el residente vaya a buscar

### Múltiples deliveries simultáneos para el mismo depto
- Agrupar en una sola notificación al residente
- Registrar cada uno por separado en `## Deliveries Activos` y log
- En el cierre, verificar que se entregaron TODOS antes de cerrar cada uno

---

## TRANSPARENCIA CON EL CONSERJE

**Principio fundamental** (AGENTS.md): El conserje no tiene visibilidad directa
del chat entre el AIC y los residentes. Por lo tanto, **cada vez que el AIC
envía un mensaje a un residente, debe informarle al conserje** qué le está
diciendo.

### Cuándo aplicar el eco

| Evento | El AIC informa al conserje |
|--------|---------------------------|
| Notificación inicial al residente (Estado 2) | Sí, siempre |
| Confirmación de ACK al residente | Sí |
| Cualquier reintento/recordatorio al residente (Estado 3) | Sí |
| Mensajes de cierre de ciclo al residente (Estado 4) | Sí |
| Preguntas de verificación al residente ("¿pudiste retirarlo?") | Sí |

### Formato del eco

```
📢 Le envié esto al depto [N]:
"[texto exacto del mensaje enviado al residente]"
```

**Ejemplo completo:**
> Conserje: "Llegó uber para el 502"
> AIC → Discord #general (webhook): "🍕 María, llegó tu pedido de Uber Eats! Baja pronto 🔥 — AIC"
> AIC → conserje: "📢 Le envié esto al #general para el depto 502:
> '🍕 María, llegó tu pedido de Uber Eats! Baja pronto 🔥 — AIC'"

> Residente 502 responde en Discord: "ya bajo en 10"
> AIC → Discord #general (webhook): "👍 Perfecto, acá lo esperamos."
> AIC → conserje: "📢 El depto 502 confirmó que baja en 10 min. Le respondí:
> '👍 Perfecto, acá lo esperamos.'"

### Excepciones (no eco)
- Si el conserje **es** el interlocutor del turno (mensajes dirigidos al conserje mismo).
- Si el webhook de Discord falla y el AIC ya avisó que no pudo notificar (con nota `webhook: pendiente`).

---

## REGLAS GENERALES

1. **El conserje reporta, el AIC se encarga.** No molestar al conserje con
   seguimientos que el AIC puede manejar solo.
2. **Siempre cerrar el ciclo.** Todo delivery debe terminar en ENTREGADO o EXPIRADO.
3. **Nunca compartir info entre departamentos.** Si el 301 pregunta por el
   302, responder: "Esa información es confidencial."
4. **Priorizar por urgencia.** Food > Supermercado > Encomienda.
5. **Ante la duda, preguntar al conserje.** Si algo no cuadra, consultar
   antes de asumir.
6. **Registrar todo.** Cada acción queda en log diario (`memory/YYYY-MM-DD.md`)
   y en `MEMORY.md` para trazabilidad.
7. **Transparencia total con el conserje.** El conserje siempre sabe qué le
   dijo el AIC al residente. Sin excepciones.
8. **Respetar el trato.** Usar el campo `trato` de `## Residentes` (tu/usted)
   en todos los mensajes al residente.
9. **Siempre llamar al webhook.** Cada RECIBIDO y ENTREGADO pasa por
   `registrarPaquete()` o `marcarEntregado()` antes de actualizar `MEMORY.md`.
   Si el webhook falla, registrar igual en memoria con nota `webhook: pendiente`.
10. **Solo Discord.** NUNCA intentar Telegram, WhatsApp, SMS ni otro canal.
    No existe @Kyonclawbot. No hay chat_id de Telegram. El webhook de Discord
    al canal #general es el UNICO mecanismo de notificacion a residentes.
