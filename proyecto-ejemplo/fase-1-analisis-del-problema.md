# Fase 1 — Análisis del problema

*Proyecto de ejemplo · Feature dummy: **Reservas de libros** en `biblioteca-api`*

> Este documento es la **Fase 1 resuelta** de un proyecto final de ejemplo. Muestra *cómo se ve* un análisis bien hecho: se lee el código real, se encuentra el acoplamiento y se generan opciones con su tabla de trade-offs. Tu proyecto es distinto, pero el **proceso** es el mismo.

---

## 1. El problema (enunciado dummy)

> La biblioteca quiere que los estudiantes puedan **reservar un libro que no tiene copias disponibles** en ese momento. Cuando una copia se devuelve, la primera reserva pendiente del libro ('la fila') debe ser notificada/atendida. Un estudiante no debe poder reservar dos veces el mismo libro mientras su reserva está activa, y puede cancelarla si cambió de idea.

### Requisitos funcionales que se derivan

1. Crear una reserva: `POST /api/reservations` (autenticado) — solo si el libro **no tiene copias disponibles**.
2. Ver mis reservas: `GET /api/reservations/me` (autenticado).
3. Cancelar una reserva propia: `DELETE /api/reservations/:id` (autenticado, dueño).
4. Ver la fila completa de reservas: `GET /api/reservations` (solo admin).
5. Avanzar la fila: cuando un libro se devuelve, la reserva **más antigua activa** pasa a estado `fulfilled`.

---

## 2. Lectura del código real (verificada contra el repo)

Antes de proponer nada, recorrí los archivos que participan en el préstamo actual:

| Archivo (ruta en el repo) | Qué aporta al análisis |
|---|---|
| `migrations/001_init.sql` | Esquema base: `users`, `books`, `loans`. `loans.book_id` es FK **NOT NULL** hacia `books`; `loans.status` es ENUM `active`/`returned` |
| `internal/models/loan.go` | `Loan{ID, UserID, BookID, LoanedAt, DueAt, ReturnedAt, Status}` y `LoanRequest{BookID}` — el préstamo **asume** que se presta un libro |
| `internal/service/loan_service.go` | `Borrow()`: valida `available_copies > 0`, valida préstamo duplicado, crea el préstamo con vencimiento +14 días, decrementa `available_copies` |
| `internal/repository/loan_repository.go` | SQL real: `INSERT INTO loans…`, `COUNT(*) … status='active'`, `FindByUser` con JOIN a `books` |
| `internal/handler/routes.go` | Grupos: público `/api/auth/*`; protegido (JWT) `loans`, `books`; admin `POST/PUT/DELETE /api/books`, `GET /api/loans` |

**Hallazgos clave del acoplamiento:**

1. **El préstamo es 100% libro-centrico**: `LoanRequest` solo lleva `book_id`; los JOINs van a `books`; la disponibilidad vive en `books.available_copies`.
2. **Las validaciones de negocio viven en `LoanService`**, no en SQL ni en handlers (patrón correcto a imitar).
3. **`loans.status` es un ENUM de 2 valores** (`active`/`returned`) — agregar un tercer valor (`reserved`) aquí **parece** barato pero mezclaría dos conceptos: "préstamo en curso" vs "espera por disponibilidad".
4. **No existe nada parecido a una fila/cola** en el esquema: no hay orden ni "posiciones de espera".

---

## 3. Opciones de diseño identificadas

### Opción A — Tabla nueva `reservations` + módulo por capas (paralelo a `loans`)

Nueva entidad `reservations(user_id, book_id, status, reserved_at)` con su propio `ReservationService`, `ReservationRepository`, `ReservationHandler` y rutas propias. El préstamo existente no se toca; `loan_service.Return()` se *extiende* al final del flujo actual (después de marcar devuelto) para "despertar" la primera reserva activa.

| Criterio | Valor |
|---|---|
| Qué modifica de lo existente | Casi nada: solo una llamada extra al final de `Return()` (para avanzar la fila) |
| Qué duplica | Estructura del módulo (archivos), siguiendo el patrón ya probado de `loans` |
| Costo de agregar "el próximo recurso" (salas, laptops) | Indiferente: las reservas son de un libro concreto, no afecta el préstamo de otros recursos |
| Riesgo | Bajo: `loans` y `books` quedan intactos |
| Complejidad | Media: módulo nuevo pero con el molde ya visto |

### Opción B — Reusar `loans` agregando `status='reserved'`

Ampliar el ENUM de `loans.status` a `'reserved'` y usar la misma tabla para reservas en espera, con `due_at` vacío mientras espera.

| Criterio | Valor |
|---|---|
| Qué modifica | La migración 001 (ENUM), `LoanRequest`, todas las queries de `loans` que filtran por estado |
| Qué duplica | Nada (una sola tabla) |
| Costo futuro | Alto: mezclar "préstamo" y "espera" corrompe las estadísticas, `FindByUser` y el JOIN; cada feature futura de préstamo arrastra el concepto de reserva |
| Riesgo | Alto: tocar el corazón del flujo funcionando |
| Complejidad | Baja para escribir, alta para mantener |

### Opción C — Contador de "espera" en `books` + reservas en memoria

Agregar `books.waitlist_count` y no persistir reservas individuales (solo llevar el conteo).

| Criterio | Valor |
|---|---|
| Qué modifica | Tabla `books` + el modelo `Book` |
| Qué duplica | El conteo es derivado, sin tabla de reservas |
| Costo futuro | Inaceptable: no hay registro de quién espera → no se puede "despertar" a nadie; no cumple el requisito de fila ordenada |
| Riesgo | Bajo (no toca loans) pero no cumple el problema |
| Complejidad | Baja |

> Alternativa desechada de plano: implementar la fila "en memoria" del proceso Go (un slice). Se pierde al reiniciar el contenedor y no es consistente entre instancias; contradice la persistencia de MySQL que ya usa el proyecto.

---

## 4. Tabla de trade-offs (resumen)

| Criterio | **A · Tabla nueva** | B · Reusar loans | C · Contador en books |
|---|---|---|---|
| Cumple la fila ordenada | ✅ | ✅ | ❌ |
| No rompe el préstamo actual | ✅ (solo extiende `Return`) | ❌ (ENUM + queries) | ✅ |
| Esfuerzo de implementación | Medio | Bajo (pero frágil) | Bajo |
| Mantenibilidad futura | Alta | Baja | Baja |
| Riesgo | Bajo | Alto | Bajo (pero incompleto) |

---

## 5. Conclusión del análisis

Las opciones **A** y **B** son genuinamente viables; **C** no cumple el requisito mínimo. La decisión entre A y B es un *trade-off* clásico: **B parece más barato hoy (cero tablas nuevas) pero acopla dos conceptos del dominio con costos futuros altos**; A paga un poco más de estructura ahora y deja el préstamo intacto. La decisión final se documenta en la **Fase 2 (ADR)**.

> *Nota didáctica: en tu proyecto, las opciones serán otras y la decisión será tuya. Lo que aquí ves es el proceso: leer → acotar → comparar con criterios honestos → dejar argumentos listos para el ADR.*