# Ecosistema de Automatización IA: Triage de Tickets de Soporte

**Caso de uso:** un cliente (ficticio) envía una consulta de soporte. El sistema la registra, la clasifica con Claude (categoría, urgencia, resumen y borrador de respuesta), la envía a un humano para aprobación por Gmail (correo con botones Aprobar/Rechazar) y, solo si se aprueba, responde al cliente por Gmail.

**Stack:** n8n (orquestador) · Airtable (base de datos) · Claude vía API de Anthropic (motor IA) · Gmail (aprobación humana y salida al cliente).

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `diagrama-arquitectura.pdf` | Diagrama de arquitectura |
| `flujo-n8n.json` | Flujo exportable/importable en n8n |
| `README.md` | Este documento (base de datos, decisiones, pruebas, enlaces) |
| `evidencias/` | Capturas de pantalla del flujo y de las pruebas |

**Link a la base de datos Airtable (modo lectura):** https://airtable.com/appv1Cw8iWwCnxCzE/shrMupyPU8Qet3XNf

## 1. Base de datos (Airtable) – "el cerebro"

Tres tablas relacionadas, sin datos aislados:

**Clientes**: `Nombre` (texto), `Email` (email), `Tickets` (link inverso a Tickets).

**Tickets**: `Asunto` (texto), `Mensaje` (texto largo), `Estado` (selección única), `Categoria` (selección única: Facturacion, Tecnico, Cuenta, Otro), `Urgencia` (número 1-5), `Resumen IA`, `Respuesta sugerida` (texto largo), `Cliente` (**link → Clientes**), `Gmail Thread ID`, `Gmail Message ID`.
- Valores de `Estado`: `Pendiente` → `Procesado por IA` → `Aprobado por Humano` (o `Rechazado` / `Error`).

**Errores**: `Origen` (selección única: Validacion, IA, Gmail), `Detalle` (texto largo), `Ticket` (**link → Tickets**), `Fecha` (Created time).

Fuente única de datos (el webhook), por lo que la sincronización entre fuentes la gestiona el propio flujo (upsert de Cliente por `Email`, luego creación del Ticket vinculado).

## 2. Flujo (n8n) – "el corazón"

1. **Webhook Ticket** (POST `/ticket-soporte`): trigger específico; el flujo solo corre cuando llega un ticket.
2. **Validar Datos**: email presente, con `@`, distinto de la casilla propia (anti-bucle) y mensaje de al menos 10 caracteres (comparación número vs número). Si falla → registra en `Errores` y responde HTTP 400.
3. **Upsert Cliente → Crear Ticket** (`Estado = Pendiente`, relación al cliente) → responde HTTP 202.
4. **Claude Triage** (nodo Anthropic, modelo `claude-haiku-5-5`): prompt dinámico con nombre, email, asunto y mensaje del ticket; `maxTokens = 1500`; 2 reintentos. La respuesta se mapea buscando el bloque `type = text` dentro de `content[]`.
5. **Parsear Respuesta IA**: extrae el JSON y valida los tipos (`urgencia` se convierte a número).
6. **Manejo de errores:** las salidas de error de Claude y del parseo van a **Registrar Error IA** (tabla `Errores`) y el ticket pasa a `Estado = Error`. Lo mismo para fallos de Gmail.
7. **Ticket Procesado IA**: actualiza el ticket (`Estado = Procesado por IA`).
8. **Human-in-the-loop:** **Gmail Aprobacion** (Send and Wait) envía al revisor un correo con el resumen y el borrador (marcado `[URGENTE]` si urgencia ≥ 4) y pausa el flujo hasta que pulsa *Aprobar y enviar* o *Rechazar*.
9. **Aprobado**: Gmail responde dentro del hilo original usando `Message ID` (si no hay `message_id`, envía un correo nuevo), y `Estado = Aprobado por Humano`.
10. **Rechazado**: `Estado = Rechazado` y no se contacta al cliente.

## 3. Pruebas (5 ejecuciones reales en n8n)

| # | Escenario | Entrada | Resultado | Evidencia |
|---|---|---|---|---|
| 1 | **Camino infeliz:** datos incompletos | Mensaje con menos de 10 caracteres / sin email | HTTP 400 y fila en `Errores` (Origen = Validacion) | `evidencias/test1.png` |
| 2 | **Camino infeliz:** falla la API de IA | Parámetro inválido hacia Claude (error 400 de la API) | Fila en `Errores` (Origen = IA) y ticket `Estado = Error` | `evidencias/test2.png` |
| 3 | **Camino infeliz:** respuesta de IA no parseable | Respuesta truncada sin JSON | Fila en `Errores` (Origen = IA) y ticket `Estado = Error` | `evidencias/test3.png` |
| 4 | Camino feliz hasta IA | Ticket válido de acceso a cuenta | Categoría `Cuenta`, urgencia 4, resumen y borrador guardados; `Estado = Procesado por IA` | `evidencias/test4.png` |
| 5 | Camino feliz completo | Mismo ticket, se pulsa *Aprobar y enviar* en el correo | Correo enviado al cliente y `Estado = Aprobado por Humano` | `evidencias/test5.png` |

Las pruebas 2 y 3 corresponden a fallos reales encontrados y corregidos durante el desarrollo (se quitó un parámetro no soportado y se corrigió el parseo del bloque de texto), lo que demuestra que el manejo de errores funciona.

## 4. Check de seguridad

1. **¿Filtro contra bucles infinitos?** Sí: el flujo descarta tickets cuyo remitente sea la casilla de soporte. Además el trigger es un webhook y la respuesta sale por Gmail, que no dispara el flujo.
2. **¿Tipos de datos correctos en los filtros?** Sí: texto contra texto (email) y número contra número (`length >= 10`, `urgencia >= 4`); con `typeValidation: strict`.
3. **¿Prompt dinámico?** Sí: se arma con las variables del ticket (nombre, email, asunto, mensaje) y no hay texto fijo del cliente.

Las API keys y credenciales no están en el repositorio: el JSON solo contiene referencias a credenciales de n8n.
