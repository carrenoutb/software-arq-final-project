# Fase 2 — ADR: Decisión de diseño

*Proyecto de ejemplo · Feature dummy: **Reservas de libros** en `biblioteca-api`*

> Este es el **resultado** de la Fase 2: un ADR que documenta la decisión comparando alternativas. Muestra cómo justificar una decisión con trade-offs reales (el 50% de la nota de tu proyecto). El ADR se escribe **con las opciones a la vista**, no después.

---

## ADR-001 · Modelado de la reserva de libros sin copias disponibles

### Estado

Aceptado — 2026-11-08

### Contexto

La biblioteca necesita que los estudiantes **reserven libros agotados** y esperen en fila: cuando una copia se devuelve, la reserva más antigua se atiende. Los requisitos funcionales completos están en la Fase 1.

Al analizar el código real se encontró:

- `loans` está **100% acoplado a libros**: `loans.book_id` (FK NOT NULL), `LoanRequest.BookID`, JOINs a `books`, y `status` ENUM `active`/`returned`.
- No existe ninguna noción de "fila/cola" en el esquema.
- Las reglas de negocio viven en `LoanService` (validaciones antes de persistir), patrón correcto que debe preservarse.

Se consideraron **tres opciones** (análisis completo en la Fase 1):

- **A. Tabla nueva `reservations` + módulo por capas paralelo a `loans`** (own repository/service/handler).
- **B. Reusar `loans` agregando `status='reserved'`** (misma tabla, nuevo valor del ENUM).
- **C. Contador `waitlist_count` en `books`** sin registrar a quién espera.

### Decisión

Se implementa la **Opción A**: una tabla `reservations` nueva, con su propio módulo `handler → service → repository`, siguiendo exactamente el patrón del módulo de préstamos. La única interacción con el flujo existente es una llamada **al final de `LoanService.Return()`**: tras marcar el libro como devuelto, se "despierta" la reserva activa más antigua de ese libro (estado `waiting → fulfilled`).

**¿Por qué sobre la B?** B parece más barato hoy (una tabla menos), pero acopla dos conceptos del dominio distintos — *"un préstamo en curso"* y *"una espera por disponibilidad"* — en la misma entidad. Consecuencias de B a mediano plazo: las consultas de préstamos tendrían que filtrar un tercer estado en cada query (incluidas `FindByUser` y el JOIN administrativo); el `due_at` quedaría vacío para reservas (rompiendo el invariante de préstamo activo); y cada nueva feature del préstamo arrastraría el concepto de reserva. El costo de B se vuelve alto y difuso.

**¿Por qué sobre la C?** C no cumple el requisito de la fila ordenada: sin registrar *quién* espera no se puede atender a la primera reserva ni notificar. Inadmisible ya en requisitos, no solo por diseño.

### Consecuencias

**✅ Positivas**

- `loans` y `books` quedan intactos: el flujo de préstamo probado (semana 14, taller `/ping`) no se arriesga — cumple la regla de oro del enunciado *"nada existente se rompe"*.
- El dominio de reserva queda autónomo y creíble: su `status` propio (`waiting` / `fulfilled` / `canceled`) no contamina el del préstamo.
- El paso de datos es directo: `reservations.user_id / book_id / reserved_at / status` — sin campos condicionales ni "nulos que significan algo".
- Mantenibilidad: si mañana la biblioteca reserva salas de estudio, el patrón se replica sin tocar `loans`.

**❌ Negativas**

- Más archivos que la opción B: un módulo completo (model, repository, service, handler, rutas, migración `002_*.sql`).
- **Hooks de integración**: `LoanService.Return()` debe invocar a la nueva lógica → la reserva introduce un acoplamiento en la dirección `loan → reservation` (la fila avanza porque alguien devolvió). Se mitiga delegando en `ReservationService.AdvanceQueue(bookID)`, de modo que `loans` no conoce el esquema de reservas, solo la operación semántica.
- La migración `002` debe ser idempotente y convivir con `001` (mismo mecanismo `docker-entrypoint-initdb.d`).

### Criterios de comparación usados (para transparencia)

Extensibilidad futura (¿se replica para otro recurso?) **>** Riesgo sobre lo existente **>** Simplicidad de implementación. La opción A gana en los dos primeros, pierde levemente en el tercero — trade-off aceptado y documentado aquí porque la regla del proyecto es "no romper nada".

---

## Decisión de negocio adicional (mini-ADR resumido)

**Pregunta:** ¿qué pasa con una reserva `waiting` si el libro deja de existir (DELETE admin)?

**Decisión:** `DELETE /api/books/:id` existente no elimina reservas; se marca el tema como pendiente de política (fuera de alcance del enunciado dummy). La reserva queda `waiting` indefinida — aceptado como limitación del ejemplo, se documenta en el código. En tu proyecto, este tipo de hueco de borde es EXACTAMENTE lo que conviene detectar y resolver (o abrir como pregunta en la defensa).

---

> *Nota: un ADR bien hecho deja ver (1) que hubo alternativas reales, (2) qué criterios pesaron, y (3) qué se sacrificó. Si tu ADR no tiene ninguna «❌», vuelve a la Fase 1: no buscaste lo suficiente.*