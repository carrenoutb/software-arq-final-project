# Fase 4 — Verificación

*Proyecto de ejemplo · Feature dummy: **Reservas de libros** en `biblioteca-api`*

> **Resultado** de la Fase 4: cómo se demuestra que la feature funciona **y** que nada existente se rompió. Regla de oro del proyecto: *"todo debe seguir funcionando perfectamente"*. Las respuestas esperadas usan los datos reales del seed (`001_init.sql`) y el tutorial REST Client del README.

---

## 0. Preparación

```bash
docker compose up --build
```

Esperado: los 3 contenedores `Up`, MySQL `(healthy)` (ver `docker compose ps`).

**Fondo de datos usado en esta verificación** (del seed de `001_init.sql`):

| Libro | `id` | Copias disponibles |
|---|---|---|
| Cien años de soledad | 1 | 3 |
| Clean Code | 2 | 2 |

Para probar la **reserva** (que exige libro sin copias), ajustamos un libro del seed al estado "agotado":

```sql
UPDATE books SET available_copies = 0 WHERE id = 2;
```

> 📸 **Evidencia 0:** salida de `docker compose ps` + resultado del `UPDATE` (Adminer en `http://localhost:8081`, o `docker compose exec mysql mysql -ubiblioteca_user -pbiblioteca_pass biblioteca`).

---

## 1. Lo existente sigue funcionando (regla de oro)

| # | Petición | Respuesta esperada | ✅ |
|---|---|---|---|
| 1.1 | `GET /health` | `{"status":"ok"}` | |
| 1.2 | `GET /ping` | `{"message":"pong"}` | |
| 1.3 | `POST /api/auth/register` (usuario nuevo: `estudiante-reserva@example.com` / `pass@123`) | `201` con `token` y `user` | |
| 1.4 | `POST /api/auth/login` con ese usuario | `200` con `token` | |
| 1.5 | `GET /api/books` (con token) | `200` con el catálogo del seed | |
| 1.6 | `POST /api/loans` prestando el libro 1 (Cien años de soledad) | `201` préstamo activo | |
| 1.7 | `GET /api/loans/me` | `200` muestra el préstamo del paso 1.6 | |
| 1.8 | `PUT /api/loans/:id/return` del préstamo del paso 1.6 | `200` "libro devuelto correctamente" | |

> 📸 **Evidencia 1:** salida real de cada petición (puede ser el panel de REST Client de VS Code, con request arriba y response abajo).

**¿Qué pasó?** Todo el flujo previo (auth, libros, préstamos) responde igual que antes de la feature. Si alguno de estos falla, la feature **rompió algo** y hay que arreglarlo antes de seguir.

---

## 2. La feature nueva — Reservas

### 2.1 Reservar un libro agotado ✅ (camino feliz)

```http
### Reservar libro agotado (Clean Code, id 2)
POST http://localhost:8080/api/reservations
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "book_id": 2
}
```

**Respuesta esperada** (`201`):

```json
{
  "id": 1,
  "user_id": 1,
  "book_id": 2,
  "status": "waiting",
  "reserved_at": "2026-11-08T15:30:00Z"
}
```

**¿Qué pasó?** El `available_copies` del libro 2 es `0` → el servicio permitió la reserva y la guardó en `waiting`. El `status` ya dice `waiting`: está en la fila.

### 2.2 Intentar reservar un libro CON copias ❌ (validación 1)

```http
### Reservar libro disponible (Cien años de soledad, id 1)
POST http://localhost:8080/api/reservations
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "book_id": 1
}
```

**Respuesta esperada** (`409 Conflict`):

```json
{"error":"el libro aún tiene copias disponibles, no se puede reservar"}
```

**¿Qué pasó?** La regla de negocio del servicio (`ErrBookHasCopies`) protegió el caso: no se puede saltar la fila si hay copia física disponible.

### 2.3 Reserva duplicada ❌ (validación 2)

```http
### Intentar reservar el mismo libro agotado otra vez
POST http://localhost:8080/api/reservations
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "book_id": 2
}
```

**Respuesta esperada** (`409 Conflict`):

```json
{"error":"ya tienes una reserva activa de este libro"}
```

**¿Qué pasó?** `HasWaiting` detectó la reserva activa del paso 2.1 y la rechazó — un estudiante no puede acaparar la fila con dos reservas del mismo libro.

### 2.4 Mis reservas ✅

```http
### Mis reservas
GET http://localhost:8080/api/reservations/me
Authorization: Bearer {{token}}
```

**Respuesta esperada** (`200`):

```json
[
  {
    "id": 1,
    "user_id": 1,
    "book_id": 2,
    "status": "waiting",
    "reserved_at": "2026-11-08T15:30:00Z",
    "book_title": "Clean Code",
    "book_author": "Robert C. Martin"
  }
]
```

**¿Qué pasó?** El `JOIN` con `books` trajo el título/autor para que el frontend no haga una segunda petición (patrón `LoanDetail`).

### 2.5 Sin token ❌ (seguridad)

```http
### Reservas sin token (debe fallar)
GET http://localhost:8080/api/reservations/me
```

**Respuesta esperada** (`401`):

```json
{"error":"token no proporcionado"}
```

**¿Qué pasó?** La ruta está en el grupo protegido por `middleware.AuthRequired` — nadie sin sesión ve ni crea reservas.

### 2.6 Fila completa (solo admin) ✅/❌

```http
### Todas las reservas — con rol student (debe fallar)
GET http://localhost:8080/api/reservations

### Todas las reservas — con rol admin (debe funcionar)
GET http://localhost:8080/api/reservations
Authorization: Bearer {{token_admin}}
```

**Respuestas esperadas:** primero `403` (`{"error":"se requiere rol de administrador"}`) y luego `200` con la fila ordenada por libro y antigüedad.

**¿Qué pasó?** `AdminRequired` compone sobre `AuthRequired` (ya existía): la fila administrativa queda reservada a roles `admin`. Para obtener `token_admin`: `UPDATE users SET role='admin' WHERE email='estudiante-reserva@example.com';` y volver a hacer login.

---

## 3. El hook — la fila avanza al devolver

### 3.1 Devolver el libro agotado... con copia nueva

Como el libro 2 llegó a `0` copias, ahora hay que **devolver** un préstamo de ese libro para simular que llegó una copia. Si no hay préstamo activo de ese libro, la podemos simular devolviendo el del paso 1.6 **cambiándolo primero al libro 2** o creando un préstamo previo cuando había copias. Para el ejemplo, la cadena limpia es:

1. `UPDATE books SET available_copies = 1 WHERE id = 2;` (admite un préstamo).
2. `POST /api/loans` prestando el libro 2 (con token).
3. `PUT /api/loans/:id/return` devolviéndolo → disponible de nuevo.

**Respuesta esperada al devolver** (`200`): el servicio de reservas avanza la fila.

### 3.2 Verificar que la fila avanzó ✅

```http
### Mis reservas, después de la devolución
GET http://localhost:8080/api/reservations/me
Authorization: Bearer {{token}}
```

**Respuesta esperada:** la reserva del libro 2 **ya no aparece** en "Mis reservas" (dejó de ser `waiting`). Para verla con su nuevo estado, consultar la BD (Adminer) o agregar un endifpoint de detalle:

```sql
SELECT id, user_id, book_id, status FROM reservations ORDER BY id;
```

| id | user_id | book_id | status |
|---|---|---|---|
| 1 | 1 | 2 | `fulfilled` |

**¿Qué pasó?** `LoanService.Return()` invocó `ReservationService.AdvanceQueue(bookID)` al final del flujo y `FulfillOldest` pasó la reserva más antigua de `waiting` → `fulfilled`. Esa es la "notificación" del modelo del ejemplo: *"tu libro está disponible"*.

---

## 4. Casos de borde y limpieza

### 4.1 Cancelar la reserva de otro usuario ❌

```http
### Otro estudiante intenta cancelar la reserva del usuario 1
DELETE http://localhost:8080/api/reservations/1
Authorization: Bearer {{token_otro}}
```

**Respuesta esperada** (`400`): `{"error":"no puedes cancelar una reserva de otro usuario"}` — la propiedad del recurso se valida en el servicio.

### 4.2 Cancelar una reserva ya atendida ❌

```http
### Cancelar la reserva "fulfilled" (id 1) tras el paso 3.2
DELETE http://localhost:8080/api/reservations/1
Authorization: Bearer {{token}}
```

**Respuesta esperada** (`400`): `{"error":"solo se pueden cancelar reservas en espera"}`.

### 4.3 Cancelar una reserva propia en espera ✅

```http
### Cancelar una reserva waiting propia
DELETE http://localhost:8080/api/reservations/:id
Authorization: Bearer {{token}}
```

**Respuesta esperada** (`200`): `{"message":"reserva cancelada correctamente"}` y en BD el `status` pasa a `canceled`.

### 4.4 Restaurar el seed

```sql
UPDATE books SET available_copies = 2 WHERE id = 2;
```

Dejar la base como estaba al finalizar la verificación.

---

## 5. Checklist final

- [ ] Contenedores `Up` y MySQL `(healthy)` (Evidencia 0)
- [ ] 1.1–1.8: health, ping, auth, books y préstamo devuelto ✅
- [ ] 2.1 reserva `201` · 2.2 libro con copias `409` · 2.3 duplicada `409`
- [ ] 2.4 `me` trae la reserva con título/autor · 2.5 sin token `401` · 2.6 admin-only `403`/`200`
- [ ] 3.1 devolución ejecutada · 3.2 fila avanzó (`waiting → fulfilled`)
- [ ] 4.1–4.3 cancelaciones: dueño, estado y propiedad validados
- [ ] 4.4 seed restaurado
- [ ] Todo lo **existente** (sección 1) sigue verde al final

> 📌 **Qué demuestra esta verificación:** la feature funciona de punta a punta, las **validaciones de negocio** responden con estados HTTP correctos (409/401/403/400), y **ningún flujo previo se rompió** — ese es exactamente el criterio del 25% "código funcionando" del proyecto.