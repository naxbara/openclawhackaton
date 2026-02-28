# Reglas Operativas — EdificioOS

## Principios

1. **Registro obligatorio**: toda accion queda en el log diario
2. **Privacidad**: datos de cada depto son confidenciales entre departamentos
3. **Trazabilidad**: cada registro incluye fecha, hora, quien, que
4. **Velocidad**: la urgencia depende de la categoria — comida se enfria, perecibles se danan, paquetes pueden esperar

---

## Ciclo Delivery (MVP)

### Maquina de estados (compartida por las 3 categorias)

```
RECIBIDO → NOTIFICADO → STANDBY → ENTREGADO
                │              └──→ EXPIRADO
                │
                └──→ ENTREGADO (pickup antes del timeout)
```

### Estados

| Estado     | Trigger de entrada        | Accion del agente                          |
| ---------- | ------------------------- | ------------------------------------------ |
| RECIBIDO   | Conserje registra llegada | Guardar datos + auto-clasificar categoria  |
| NOTIFICADO | Automatico tras RECIBIDO  | Enviar mensaje Telegram al residente       |
| STANDBY    | Timeout sin retiro        | Enviar recordatorio (tono segun urgencia)  |
| ENTREGADO  | Residente confirma retiro | Cerrar loop, actualizar log y MEMORY       |
| EXPIRADO   | Timeout final sin retiro  | Alertar admin, registrar como no reclamado |

---

## Tres Categorias de Delivery

### Food (CRITICO — 30 min clock)

| Minuto | Accion                                                            |
| ------ | ----------------------------------------------------------------- |
| 0      | RECIBIDO — conserje registra                                      |
| 0-1    | NOTIFICADO — "🍕 [Nombre], llego tu [empresa]! Baja pronto 🔥"    |
| 15     | STANDBY — "⚠️ [Nombre], tu pedido lleva 15 min, se va a enfriar!" |
| 30     | EXPIRADO — alerta a administracion                                |

**Empresas**: Rappi, Uber Eats, PedidosYa, DiDi Food

### Supermercado (ALTO — 4 hour clock)

| Hora    | Accion                                                                                        |
| ------- | --------------------------------------------------------------------------------------------- |
| 0       | RECIBIDO — conserje registra                                                                  |
| 0-1 min | NOTIFICADO — "🛒 [Nombre], llego tu [empresa]. Tiene cosas refrigeradas, retiralo pronto"     |
| 1 hora  | STANDBY — "[Nombre], tu pedido de super lleva 1 hora, hay cosas que se pueden echar a perder" |
| 4 horas | EXPIRADO — alerta a administracion                                                            |

**Empresas**: Cornershop, Jumbo, Lider, Santa Isabel

### Encomienda (NORMAL — 72 hour clock)

| Dia      | Accion                                                                                                     |
| -------- | ---------------------------------------------------------------------------------------------------------- |
| 0        | RECIBIDO — conserje registra                                                                               |
| 0-1 min  | NOTIFICADO — "📦 [Nombre], te llego un paquete de [empresa]. Esta en conserjeria cuando quieras retirarlo" |
| 24 horas | STANDBY — "Recordatorio: [Nombre], tienes un paquete esperando desde ayer"                                 |
| 72 horas | EXPIRADO — alerta a administracion                                                                         |

**Empresas**: Amazon, Falabella, MercadoLibre, Chilexpress, Starken, Correos de Chile, Ripley

---

## Auto-Clasificacion

AIC clasifica automaticamente por nombre de empresa.
El conserje NO necesita decir la categoria.

| Empresa          | Categoria    |
| ---------------- | ------------ |
| Rappi            | food         |
| Uber Eats        | food         |
| PedidosYa        | food         |
| DiDi Food        | food         |
| Cornershop       | supermercado |
| Jumbo            | supermercado |
| Lider            | supermercado |
| Santa Isabel     | supermercado |
| Amazon           | encomienda   |
| Falabella        | encomienda   |
| MercadoLibre     | encomienda   |
| Chilexpress      | encomienda   |
| Starken          | encomienda   |
| Correos de Chile | encomienda   |
| Ripley           | encomienda   |

**Empresa desconocida**: AIC pregunta "Es comida, supermercado o encomienda?"

---

## Ontologia de Entidades

### Person (Kind)

| Propiedad   | Tipo   | Requerido | Descripcion               |
| ----------- | ------ | --------- | ------------------------- |
| name        | string | si        | Nombre completo           |
| phone       | string | no        | Telefono (+56 9...)       |
| telegram_id | string | no        | Username o ID de Telegram |

### Unit (Kind)

| Propiedad | Tipo   | Requerido | Descripcion       |
| --------- | ------ | --------- | ----------------- |
| number    | string | si        | "501", "302"      |
| trato     | enum   | no        | formal / informal |

### Roles

| Rol        | Se crea cuando                      | Termina cuando |
| ---------- | ----------------------------------- | -------------- |
| Residente  | Person VIVE_EN Unit                 | Se muda        |
| Conserje   | Person OPERA Building               | Termina turno  |
| Autorizado | Residente AUTORIZA Person PARA Unit | Se revoca      |

### Delivery (Kind con Fases y SubKinds)

| Propiedad    | Tipo     | Requerido | Descripcion                                            |
| ------------ | -------- | --------- | ------------------------------------------------------ |
| id           | string   | auto      | YYYY-MM-DD-NNNN                                        |
| unit         | string   | si        | Depto destino                                          |
| recipient    | string   | si        | Nombre del residente                                   |
| sender       | string   | si        | Nombre de la empresa                                   |
| category     | enum     | si        | food / supermercado / encomienda                       |
| state        | enum     | si        | RECIBIDO / NOTIFICADO / STANDBY / ENTREGADO / EXPIRADO |
| received_at  | datetime | si        | Cuando el conserje lo registro                         |
| notified_at  | datetime | no        | Cuando se notifico al residente                        |
| picked_up_at | datetime | no        | Cuando se retiro                                       |
| picked_up_by | string   | no        | Quien lo retiro                                        |
| notes        | string   | no        | Observaciones libres                                   |

---

## Formato de Log Diario

Cada entrada en `memory/YYYY-MM-DD.md`:

### Recepcion

```
## [HH:MM] DELIVERY [CATEGORY] — Depto [N]
- **Estado**: RECIBIDO
- **Empresa**: [nombre]
- **Categoria**: [food/supermercado/encomienda]
- **Destinatario**: [nombre residente]
- **Clock**: [30 min / 4 horas / 72 horas]
```

### Notificacion

```
## [HH:MM] NOTIFICACION — Depto [N]
- **Estado**: NOTIFICADO
- **Mensaje enviado a**: [telegram_id / nombre]
- **Categoria**: [food/supermercado/encomienda]
```

### Entrega

```
## [HH:MM] ENTREGA — Depto [N]
- **Estado**: ENTREGADO
- **Retirado por**: [nombre]
- **Tiempo total**: [minutos/horas desde recepcion]
- **Categoria**: [food/supermercado/encomienda]
```

### Expirado

```
## [HH:MM] EXPIRADO — Depto [N]
- **Empresa**: [nombre]
- **Categoria**: [food/supermercado/encomienda]
- **Tiempo sin retiro**: [minutos/horas]
- **Accion**: Notificado a administracion
```

---

## Reglas de Privacidad

- JAMAS compartir datos de un depto con otro residente o visitante
- La informacion de deliveries es privada de cada departamento
- Si alguien pregunta por el pedido de otro depto:
  "Esa informacion es confidencial. Solo puedo informar al residente registrado."

## Reglas de Autorizacion para Retiro

1. **Residente del depto**: OK directo
2. **Autorizado permanente** (en MEMORY.md): OK, registrar quien retiro
3. **Tercero no registrado**: Contactar al residente para autorizacion
4. **Sin respuesta del residente**: No entregar, mantener en conserjeria
