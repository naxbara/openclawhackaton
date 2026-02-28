# Checklist Proactivo — AIC
# Ciclo: cada 5 minutos
# Canal: SOLO Discord webhook al #general (NUNCA Telegram/WhatsApp/SMS)

## 1. Deliveries de COMIDA en STANDBY (CRITICO)
- Revisar "Deliveries Activos" en MEMORY.md
- Si algun delivery FOOD lleva > 15 min sin retiro:
  Enviar recordatorio urgente via Discord webhook al canal #general
- Si lleva > 30 min:
  Cambiar estado a EXPIRADO
  Notificar a administracion via Discord webhook

## 2. Deliveries de SUPERMERCADO en STANDBY (ALTO)
- Si algun delivery SUPERMERCADO lleva > 1 hora sin retiro:
  Enviar recordatorio via Discord webhook (mencionar perecibles)
- Si lleva > 4 horas:
  Cambiar estado a EXPIRADO
  Notificar a administracion via Discord webhook

## 3. Deliveries de ENCOMIENDA en STANDBY (NORMAL)
- Si algun delivery ENCOMIENDA lleva > 24 horas sin retiro:
  Enviar recordatorio via Discord webhook
- Si lleva > 72 horas:
  Cambiar estado a EXPIRADO
  Notificar a administracion via Discord webhook

## 4. Resumen (si se pide)
- Total deliveries del dia por categoria
- Entregados vs expirados
- Tiempo promedio de retiro por categoria
